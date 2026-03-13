# Hot-Reload Config & Rolling Deploys Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate container restarts for config changes (soul.md, model, skills) via in-process hot-reload, and reduce downtime during code deploys via ECS rolling updates.

**Architecture:** The daemon agent polls a lightweight config-hash endpoint every 5s. When a change is detected, it re-fetches the full config and applies it in-process (the file watcher already handles agent recreation). For code deploys, ECS runs the new container alongside the old one, then drains the old after the new is healthy. SIGTERM handling ensures clean Telegram handoff.

**Tech Stack:** Python (FastAPI, httpx, asyncio), TypeScript (NestJS), AWS ECS Fargate, aiogram 3

---

## Chunk 1: Config Hot-Reload

### Task 1: Add config-hash endpoint to NestJS

Computes a SHA-256 hash of the agent's mutable config fields (soul_md, model, model_provider, skills_config) and returns it. The daemon polls this every 5s instead of fetching full config.

**Files:**
- Modify: `gigabrain-backend/src/user-agent/agent-runtime.controller.ts:56-69`

- [ ] **Step 1: Add the `GET me/config-hash` endpoint**

Add this method to `AgentRuntimeController` right after the existing `getConfig` method (after line 69):

```typescript
@Get('me/config-hash')
async getConfigHash(@Request() req: any) {
  const agent = await this.getOwnedAgent(req);
  const data = agent.data || {};

  // Hash the mutable config fields that the daemon cares about
  const hashInput = JSON.stringify({
    instructions: data.instructions || '',
    model: data.model || '',
    model_provider: data.model_provider || '',
    skills_config: data.skills_config || {},
  });

  const crypto = await import('crypto');
  const hash = crypto.createHash('sha256').update(hashInput).digest('hex');

  return { config_hash: hash };
}
```

Import `crypto` is already available in Node.js — no new dependency needed.

- [ ] **Step 2: Verify the endpoint works**

Run: `curl -H "Authorization: Bearer <agent-api-key>" http://localhost:3000/v2/agents/me/config-hash`
Expected: `{ "config_hash": "sha256hexstring..." }`

- [ ] **Step 3: Commit**

```bash
git add gigabrain-backend/src/user-agent/agent-runtime.controller.ts
git commit -m "feat: add config-hash endpoint for daemon hot-reload polling"
```

---

### Task 2: Add soul_md to the Python config response

Currently `GET /{agent_id}/config` does NOT return `soul_md`. The daemon needs it to write the updated soul.md to disk during hot-reload (not just on startup).

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py:417-431`

- [ ] **Step 1: Add `soul_md` and `memory_md` to the config response**

In the `get_agent_config` endpoint, add these fields to the response dict (around line 428, before the `skills_config` block):

```python
        response = {
            "agent_id": agent_id,
            "telegram_bot_token": agent.telegram_bot_token or "",
            "telegram_allowed_user_ids": str(owner_tg_user_id) if owner_tg_user_id else "",
            "model": model_id,
            "model_provider": model_provider,
            "integrations": integrations,
            "settings": {
                "hl_builder_address": orch_settings.hl_builder_address,
                "hl_builder_fee_bps": orch_settings.hl_builder_fee_bps,
            },
            "soul_md": agent_data.get("instructions", ""),
            "memory_md": agent_data.get("memory_md", ""),
        }
```

Note: `soul_md` is stored as `"instructions"` in the JSONB column.

- [ ] **Step 2: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py
git commit -m "feat: include soul_md and memory_md in agent config response"
```

---

### Task 3: Remove container restart from config update endpoints

The `update_soul`, `update_model`, and `update_skills_config` endpoints currently call `_restart_live_service_if_needed()`. Remove these calls so config updates are DB-only. The daemon will pick up changes via polling.

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py:750-880`

- [ ] **Step 1: Remove restart from `update_soul` (lines 750-771)**

Replace the endpoint to remove `container_id` tracking and the restart call:

```python
@router.post("/{agent_id}/soul")
async def update_soul(agent_id: str, request: UpdateSoulRequest):
    """Update soul.md in the agent record. Daemon hot-reloads via config polling."""
    await _verify_ownership(agent_id, request.user_id)
    session = _get_db().get_session()
    try:
        repo = UserAgentRepository(session)
        agent = await repo.get_by_id(UUID(agent_id))
        if not agent:
            raise HTTPException(status_code=404, detail="Agent not found")

        data = dict(agent.data or {})
        data["instructions"] = request.soul_md
        agent.data = data
        await session.commit()
    finally:
        await session.close()

    return {"agent_id": agent_id, "updated": True}
```

- [ ] **Step 2: Remove restart from `update_model` (lines 774-806)**

Replace the endpoint to remove `container_id` tracking and the restart call:

```python
@router.post("/{agent_id}/model")
async def update_model(agent_id: str, request: UpdateModelRequest):
    """Update the LLM model for an agent. Daemon hot-reloads via config polling."""
    await _verify_ownership(agent_id, request.user_id)

    normalized_model_provider = request.model_provider or None

    session = _get_db().get_session()
    try:
        await _validate_platform_model_selection(
            session,
            request.user_id,
            request.model,
            normalized_model_provider,
        )

        repo = UserAgentRepository(session)
        agent = await repo.get_by_id(UUID(agent_id))
        if not agent:
            raise HTTPException(status_code=404, detail="Agent not found")

        data = dict(agent.data or {})
        data["model"] = request.model
        data["model_provider"] = normalized_model_provider
        agent.data = data
        await session.commit()
    finally:
        await session.close()

    return {"agent_id": agent_id, "status": "updated"}
```

- [ ] **Step 3: Remove restart from `update_skills_config` (lines 847-880)**

Replace the endpoint to remove `container_id` tracking and the restart call:

```python
@router.post("/{agent_id}/skills-config")
async def update_skills_config(agent_id: str, request: UpdateSkillsConfigRequest):
    """Update which skills are disabled for an agent. Daemon hot-reloads via config polling."""
    await _verify_ownership(agent_id, request.user_id)

    session = _get_db().get_session()
    try:
        repo = UserAgentRepository(session)
        agent = await repo.get_by_id(UUID(agent_id))
        if not agent:
            raise HTTPException(status_code=404, detail="Agent not found")

        data = agent.data or {}
        reported_skills: list[dict] = data.get("skills", [])

        builtin_names = {s["name"] for s in reported_skills if s.get("tier") == "built-in"}
        invalid = builtin_names & set(request.disabled)
        if invalid:
            raise HTTPException(
                status_code=400,
                detail=f"Built-in skills cannot be disabled: {', '.join(sorted(invalid))}",
            )

        data["skills_config"] = {"disabled": request.disabled}
        agent.data = data
        await session.commit()
    finally:
        await session.close()

    return {"agent_id": agent_id, "updated": True}
```

- [ ] **Step 4: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py
git commit -m "feat: remove container restart from config update endpoints

Config changes (soul.md, model, skills) are now DB-only updates.
The daemon picks up changes via config-hash polling (5s interval)."
```

---

### Task 4: Add config polling loop to daemon

Add a background task that polls `GET /me/config-hash` every 5 seconds. When the hash changes, it re-fetches the full config and writes files to disk. The existing file watcher (`watch_config_files`) then detects the file changes and recreates the Agno agent.

**Files:**
- Modify: `intelligence-monorepo/services/daemon/daemon-core/daemon/config.py:346-468`
- Modify: `intelligence-monorepo/services/daemon/daemon-core/daemon/telegram.py:636-769`

- [ ] **Step 1: Add `fetch_config_hash()` and `reload_config_from_backend()` to config.py**

Add these functions at the end of `config.py` (after line 468):

```python
async def fetch_config_hash() -> str | None:
    """Fetch the config hash from the backend API."""
    if not settings.platform_mode or not settings.gigabrain_api_key:
        return None

    import httpx

    base_url = settings.gigabrain_api_url.rstrip("/")
    url = f"{base_url}/v2/agents/me/config-hash"
    headers = {"Authorization": f"Bearer {settings.gigabrain_api_key}"}

    async with httpx.AsyncClient(timeout=10) as client:
        resp = await client.get(url, headers=headers)
        resp.raise_for_status()
        return resp.json().get("config_hash", "")


async def reload_config_from_backend() -> None:
    """Re-fetch full config from the backend and write updated files to disk.

    The file watcher in telegram.py will detect the changes and recreate
    the Agno agent automatically. Only fields that the daemon can hot-reload
    are updated (soul.md, memory.md, model, skills_config). Credentials and
    wallet keys are NOT updated — those require a container restart.
    """
    if not settings.platform_mode or not settings.gigabrain_api_key:
        return

    import httpx
    import json
    from loguru import logger

    base_url = settings.gigabrain_api_url.rstrip("/")
    url = f"{base_url}/v2/agents/me/config"
    headers = {"Authorization": f"Bearer {settings.gigabrain_api_key}"}

    try:
        async with httpx.AsyncClient(timeout=15) as client:
            resp = await client.get(url, headers=headers)
            resp.raise_for_status()
            data = resp.json()
    except Exception as e:
        logger.error(f"Failed to reload config from backend: {e}")
        return

    # Update model settings in memory
    if model := data.get("model"):
        object.__setattr__(settings, "model_id", model)
    model_provider = data.get("model_provider")
    if model_provider is not None:
        object.__setattr__(settings, "llm_provider", model_provider or "")

    # Update LLM API key if provider changed
    integrations = data.get("integrations", {})
    if model_provider and model_provider in integrations:
        if api_key := integrations[model_provider].get("api_key"):
            object.__setattr__(settings, "openrouter_api_key", api_key)

    # Write config files to disk — file watcher will trigger agent reload
    if settings.data_dir:
        if "soul_md" in data:
            _write_runtime_file(settings.soul_md_path, data.get("soul_md", ""))
        if "memory_md" in data:
            _write_runtime_file(settings.memory_md_path, data.get("memory_md", ""))
        if "skills_config" in data:
            _write_runtime_file(
                settings.skills_config_path,
                json.dumps(data["skills_config"], indent=2),
            )

    _export_settings_to_env()
    logger.info(
        f"Config hot-reloaded: model={settings.model_id}, "
        f"provider={settings.llm_provider}"
    )
```

- [ ] **Step 2: Add the polling loop to `telegram.py:main()`**

Add this background task function inside `main()`, right after the `watch_config_files` function (after line 757), and before `watcher_task = asyncio.create_task(...)`:

```python
    async def poll_config_updates():
        """Poll backend for config changes every 5 seconds."""
        from daemon.config import fetch_config_hash, reload_config_from_backend

        if not settings.platform_mode or not settings.gigabrain_api_key:
            return

        last_hash: str | None = None
        while True:
            await asyncio.sleep(5)
            try:
                current_hash = await fetch_config_hash()
                if current_hash is None:
                    continue
                if last_hash is not None and current_hash != last_hash:
                    logger.info(f"Config hash changed ({last_hash[:8]}→{current_hash[:8]}) — reloading")
                    await reload_config_from_backend()
                last_hash = current_hash
            except Exception as e:
                logger.warning(f"Config poll failed: {e}")
```

- [ ] **Step 3: Start the polling task in `main()`**

After `watcher_task` and `sync_task` creation (around line 769), add:

```python
    config_poll_task = asyncio.create_task(poll_config_updates())
```

And in the `finally` block (around line 781), add cleanup:

```python
    finally:
        watcher_task.cancel()
        sync_task.cancel()
        config_poll_task.cancel()
        _scheduler_service.shutdown()
        await _bot.session.close()
        logger.info("Bot shutdown complete")
```

- [ ] **Step 4: Commit**

```bash
git add intelligence-monorepo/services/daemon/daemon-core/daemon/config.py
git add intelligence-monorepo/services/daemon/daemon-core/daemon/telegram.py
git commit -m "feat: add config polling loop for daemon hot-reload

Daemon polls GET /me/config-hash every 5s. When the hash changes,
it re-fetches the full config, writes updated files to disk, and
the existing file watcher recreates the Agno agent in-process.
Zero downtime for soul.md, model, and skills config changes."
```

---

## Chunk 2: Rolling Deploys

### Task 5: Update ECS deployment configuration

Change the ECS service deployment config from `minimumHealthyPercent: 0, maximumPercent: 100` (stop-first) to `minimumHealthyPercent: 100, maximumPercent: 200` (start-first rolling update).

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/brain_core/shared/services/orchestrator/container_manager.py:98-101`

- [ ] **Step 1: Update deployment config in `create_service`**

Change lines 98-101 from:

```python
            "deploymentConfiguration": {
                "maximumPercent": 100,
                "minimumHealthyPercent": 0,
            },
```

To:

```python
            "deploymentConfiguration": {
                "maximumPercent": 200,
                "minimumHealthyPercent": 100,
            },
```

- [ ] **Step 2: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/brain_core/shared/services/orchestrator/container_manager.py
git commit -m "feat: enable rolling deploys for daemon agent ECS services

Change from stop-first (max=100%, min=0%) to start-first
(max=200%, min=100%). New task starts and becomes healthy
before old task is drained."
```

**Note:** This only applies to newly created services. Existing services need a one-time update — either via the AWS console or a migration script calling `update_service` with the new deployment config. Consider adding a task or noting this for deployment.

---

### Task 6: Add SIGTERM handling for graceful Telegram shutdown

When ECS sends SIGTERM during a rolling update, the old container must stop Telegram polling cleanly so the new container can take over. aiogram's `start_polling` already handles SIGTERM somewhat, but we need explicit handling to ensure clean shutdown and status reporting.

**Files:**
- Modify: `intelligence-monorepo/services/daemon/daemon-core/daemon/telegram.py:636-785`

- [ ] **Step 1: Add SIGTERM handler and shutdown event**

Add this at the top of `main()`, right after the global declarations (after line 637):

```python
    import signal

    shutdown_event = asyncio.Event()

    def _handle_sigterm(*_):
        logger.info("SIGTERM received — initiating graceful shutdown")
        shutdown_event.set()

    loop = asyncio.get_running_loop()
    loop.add_signal_handler(signal.SIGTERM, _handle_sigterm)
```

- [ ] **Step 2: Add shutdown watcher task**

Add this right before `dp.start_polling()` (around line 775):

```python
    async def shutdown_watcher():
        """Watch for SIGTERM and gracefully stop the dispatcher."""
        await shutdown_event.wait()
        logger.info("Shutdown event triggered — stopping Telegram polling")
        if settings.platform_mode and settings.gigabrain_api_key:
            await _report_status_to_api("updating")
        await dp.stop_polling()

    shutdown_task = asyncio.create_task(shutdown_watcher())
```

- [ ] **Step 3: Clean up the shutdown task in finally block**

Update the `finally` block to also cancel the shutdown task:

```python
    finally:
        watcher_task.cancel()
        sync_task.cancel()
        config_poll_task.cancel()
        shutdown_task.cancel()
        _scheduler_service.shutdown()
        await _bot.session.close()
        logger.info("Bot shutdown complete")
```

- [ ] **Step 4: Commit**

```bash
git add intelligence-monorepo/services/daemon/daemon-core/daemon/telegram.py
git commit -m "feat: add SIGTERM handler for graceful Telegram shutdown

On SIGTERM (ECS rolling update), the daemon stops Telegram
polling, reports 'updating' status, and shuts down cleanly.
This ensures the old container releases the bot token before
the new container starts polling."
```

---

### Task 7: Add Telegram start delay for rolling deploys

During a rolling update, the new container must wait for the old one to stop polling Telegram before starting its own polling. Add a configurable startup delay in platform mode.

**Files:**
- Modify: `intelligence-monorepo/services/daemon/daemon-core/daemon/config.py:15-56`
- Modify: `intelligence-monorepo/services/daemon/daemon-core/daemon/telegram.py:775-777`

- [ ] **Step 1: Add `telegram_start_delay` setting**

Add this field to the `Settings` class in `config.py` (after `port` on line 44):

```python
    # Rolling deploy: delay Telegram polling start to allow old container to drain
    telegram_start_delay: int = 20  # seconds (0 = no delay, only applies in platform_mode)
```

- [ ] **Step 2: Add delay before Telegram polling starts**

In `telegram.py`, right before `dp.start_polling()` (around line 775), add:

```python
    # In platform mode, delay Telegram polling to allow old container to drain
    # during rolling deploys. The health server is already running, so ECS
    # sees us as healthy and can start draining the old task.
    if settings.platform_mode and settings.telegram_start_delay > 0:
        logger.info(
            f"Rolling deploy: waiting {settings.telegram_start_delay}s "
            f"before starting Telegram polling"
        )
        await asyncio.sleep(settings.telegram_start_delay)
```

- [ ] **Step 3: Commit**

```bash
git add intelligence-monorepo/services/daemon/daemon-core/daemon/config.py
git add intelligence-monorepo/services/daemon/daemon-core/daemon/telegram.py
git commit -m "feat: add Telegram start delay for rolling deploys

New containers wait 20s before starting Telegram polling,
giving ECS time to drain the old container. The health server
starts immediately so ECS marks the new task as healthy and
triggers the old task's SIGTERM."
```

---

### Task 8: Update existing ECS services deployment config (migration)

Existing services were created with the old deployment config (`max=100%, min=0%`). They need to be updated to the new config. This is a one-time operation.

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/brain_core/shared/services/orchestrator/container_manager.py`

- [ ] **Step 1: Add `update_deployment_config` method to ContainerManager**

Add this method after `restart_service` (after line 195):

```python
    async def update_deployment_config(self, service_name: str) -> None:
        """Update an existing service's deployment config to enable rolling deploys."""
        loop = asyncio.get_event_loop()
        await loop.run_in_executor(
            None,
            partial(
                self._ecs.update_service,
                cluster=self._settings.ecs_cluster,
                service=service_name,
                deploymentConfiguration={
                    "maximumPercent": 200,
                    "minimumHealthyPercent": 100,
                },
            ),
        )
        logger.info(f"Updated deployment config for service {service_name}")
```

- [ ] **Step 2: Add a one-time migration endpoint or script**

Add a maintenance endpoint to `platform_agents.py` (at the end of the file, guarded by internal API key):

```python
@router.post("/admin/migrate-deployment-config")
async def migrate_deployment_config(request: Request):
    """One-time migration: update all active services to rolling deploy config."""
    if not _has_valid_internal_api_key(request):
        raise HTTPException(status_code=403, detail="Internal API key required")

    cm = _get_container_manager()
    session = _get_db().get_session()
    try:
        repo = UserAgentRepository(session)
        agents = await repo.get_all_daemon_agents()
        updated = 0
        for agent in agents:
            if agent.container_id and agent.agent_status in ("live", "paused", "updating"):
                try:
                    await cm.update_deployment_config(agent.container_id)
                    updated += 1
                except Exception as e:
                    logger.warning(f"Failed to update service {agent.container_id}: {e}")
        return {"updated": updated, "total": len(agents)}
    finally:
        await session.close()
```

Note: The repository method `get_all_daemon_agents()` may not exist yet. If not, use a raw query:

```python
        results = await session.execute(
            text("SELECT id, container_id, agent_status FROM user_agents WHERE runtime_type = 'persistent' AND agent_status IS NOT NULL")
        )
        agents = results.fetchall()
```

- [ ] **Step 3: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/brain_core/shared/services/orchestrator/container_manager.py
git add intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py
git commit -m "feat: add migration for existing services to rolling deploy config

Adds update_deployment_config to ContainerManager and a one-time
admin endpoint to migrate all existing ECS services from stop-first
to start-first rolling deploys."
```

---

### Task 9: Frontend — remove UPDATING polling for config changes

Since config changes no longer trigger container restarts, the frontend shouldn't show UPDATING status for them. The `UPDATING` status is now only for code deploys and manual restarts. The list page polling fix from earlier in this session already handles this correctly — it polls when any agent is in a transitional state and stops when all are settled.

**Files:**
- Verify: `gigabrain-frontend/src/app/(dashboard)/agents/hooks/useAgents.ts` (already fixed earlier)

- [ ] **Step 1: Verify the polling fix is in place**

Read `useAgents.ts` and confirm the `hasTransitional` polling logic is present in `useAgents()`.

- [ ] **Step 2: No code changes needed**

The frontend already handles this correctly. Config updates no longer set status to UPDATING, so the badge stays LIVE. When actual restarts happen (code deploys), the rolling update makes the UPDATING window much shorter.

---

## Deployment Order

1. **Deploy NestJS backend** (Task 1) — adds `/me/config-hash` endpoint
2. **Deploy Python backend** (Tasks 2, 3, 8) — adds soul_md to config, removes restarts, adds migration endpoint
3. **Deploy daemon image** (Tasks 4, 6, 7) — adds config polling, SIGTERM handling, startup delay
4. **Run migration** (Task 8) — `POST /admin/migrate-deployment-config` to update existing ECS services
5. **Verify** — update a soul.md and confirm agent stays LIVE, no container restart

Steps 1-2 are backward-compatible (daemon still works without polling). Step 3 requires steps 1-2 to be deployed first. Step 4 can run anytime after step 2.
