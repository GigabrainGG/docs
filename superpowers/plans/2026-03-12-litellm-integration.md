# LiteLLM Consumer LLM Integration Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let daemon agents use Claude Max / ChatGPT Pro consumer subscriptions via a centralized LiteLLM proxy on Railway.

**Architecture:** LiteLLM is a new provider (`"litellm"`) alongside existing ones. When selected, the daemon uses `OpenAIChat` pointed at the LiteLLM Railway URL with a per-user virtual key. The backend creates virtual keys via LiteLLM Admin API when users connect their subscription tokens.

**Tech Stack:** Python (FastAPI, Agno, httpx), NestJS (GraphQL), Next.js (React), LiteLLM proxy on Railway

**LiteLLM instance:** `https://litellm-production-f341.up.railway.app`

---

## Chunk 1: Daemon Provider Support

### Task 1: Add `litellm` provider to daemon config

**Files:**
- Modify: `intelligence-monorepo/services/daemon/daemon-core/daemon/config.py:18` (add litellm_base_url)
- Modify: `intelligence-monorepo/services/daemon/daemon-core/daemon/config.py:428-434` (load litellm_base_url from config)

- [ ] **Step 1: Add `litellm_base_url` setting**

In `config.py`, after line 18 (`model_id`), add:

```python
    litellm_base_url: str = ""  # centralized LiteLLM proxy URL
```

Update the comment on line 18 to include litellm:

```python
    llm_provider: str = "openrouter"  # openrouter | anthropic | openai | xai | venice | litellm
```

- [ ] **Step 2: Load `litellm_base_url` from platform config**

In `config.py`, after line 434 (`object.__setattr__(settings, "openrouter_api_key", api_key)`), add handling for litellm:

```python
    # LiteLLM proxy URL (centralized gateway for consumer subscriptions)
    if litellm_url := data.get("litellm_base_url"):
        object.__setattr__(settings, "litellm_base_url", litellm_url)
```

- [ ] **Step 3: Commit**

```bash
git add intelligence-monorepo/services/daemon/daemon-core/daemon/config.py
git commit -m "feat(daemon): add litellm_base_url config setting"
```

### Task 2: Add `litellm` case to `_create_model()`

**Files:**
- Modify: `intelligence-monorepo/services/daemon/daemon-core/daemon/agent.py:89-107`

- [ ] **Step 1: Add litellm provider case**

In `agent.py`, after the `venice` block (line 99) and before the default OpenRouter block (line 101), add:

```python
    if provider == "litellm":
        if not settings.litellm_base_url:
            raise ValueError(
                "LiteLLM provider selected but no proxy URL configured. "
                "Set LITELLM_BASE_URL or connect via platform settings."
            )
        try:
            from agno.models.openai import OpenAIChat
        except ImportError as e:
            raise _missing_dependency("openai", e) from e

        return OpenAIChat(
            id=model_id,
            api_key=api_key,
            base_url=settings.litellm_base_url,
        )
```

- [ ] **Step 2: Verify no import changes needed**

`OpenAIChat` is already imported conditionally in the `openai` case. The litellm case does the same conditional import. No top-level import changes.

- [ ] **Step 3: Commit**

```bash
git add intelligence-monorepo/services/daemon/daemon-core/daemon/agent.py
git commit -m "feat(daemon): add litellm provider to _create_model()"
```

---

## Chunk 2: Backend Integration Endpoints

### Task 3: Add `litellm` to valid providers and integration upsert

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/integrations.py:29` (add litellm to VALID_LLM_PROVIDERS)
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/integrations.py:85` (validation for litellm)

- [ ] **Step 1: Add `litellm` to valid providers set**

Line 29, change:

```python
VALID_LLM_PROVIDERS = {"openrouter", "openai", "anthropic", "xai", "venice"}
```

to:

```python
VALID_LLM_PROVIDERS = {"openrouter", "openai", "anthropic", "xai", "venice", "litellm"}
```

- [ ] **Step 2: Add LiteLLM validation to `_validate_llm_provider_key()`**

Find the `_validate_llm_provider_key` function. Add a case for litellm that validates the virtual key by calling the LiteLLM `/key/info` endpoint:

```python
    elif provider == "litellm":
        litellm_url = os.environ.get("LITELLM_BASE_URL", "")
        if not litellm_url:
            raise HTTPException(status_code=500, detail="LiteLLM proxy URL not configured")
        resp = await client.get(
            f"{litellm_url}/key/info",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        if resp.status_code != 200:
            raise HTTPException(
                status_code=400, detail="Invalid LiteLLM virtual key"
            )
```

Add `import os` at the top if not already present.

- [ ] **Step 3: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/integrations.py
git commit -m "feat(backend): add litellm to valid LLM providers"
```

### Task 4: Add LiteLLM virtual key creation endpoint

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/integrations.py` (add new endpoint)

- [ ] **Step 1: Add request/response models**

After the existing `ModelItem` class (around line 64), add:

```python
class CreateLitellmKeyRequest(BaseModel):
    user_id: str
    subscription_type: str  # "claude_max" or "chatgpt_pro"
    session_token: str


class CreateLitellmKeyResponse(BaseModel):
    virtual_key: str
    subscription_type: str
    masked_token: str
```

- [ ] **Step 2: Add the endpoint**

After the `fetch_llm_models` endpoint, add:

```python
@router.post("/litellm/connect", response_model=CreateLitellmKeyResponse)
async def connect_litellm_subscription(body: CreateLitellmKeyRequest):
    """Create a LiteLLM virtual key for a consumer subscription.

    1. Calls LiteLLM Admin API to generate a virtual key
    2. Stores the virtual key as the user's litellm integration
    """
    import httpx

    valid_types = {"claude_max", "chatgpt_pro"}
    if body.subscription_type not in valid_types:
        raise HTTPException(
            status_code=400,
            detail=f"subscription_type must be one of: {', '.join(valid_types)}",
        )

    litellm_url = os.environ.get("LITELLM_BASE_URL", "").rstrip("/")
    litellm_master_key = os.environ.get("LITELLM_MASTER_KEY", "")
    if not litellm_url or not litellm_master_key:
        raise HTTPException(status_code=500, detail="LiteLLM not configured on backend")

    # Create virtual key via LiteLLM Admin API
    try:
        async with httpx.AsyncClient(timeout=15) as client:
            resp = await client.post(
                f"{litellm_url}/key/generate",
                json={
                    "metadata": {
                        "user_id": body.user_id,
                        "subscription_type": body.subscription_type,
                    },
                    "key_alias": f"gb-{body.user_id[:8]}-{body.subscription_type}",
                },
                headers={"Authorization": f"Bearer {litellm_master_key}"},
            )
            resp.raise_for_status()
            key_data = resp.json()
    except httpx.HTTPStatusError as e:
        raise HTTPException(
            status_code=502,
            detail=f"LiteLLM key generation failed: {e.response.text}",
        )
    except httpx.RequestError as e:
        raise HTTPException(status_code=502, detail=f"LiteLLM unreachable: {e}")

    virtual_key = key_data.get("key", "")
    if not virtual_key:
        raise HTTPException(status_code=502, detail="LiteLLM returned no key")

    # Store as a litellm integration (reuse existing upsert flow)
    masked = mask_api_key(virtual_key)

    # Now upsert via the standard flow
    upsert_body = UpsertLlmKeyRequest(
        user_id=body.user_id,
        provider="litellm",
        api_key=virtual_key,
        label=f"LiteLLM ({body.subscription_type.replace('_', ' ').title()})",
    )
    # Call upsert directly (skip re-validation since we just created the key)
    encrypted = encrypt_credential(virtual_key)
    now = datetime.now(UTC).isoformat()

    db = get_database_manager()
    session = db.get_session()
    try:
        repo = UserIntegrationRepository(session)
        existing = await repo.get_all_by_user_and_type(
            body.user_id, IntegrationType.LLM_PROVIDER
        )
        match = next(
            (i for i in existing if i.credentials and i.credentials.get("provider") == "litellm"),
            None,
        )

        credentials = {
            "provider": "litellm",
            "apiKey": encrypted,
            "maskedApiKey": masked,
            "validatedAt": now,
            "subscriptionType": body.subscription_type,
        }

        if match:
            match.credentials = credentials
            session.add(match)
        else:
            from users_database.models.user_integration import UserIntegration
            integration = UserIntegration(
                userId=UUID(body.user_id),
                integrationType="llm_provider",
                label=f"LiteLLM ({body.subscription_type.replace('_', ' ').title()})",
                isDefault=False,
                isActive=True,
                schemaVersion=2,
                credentials=credentials,
            )
            session.add(integration)

        await session.commit()
    except Exception as e:
        await session.rollback()
        raise HTTPException(status_code=500, detail=str(e)) from e
    finally:
        await session.close()

    return CreateLitellmKeyResponse(
        virtual_key=masked,
        subscription_type=body.subscription_type,
        masked_token=masked,
    )
```

- [ ] **Step 3: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/integrations.py
git commit -m "feat(backend): add LiteLLM virtual key creation endpoint"
```

### Task 5: Add `litellm_base_url` to config endpoint response

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py:417-433`

- [ ] **Step 1: Add `litellm_base_url` to config response**

In `platform_agents.py`, in the config endpoint response dict (around line 417-430), add `litellm_base_url`:

```python
        response = {
            "agent_id": agent_id,
            "telegram_bot_token": agent.telegram_bot_token or "",
            "telegram_allowed_user_ids": str(owner_tg_user_id) if owner_tg_user_id else "",
            "model": model_id,
            "model_provider": model_provider,
            "integrations": integrations,
            "litellm_base_url": os.environ.get("LITELLM_BASE_URL", ""),
            "settings": {
                "hl_builder_address": orch_settings.hl_builder_address,
                "hl_builder_fee_bps": orch_settings.hl_builder_fee_bps,
            },
            "soul_md": agent_data.get("instructions", ""),
            "memory_md": agent_data.get("memory_md", ""),
        }
```

Add `import os` at top if not already present.

- [ ] **Step 2: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py
git commit -m "feat(backend): include litellm_base_url in agent config response"
```

### Task 6: Add LiteLLM model listing

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/integrations.py` (extend fetch_llm_models)

- [ ] **Step 1: Add litellm case to `fetch_llm_models`**

In the `fetch_llm_models` function, add a case for litellm before the generic provider handling. When provider is "litellm", call LiteLLM's `/model/info` endpoint:

```python
    if body.provider == "litellm":
        litellm_url = os.environ.get("LITELLM_BASE_URL", "").rstrip("/")
        litellm_master_key = os.environ.get("LITELLM_MASTER_KEY", "")
        if not litellm_url or not litellm_master_key:
            return []
        try:
            async with httpx.AsyncClient(timeout=15) as client:
                resp = await client.get(
                    f"{litellm_url}/model/info",
                    headers={"Authorization": f"Bearer {litellm_master_key}"},
                )
                resp.raise_for_status()
                data = resp.json()
        except Exception:
            return []

        models = []
        for m in data.get("data", []):
            model_info = m.get("model_info", {})
            models.append(ModelItem(
                id=m.get("model_name", ""),
                name=m.get("model_name", ""),
                contextLength=model_info.get("max_input_tokens", 0),
            ))
        return models
```

Add `import httpx` at top if not already present.

- [ ] **Step 2: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/integrations.py
git commit -m "feat(backend): add LiteLLM model listing support"
```

---

## Chunk 3: NestJS GraphQL Wiring

### Task 7: Add LiteLLM connect mutation to NestJS

**Files:**
- Modify: `gigabrain-backend/src/user-agent/agent-platform.resolver.ts` (add mutation)
- Modify: `gigabrain-backend/src/user-agent/agent-platform.service.ts` (add service method)

- [ ] **Step 1: Add service method**

In `agent-platform.service.ts`, add a method to call the Python backend's `/integrations/litellm/connect` endpoint:

```typescript
  async connectLitellm(
    userId: string,
    subscriptionType: string,
    sessionToken: string,
  ): Promise<any> {
    const { data } = await axios.post(
      this.url('/integrations/litellm/connect'),
      {
        user_id: userId,
        subscription_type: subscriptionType,
        session_token: sessionToken,
      },
      { headers: this.headers },
    );
    return data;
  }
```

- [ ] **Step 2: Add GraphQL input type and mutation**

In the resolver file, add the input DTO and mutation:

```typescript
@InputType()
class ConnectLitellmInput {
  @Field()
  subscriptionType: string; // "claude_max" | "chatgpt_pro"

  @Field()
  sessionToken: string;
}

// In the resolver class:
@Mutation(() => AgentActionResponse, { name: 'connectLitellm' })
@UseGuards(GqlAuthGuard)
async connectLitellm(
  @Context() context,
  @Args('input') input: ConnectLitellmInput,
): Promise<AgentActionResponse> {
  const user = context.req.user;
  await this.agentService.connectLitellm(
    user.id,
    input.subscriptionType,
    input.sessionToken,
  );
  return { agent_id: '', status: 'connected' };
}
```

- [ ] **Step 3: Update GraphQL schema**

Run the NestJS schema generation or manually add the mutation and input type to `schema.gql`.

- [ ] **Step 4: Commit**

```bash
git add gigabrain-backend/src/user-agent/agent-platform.resolver.ts \
        gigabrain-backend/src/user-agent/agent-platform.service.ts \
        gigabrain-backend/src/schema.gql
git commit -m "feat(nestjs): add connectLitellm GraphQL mutation"
```

---

## Chunk 4: Frontend UI

### Task 8: Add LiteLLM connection UI to daemon settings

**Files:**
- Modify: `gigabrain-frontend/src/app/(dashboard)/agents/[id]/components/DaemonSettingsTab.tsx`

- [ ] **Step 1: Read current settings tab implementation**

Read the full `DaemonSettingsTab.tsx` to understand the existing provider connection UI pattern.

- [ ] **Step 2: Add LiteLLM connection section**

Add a new section below the existing provider connections. This includes:
- A "Connect Consumer Subscription" card
- Two buttons: "Connect Claude Max" and "Connect ChatGPT Pro"
- A modal/dialog that appears when clicked, with:
  - Instructions for extracting session token from browser DevTools
  - A textarea for pasting the token
  - A "Connect" button that calls the `connectLitellm` mutation
  - A warning about ToS and experimental status

- [ ] **Step 3: Add GraphQL mutation**

Add the `CONNECT_LITELLM` mutation:

```typescript
const CONNECT_LITELLM = gql`
  mutation connectLitellm($input: ConnectLitellmInput!) {
    connectLitellm(input: $input) {
      status
    }
  }
`;
```

- [ ] **Step 4: Add LiteLLM to the model picker**

When `connectedProviders` includes "litellm", show LiteLLM models in the model picker dropdown. The models come from the same `fetchModels` query with `provider: "litellm"`.

- [ ] **Step 5: Commit**

```bash
git add gigabrain-frontend/src/app/(dashboard)/agents/[id]/components/DaemonSettingsTab.tsx
git commit -m "feat(frontend): add LiteLLM consumer subscription connection UI"
```

---

## Chunk 5: Environment Config & Testing

### Task 9: Add environment variables to backend deployments

**Files:**
- Backend deployment config (Railway / ECS env vars)

- [ ] **Step 1: Add env vars to Python backend**

Add these environment variables to the brain-core backend:

```
LITELLM_BASE_URL=https://litellm-production-f341.up.railway.app
LITELLM_MASTER_KEY=<master-key-from-railway>
```

- [ ] **Step 2: Verify LiteLLM proxy is configured**

Test the LiteLLM proxy has the required models configured:

```bash
curl -s https://litellm-production-f341.up.railway.app/model/info \
  -H "Authorization: Bearer <master-key>" | jq '.data[].model_name'
```

Should list chatgpt/* and claude/* models.

- [ ] **Step 3: Commit env config if in a config file**

```bash
git commit -m "chore: add LiteLLM env vars to deployment config"
```

### Task 10: End-to-end smoke test

- [ ] **Step 1: Create a LiteLLM virtual key manually**

```bash
curl -X POST https://litellm-production-f341.up.railway.app/key/generate \
  -H "Authorization: Bearer <master-key>" \
  -H "Content-Type: application/json" \
  -d '{"metadata": {"user_id": "test"}}'
```

- [ ] **Step 2: Test chat completion via virtual key**

```bash
curl -X POST https://litellm-production-f341.up.railway.app/v1/chat/completions \
  -H "Authorization: Bearer <virtual-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "chatgpt/gpt-5.4",
    "messages": [{"role": "user", "content": "Hello"}],
    "max_tokens": 50
  }'
```

- [ ] **Step 3: Test from daemon locally**

Set env vars and run daemon with litellm provider:

```bash
LLM_PROVIDER=litellm \
OPENROUTER_API_KEY=<virtual-key> \
LITELLM_BASE_URL=https://litellm-production-f341.up.railway.app \
MODEL_ID=chatgpt/gpt-5.4 \
python -m daemon.telegram
```

Verify the agent responds via Telegram.

- [ ] **Step 4: Test frontend flow**

1. Go to agent Settings tab
2. Click "Connect ChatGPT Pro" or "Connect Claude Max"
3. Paste a session token
4. Verify it appears as connected
5. Select a LiteLLM model from the model picker
6. Send a message in Chat — verify agent responds
