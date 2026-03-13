# V1 to SuperAgent Migration Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an "Upgrade to SuperAgent" flow that converts V1 (scheduled) agents to SuperAgents in-place, preserving wallets and agent ID. Reuses the existing SuperAgent create wizard.

**Architecture:** V1 agents share the same `user_agents` DB table as SuperAgents, distinguished by `runtime_type`. Migration updates the record in-place (v1 → persistent), then launches an ECS container. The existing `/agents/create` wizard is reused with a `?migrate=agentId` query param — no new frontend components. Wallets stay linked by `agent_id` FK.

**Tech Stack:** Python FastAPI (backend endpoint), NestJS/GraphQL (API gateway), Next.js/React (minimal frontend mods)

---

## File Structure

| File | Action | Responsibility |
|------|--------|---------------|
| `intelligence-monorepo/agents/brain-core/brain_core/shared/services/orchestrator/launch_service.py` | Modify | Add `migrate()` method (like `launch()` but skips DB create + wallet provisioning) |
| `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py` | Modify | Add `MigrateAgentRequest` model + `POST /v2/agents/{agent_id}/migrate` endpoint |
| `gigabrain-backend/src/user-agent/dto/agent-platform.types.ts` | Modify | Add `MigrateAgentInput` GraphQL type |
| `gigabrain-backend/src/user-agent/agent-platform.service.ts` | Modify | Add `migrateAgent()` proxy method |
| `gigabrain-backend/src/user-agent/agent-platform.resolver.ts` | Modify | Add `migrateAgentToSuperAgent` mutation |
| `gigabrain-frontend/src/graphql/mutations.tsx` | Modify | Add `MIGRATE_AGENT` mutation |
| `gigabrain-frontend/src/app/(dashboard)/agents/create/page.tsx` | Modify | Add `?migrate=agentId` support (pre-fill name, call migrate mutation) |
| `gigabrain-frontend/src/app/(dashboard)/agents-legacy/[id]/page.tsx` | Modify | Add "Upgrade to SuperAgent" button |

---

## Chunk 1: Python Backend

### Task 1: Add migrate method to LaunchService

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/brain_core/shared/services/orchestrator/launch_service.py`

The `migrate()` method is like `launch()` but operates on an existing agent record. It skips DB creation and wallet provisioning since the agent and wallet already exist.

- [ ] **Step 1: Add the migrate method**

Add after the `launch()` method (after line ~225, before `async def pause`):

```python
async def migrate(
    self,
    agent_id: str,
    user_id: str,
    soul_md: str,
    telegram_bot_token: str,
    model: str | None = None,
    model_provider: str | None = None,
) -> dict:
    """
    Migrate a V1 agent to SuperAgent in-place.

    Skips DB create and wallet provisioning (agent + wallet already exist).
    Updates the existing record, creates API key, launches ECS service.
    """
    short = agent_id[:8]

    # --- Step 1: Validate Telegram bot token ---
    logger.info(f"[{short}] Validating Telegram bot token...")
    bot_info = await self.validate_telegram_token(telegram_bot_token)
    bot_username = bot_info.get("username", "")

    # Check bot token uniqueness
    session = self._get_db().get_session()
    try:
        repo = UserAgentRepository(session)
        existing = await repo.find_by_bot_token(telegram_bot_token)
        if existing and str(existing.id) != agent_id:
            raise LaunchError(
                f"Bot @{bot_username} is already in use by agent '{existing.name}'.",
                stage="bot_token_check",
            )
    finally:
        await session.close()

    # --- Step 2: Pre-flight credit check ---
    from brain_core.shared.credits import RunType, get_credit_manager

    credit_manager = get_credit_manager()
    has_credits, balance, required = await credit_manager.check_balance(
        user_id, RunType.AGENT_COMPUTE
    )
    if not has_credits:
        raise LaunchError(
            f"Insufficient credits (balance={balance:.2f}, need={required:.2f})",
            stage="credit_check",
        )

    # --- Step 3: Update existing agent record ---
    # Note: Ownership and runtime_type are already verified by the endpoint.
    logger.info(f"[{short}] Converting agent record to persistent...")
    agent_name = ""
    wallet_address = ""
    session = self._get_db().get_session()
    try:
        repo = UserAgentRepository(session)
        agent = await repo.get_by_id(uuid.UUID(agent_id))
        if not agent:
            raise LaunchError("Agent not found", stage="db_update")

        agent_name = agent.name or ""

        agent.runtime_type = RuntimeType.PERSISTENT.value
        agent.agent_status = AgentStatus.PROVISIONING.value
        agent.enabled = True
        agent.triggers = []
        agent.trigger_embedding = None
        agent.data = {
            "instructions": soul_md,
            "memory_md": "",
            "goal": "",
            "model": model or "anthropic/claude-sonnet-4",
            "model_provider": model_provider,
            "runtime": "persistent",
        }
        agent.telegram_bot_token = telegram_bot_token
        agent.telegram_bot_username = bot_username
        await session.commit()

        # Fetch wallet address
        wallet_repo = AgentWalletRepository(session)
        wallet = await wallet_repo.get_by_agent_id(uuid.UUID(agent_id))
        if wallet:
            wallet_address = wallet.privy_wallet_address or ""
    except LaunchError:
        raise
    except Exception as e:
        logger.error(f"[{short}] Failed to update agent record: {e}")
        raise LaunchError(str(e), stage="db_update") from e
    finally:
        await session.close()

    # --- Step 4: Create per-agent API key ---
    try:
        agent_api_key = await self._rotate_agent_api_key(agent_id, user_id)
        logger.info(f"[{short}] API key created")
    except Exception as e:
        logger.error(f"[{short}] API key creation failed: {e}")
        await self._update_status(agent_id, AgentStatus.FAILED)
        raise LaunchError(str(e), stage="api_key_create") from e

    # --- Step 5: Create ECS service ---
    logger.info(f"[{short}] Creating ECS service...")
    await self._update_status(agent_id, AgentStatus.LAUNCHING)

    try:
        service_name = await self._cm.create_service(
            agent_id=agent_id,
            agent_api_key=agent_api_key,
        )
    except Exception as e:
        logger.error(f"[{short}] ECS service creation failed: {e}")
        await self._update_status(agent_id, AgentStatus.FAILED)
        raise LaunchError(str(e), stage="ecs_launch") from e

    await self._update_container_id(agent_id, service_name)

    # --- Step 6: Wait for steady state ---
    logger.info(f"[{short}] Waiting for service steady state...")
    running = await self._cm.wait_for_steady_state(service_name)

    if running:
        await self._update_status(agent_id, AgentStatus.LIVE)
        logger.info(f"[{short}] Migration complete! Agent is LIVE")
    else:
        await self._update_status(agent_id, AgentStatus.FAILED)
        logger.error(f"[{short}] Service failed to reach steady state")

    return {
        "agent_id": agent_id,
        "name": agent_name,
        "service_name": service_name,
        "bot_username": bot_username,
        "wallet_address": wallet_address,
        "status": AgentStatus.LIVE.value if running else AgentStatus.FAILED.value,
    }
```

- [ ] **Step 2: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/brain_core/shared/services/orchestrator/launch_service.py
git commit -m "feat: add migrate() method to LaunchService for V1->SuperAgent conversion"
```

---

### Task 2: Add migrate endpoint to platform_agents.py

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py`

- [ ] **Step 1: Add RuntimeType import**

Add `RuntimeType` to the `from users_database import (...)` block at the top of the file (around line 19-26). It is already exported from users_database but not currently imported in this file:

```python
from users_database import (
    AgentStatus,
    DatabaseManager,
    RuntimeType,  # ADD THIS
    UserAgentRepository,
    UserIntegrationRepository,
    UserRepository,
    get_database_manager,
)
```

- [ ] **Step 2: Add request model**

Add after the existing `UpdateSkillsConfigRequest` model (around line 261):

```python
class MigrateAgentRequest(BaseModel):
    user_id: str
    soul_md: str
    telegram_bot_token: str = Field(..., min_length=40)
    model: str | None = None
    model_provider: str | None = None
```

- [ ] **Step 3: Add the migrate endpoint**

Add after the `launch_agent` endpoint (after line 506), before the LIFECYCLE section:

**Important:** `_verify_ownership()` returns `None` (it only raises on failure). Follow the existing pattern: call `_verify_ownership()` first, then open a separate DB session to fetch the agent.

```python
@router.post("/{agent_id}/migrate", response_model=LaunchAgentResponse)
async def migrate_agent_to_superagent(agent_id: str, request: MigrateAgentRequest):
    """
    Migrate a V1 agent to SuperAgent in-place.

    Converts the agent record, launches ECS container. Wallet stays linked.
    """
    await _verify_ownership(agent_id, request.user_id)

    # Check runtime_type — reject if already a SuperAgent
    session = _get_db().get_session()
    try:
        repo = UserAgentRepository(session)
        agent = await repo.get_by_id(UUID(agent_id))
        if not agent:
            raise HTTPException(status_code=404, detail="Agent not found")
        if agent.runtime_type == RuntimeType.PERSISTENT.value:
            raise HTTPException(status_code=409, detail="Agent is already a SuperAgent")
        # Disable V1 agent (stop triggers) before migration
        if agent.enabled:
            agent.enabled = False
            await session.commit()
    finally:
        await session.close()

    # Validate model selection
    normalized_model_provider = request.model_provider or None
    session = _get_db().get_session()
    try:
        await _validate_platform_model_selection(
            session,
            request.user_id,
            request.model,
            normalized_model_provider,
        )
    finally:
        await session.close()

    try:
        result = await _get_launch_service().migrate(
            agent_id=agent_id,
            user_id=request.user_id,
            soul_md=request.soul_md,
            telegram_bot_token=request.telegram_bot_token,
            model=request.model,
            model_provider=normalized_model_provider,
        )
        return LaunchAgentResponse(
            agent_id=agent_id,
            name=result["name"],
            bot_username=result["bot_username"],
            wallet_address=result["wallet_address"],
            status=result["status"],
        )
    except LaunchError as e:
        logger.error(f"Migration failed at stage '{e.stage}': {e}")
        status_map = {
            "telegram_validation": 400,
            "bot_token_check": 409,
            "credit_check": 402,
            "db_update": 500,
            "api_key_create": 500,
            "ecs_launch": 500,
        }
        raise HTTPException(
            status_code=status_map.get(e.stage, 500),
            detail=str(e),
        ) from e
    except Exception as e:
        logger.error(f"Unexpected migration error: {e}")
        raise HTTPException(status_code=500, detail="Internal migration error") from e
```

- [ ] **Step 4: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py
git commit -m "feat: add migrate endpoint for V1->SuperAgent conversion"
```

---

## Chunk 2: NestJS GraphQL Layer

### Task 3: Add GraphQL types for migration

**Files:**
- Modify: `gigabrain-backend/src/user-agent/dto/agent-platform.types.ts`

- [ ] **Step 1: Add MigrateAgentInput type**

Add at the end of the file (after `AgentDeployments` class, line 264):

```typescript
@InputType()
export class MigrateAgentInput {
  @Field()
  soul_md: string;

  @Field()
  telegram_bot_token: string;

  @Field({ nullable: true })
  model?: string;

  @Field({ nullable: true })
  model_provider?: string;
}
```

- [ ] **Step 2: Commit**

```bash
git add gigabrain-backend/src/user-agent/dto/agent-platform.types.ts
git commit -m "feat: add MigrateAgentInput GraphQL type"
```

---

### Task 4: Add service method for migration

**Files:**
- Modify: `gigabrain-backend/src/user-agent/agent-platform.service.ts`

- [ ] **Step 1: Add migrateAgent method**

Add after the existing `updateSoul` method (after line 205). **Important:** Use the existing `this.url()` helper and `const { data }` destructuring pattern.

```typescript
async migrateAgent(
  agentId: string,
  userId: string,
  input: {
    soul_md: string;
    telegram_bot_token: string;
    model?: string;
    model_provider?: string;
  },
): Promise<any> {
  const { data } = await axios.post(
    this.url(`/${agentId}/migrate`),
    { user_id: userId, ...input },
    { headers: this.headers },
  );
  return data;
}
```

- [ ] **Step 2: Commit**

```bash
git add gigabrain-backend/src/user-agent/agent-platform.service.ts
git commit -m "feat: add migrateAgent service method"
```

---

### Task 5: Add resolver mutation for migration

**Files:**
- Modify: `gigabrain-backend/src/user-agent/agent-platform.resolver.ts`

- [ ] **Step 1: Update imports**

Add `MigrateAgentInput` to the import from `./dto/agent-platform.types` (line 6-23):

```typescript
import {
  // ... existing imports ...
  MigrateAgentInput,
} from './dto/agent-platform.types';
```

- [ ] **Step 2: Add migrateAgentToSuperAgent mutation**

Add after the `launch` mutation (after line 53). **Important:** Every `@Mutation()` in this file includes an explicit `{ name: '...' }` option. Return `LaunchAgentResponse` so the frontend gets `agent_id`, `name`, `bot_username`, `wallet_address`, `status`.

```typescript
@Mutation(() => LaunchAgentResponse, { name: 'migrateAgentToSuperAgent' })
@UseGuards(GqlAuthGuard)
async migrateToSuperAgent(
  @Context() context,
  @Args('agentId') agentId: string,
  @Args('input') input: MigrateAgentInput,
): Promise<LaunchAgentResponse> {
  const user = context.req.user;
  return this.agentService.migrateAgent(agentId, user.id, {
    soul_md: input.soul_md,
    telegram_bot_token: input.telegram_bot_token,
    model: input.model,
    model_provider: input.model_provider,
  });
}
```

- [ ] **Step 3: Commit**

```bash
git add gigabrain-backend/src/user-agent/agent-platform.resolver.ts
git commit -m "feat: add migrateAgentToSuperAgent mutation"
```

---

## Chunk 3: Frontend (Reuse Existing Create Wizard)

### Task 6: Add GraphQL mutation for migration

**Files:**
- Modify: `gigabrain-frontend/src/graphql/mutations.tsx`

- [ ] **Step 1: Add MIGRATE_AGENT mutation**

Add at the end of the file. Returns the same `LaunchAgentResponse` fields as `LAUNCH_AGENT`:

```typescript
export const MIGRATE_AGENT = gql`
  mutation migrateAgentToSuperAgent($agentId: String!, $input: MigrateAgentInput!) {
    migrateAgentToSuperAgent(agentId: $agentId, input: $input) {
      agent_id
      name
      bot_username
      wallet_address
      status
    }
  }
`
```

- [ ] **Step 2: Commit**

```bash
git add gigabrain-frontend/src/graphql/mutations.tsx
git commit -m "feat: add MIGRATE_AGENT GraphQL mutation"
```

---

### Task 7: Add migration mode + random name generation to create wizard

**Files:**
- Modify: `gigabrain-frontend/src/app/(dashboard)/agents/create/page.tsx`

Two changes in one task:
1. **Random name generation** — pre-fill with a sci-fi themed name on page load (for new agents). Refresh button to get another. 50 adjectives × 20 nouns = 1,000 combos.
2. **Migration mode** — `?migrate=agentId` support: pre-fill name from V1 agent (read-only), call migrate mutation, swap UI text.

- [ ] **Step 1: Add imports**

Add to the existing imports at the top of the file:

```typescript
import { useSearchParams } from 'next/navigation'
import { GET_USER_AGENT } from '@/graphql/queries'
import { MIGRATE_AGENT } from '@/graphql/mutations'
import { RefreshCw } from 'lucide-react'
```

- [ ] **Step 2: Add name generation constants and function**

Add after the existing `SOUL_PLACEHOLDER` constant (after line 39), before the `LaunchResult` interface:

```typescript
const NAME_ADJECTIVES = [
  'Quantum', 'Phantom', 'Nebula', 'Shadow', 'Nova', 'Cipher', 'Apex', 'Flux',
  'Zenith', 'Volt', 'Arc', 'Prism', 'Rogue', 'Drift', 'Pulse', 'Crimson',
  'Stellar', 'Echo', 'Vector', 'Nexus', 'Obsidian', 'Neon', 'Cobalt', 'Onyx',
  'Vortex', 'Helix', 'Plasma', 'Stealth', 'Iron', 'Astral', 'Hyper', 'Lunar',
  'Solar', 'Cryo', 'Binary', 'Chrome', 'Omega', 'Delta', 'Sigma', 'Turbo',
  'Nano', 'Sonic', 'Warp', 'Ember', 'Frost', 'Storm', 'Blaze', 'Static',
  'Aether', 'Radiant',
]

const NAME_NOUNS = [
  'Scout', 'Alpha', 'Sentinel', 'Oracle', 'Spectre', 'Maverick', 'Titan',
  'Vanguard', 'Recon', 'Warden', 'Trader', 'Pilot', 'Runner', 'Striker',
  'Hunter', 'Seeker', 'Drifter', 'Prowler', 'Navigator', 'Catalyst',
]

function generateAgentName(): string {
  const adj = NAME_ADJECTIVES[Math.floor(Math.random() * NAME_ADJECTIVES.length)]
  const noun = NAME_NOUNS[Math.floor(Math.random() * NAME_NOUNS.length)]
  return `${adj} ${noun}`
}
```

- [ ] **Step 3: Add migration state, data fetching, and name pre-fill**

Inside `CreateAgentPage()`, after `const { launch, isLaunching } = useAgents()` (line 51), add:

```typescript
const searchParams = useSearchParams()
const migrateAgentId = searchParams.get('migrate')
const isMigrateMode = !!migrateAgentId

// Fetch V1 agent data when in migration mode
const { data: legacyAgentData } = useQuery<{
  getUserAgent: { id: string; name: string; data?: Record<string, unknown> }
}>(GET_USER_AGENT, {
  variables: { userAgentId: migrateAgentId ?? '' },
  skip: !migrateAgentId,
})

// Pre-fill name: from V1 agent in migration mode, random name otherwise
useEffect(() => {
  if (isMigrateMode && legacyAgentData?.getUserAgent?.name) {
    setName(legacyAgentData.getUserAgent.name)
  } else if (!isMigrateMode && !name) {
    setName(generateAgentName())
  }
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, [legacyAgentData, isMigrateMode])

const [migrateMutation] = useMutation(MIGRATE_AGENT)
```

- [ ] **Step 4: Add migration path to handleLaunch**

Replace the `handleLaunch` callback (lines 147-195) with an updated version that handles both launch and migration modes. The key change is in the `try` block — when `isMigrateMode`, call `migrateMutation` instead of `launch`:

```typescript
const handleLaunch = useCallback(async () => {
  if (!name.trim()) {
    customToast.error('Agent name is required')
    return
  }
  if (!botToken || !tokenValid) {
    customToast.error('Valid Telegram bot token is required')
    return
  }
  if (!selectedModel || !modelProvider) {
    customToast.error('Choose a model before launching')
    return
  }
  if (modelProvider && !connectedProviders.includes(modelProvider)) {
    customToast.error(`Connect your ${modelProvider} API key in Settings first`)
    return
  }

  const isAuthenticated = requireAuth()
  if (!isAuthenticated) return

  if (!soulMd.trim()) {
    customToast.error('Soul.md is required — describe your agent\'s identity')
    return
  }

  setLaunching(true)

  try {
    if (isMigrateMode && migrateAgentId) {
      // Migration path — convert existing V1 agent
      const { data } = await migrateMutation({
        variables: {
          agentId: migrateAgentId,
          input: {
            soul_md: soulMd,
            telegram_bot_token: botToken.trim(),
            model: selectedModel,
            model_provider: modelProvider,
          },
        },
      })
      const result = data?.migrateAgentToSuperAgent
      if (result) {
        setLaunchResult(result)
        customToast.success('Agent upgraded to SuperAgent!')
      } else {
        setLaunching(false)
        customToast.error('Upgrade failed — please try again')
      }
    } else {
      // Normal launch path
      const input: LaunchAgentInput = {
        name: name.trim(),
        telegram_bot_token: botToken.trim(),
        soul_md: soulMd,
        model: selectedModel,
        model_provider: modelProvider,
      }
      const result = await launch(input)
      if (result) {
        setLaunchResult(result)
        customToast.success('Agent launched!')
      } else {
        setLaunching(false)
        customToast.error('Launch failed — please try again')
      }
    }
  } catch (err: any) {
    setLaunching(false)
    customToast.error(err?.message || (isMigrateMode ? 'Upgrade failed' : 'Launch failed'))
  }
}, [name, botToken, tokenValid, soulMd, selectedModel, modelProvider, connectedProviders, launch, requireAuth, isMigrateMode, migrateAgentId, migrateMutation])
```

- [ ] **Step 5: Update UI text and name input for migration mode**

Make these conditional text changes in the JSX:

**Page title** — replace the `title="Launch Agent"` in all three `PageHeader` components (lines 236, 289, 324) with:

```tsx
title={isMigrateMode ? 'Upgrade to SuperAgent' : 'Launch Agent'}
```

**Success heading** — replace `"Agent is Live!"` (line 247) with:

```tsx
{isMigrateMode ? 'Agent Upgraded!' : 'Agent is Live!'}
```

**Success subtitle** — replace the `<p>` with `{name}` (line 249) with:

```tsx
<span className="text-white">{name}</span> {isMigrateMode ? 'has been upgraded to a SuperAgent.' : 'is running and ready to chat.'}
```

**Launching messages** — replace the `launchMessages` useMemo (lines 201-207). Wrap it to show upgrade-specific messages when migrating:

```tsx
const launchMessages = useMemo(() => isMigrateMode ? [
  { title: 'Stopping scheduled triggers...', subtitle: 'Preparing for upgrade', icon: Brain },
  { title: 'Converting to SuperAgent...', subtitle: 'Updating agent configuration', icon: FlaskConical },
  { title: 'Launching container...', subtitle: 'Spinning up your always-on agent', icon: Wrench },
  { title: 'Almost ready...', subtitle: 'Connecting to Telegram', icon: Sparkles },
  { title: 'Final checks...', subtitle: 'Your SuperAgent is about to go live', icon: Eye },
] : [
  { title: 'Waking up your agent...', subtitle: 'Neurons firing, synapses connecting', icon: Brain },
  { title: 'Brewing intelligence...', subtitle: 'Pouring knowledge into a fresh mind', icon: FlaskConical },
  { title: 'Assembling the brain...', subtitle: 'Wiring up wallet, memory, and instincts', icon: Wrench },
  { title: 'Almost alive...', subtitle: 'Teaching it to think, trade, and talk', icon: Sparkles },
  { title: 'Final sparks...', subtitle: 'Your agent is about to open its eyes', icon: Eye },
], [isMigrateMode])
```

**Agent name input** — replace the entire name `<div>` block (lines 345-357) with a version that has a refresh button for new agents and is read-only for migration:

```tsx
<div>
  <label className="block text-[#888] text-sm font-mono uppercase tracking-wide mb-2">
    Agent Name
  </label>
  <div className="relative">
    <input
      type="text"
      value={name}
      onChange={(e) => !isMigrateMode && setName(e.target.value)}
      readOnly={isMigrateMode}
      placeholder="e.g. Alpha Scout, Portfolio Manager, DeFi Analyst"
      maxLength={100}
      className={`w-full px-4 py-3 border border-neutral-900 outline-none focus-within:border-[#262626] bg-[#121212] rounded-md text-white text-base placeholder:text-[#444] ${isMigrateMode ? 'opacity-60 cursor-not-allowed' : 'pr-10'}`}
    />
    {!isMigrateMode && (
      <button
        type="button"
        onClick={() => setName(generateAgentName())}
        className="absolute right-3 top-1/2 -translate-y-1/2 text-[#555] hover:text-white transition-colors"
        title="Generate random name"
      >
        <RefreshCw className="w-4 h-4" />
      </button>
    )}
  </div>
</div>
```

**Launch button disabled** — replace `isLaunching` in the button's `disabled` prop (line 477) to also account for the local `launching` state in migration mode:

```tsx
disabled={!name.trim() || !tokenValid || !selectedModel || !modelProvider || isLaunching || launching}
```

**Launch button text** — replace `'Launch Agent'` / `'Launching...'` (line 483) with:

```tsx
{(isLaunching || launching) ? (isMigrateMode ? 'Upgrading...' : 'Launching...') : (isMigrateMode ? 'Upgrade to SuperAgent' : 'Launch Agent')}
```

- [ ] **Step 6: Commit**

```bash
git add gigabrain-frontend/src/app/\(dashboard\)/agents/create/page.tsx
git commit -m "feat: add random name generation + migration mode to create wizard"
```

---

### Task 8: Add Upgrade button to V1 agent detail page

**Files:**
- Modify: `gigabrain-frontend/src/app/(dashboard)/agents-legacy/[id]/page.tsx`

- [ ] **Step 1: Add imports**

Add `ArrowUpRight` to the existing lucide-react imports (line 7):

```typescript
import { ArrowLeft, BrainCircuit, Wallet, Sparkles, BookOpen, TrendingUp, History, SlidersHorizontal, Plus, X, ArrowUpRight } from 'lucide-react'
```

Add `Link` from next/link:

```typescript
import Link from 'next/link'
```

- [ ] **Step 2: Add the Upgrade button**

Inside the PageHeader section (around line 350-362), add a `rightSlot` prop. If `PageHeader` doesn't support `rightSlot`, place the button just below the PageHeader instead:

**Option A** — If PageHeader supports rightSlot:

Add `rightSlot` prop to the PageHeader at line 352:

```tsx
rightSlot={
  agent && (
    <Link
      href={`/agents/create?migrate=${agent.id}`}
      className="flex items-center gap-1.5 px-3 py-1.5 bg-brand/10 hover:bg-brand/20 text-brand text-sm rounded-sm transition-colors"
    >
      <ArrowUpRight className="w-3.5 h-3.5" />
      Upgrade to SuperAgent
    </Link>
  )
}
```

**Option B** — If PageHeader doesn't support rightSlot, add the button as a banner after the PageHeader, before the main content (around line 363, before the loading/content conditional):

```tsx
{agent && (
  <div className="flex justify-end mb-4">
    <Link
      href={`/agents/create?migrate=${agent.id}`}
      className="flex items-center gap-1.5 px-3 py-1.5 bg-brand/10 hover:bg-brand/20 text-brand text-sm rounded-sm transition-colors"
    >
      <ArrowUpRight className="w-3.5 h-3.5" />
      Upgrade to SuperAgent
    </Link>
  </div>
)}
```

Check `PageHeader` component to determine which option to use.

- [ ] **Step 3: Commit**

```bash
git add gigabrain-frontend/src/app/\(dashboard\)/agents-legacy/\[id\]/page.tsx
git commit -m "feat: add Upgrade to SuperAgent button on V1 agent detail page"
```

---

## Verification Checklist

After all tasks are complete, verify end-to-end:

- [ ] **Python migrate**: `POST /v2/agents/{v1_agent_id}/migrate` converts agent, returns `status: "live"`
- [ ] **Idempotency**: Calling migrate on an already-migrated agent returns 409
- [ ] **Wallet preservation**: After migration, wallet is still linked and accessible on the SuperAgent detail page
- [ ] **NestJS**: GraphQL mutation `migrateAgentToSuperAgent` works via playground
- [ ] **Frontend flow**: Click "Upgrade to SuperAgent" on V1 detail → create wizard opens with name pre-filled → fill in bot token + model + soul.md → click Upgrade → success screen
- [ ] **Redirect**: After successful migration, "View Agent" button navigates to `/agents/{id}` (SuperAgent detail page)
- [ ] **Error handling**: Insufficient credits shows proper error, invalid Telegram token shows error, already-migrated agent shows error
