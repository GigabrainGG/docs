# Hot-Reload Config & Rolling Deploys for Daemon Agents

## Problem

Daemon agents experience unnecessary downtime in two scenarios:

1. **Config changes** (soul.md, model, skills): The backend triggers a full ECS `forceNewDeployment`, killing the container and starting a fresh one. The daemon already has in-process hot-reload via file watcher — the restart is unnecessary.

2. **Code deploys** (new Docker image): ECS deployment config uses `minimumHealthyPercent: 0`, which kills the old task before starting the new one, causing 30-60s downtime.

## Solution Overview

### Part 1: Config Hot-Reload (Zero Downtime)

Stop restarting containers for config changes. Instead, the running daemon polls for config updates and applies them in-process.

### Part 2: Rolling Deploys (Near-Zero Downtime)

Change ECS deployment strategy so new containers start before old ones stop, with graceful Telegram handoff.

---

## Part 1: Config Hot-Reload

### New Backend Endpoint

**`GET /v2/agents/me/config-hash`**

Returns a hash of the agent's current config (soul.md + model + model_provider + skills_config). The daemon polls this every 5 seconds.

```json
{ "config_hash": "sha256:abc123..." }
```

The hash is computed from the concatenation of:
- `agent.data["instructions"]` (soul.md)
- `agent.data["model"]`
- `agent.data["model_provider"]`
- `agent.data["skills_config"]` (JSON-serialized)

This endpoint is authenticated with the agent's API key (same as `/me/config`).

### Daemon: Config Polling Loop

New background task in `telegram.py`:

```python
async def poll_config_updates():
    """Poll backend for config changes every 5 seconds."""
    last_hash = None
    while True:
        await asyncio.sleep(5)
        try:
            current_hash = await fetch_config_hash()
            if last_hash is not None and current_hash != last_hash:
                logger.info("Config changed — re-fetching from backend")
                await reload_config_from_backend()
            last_hash = current_hash
        except Exception as e:
            logger.warning(f"Config poll failed: {e}")
```

`reload_config_from_backend()` does:
1. `GET /v2/agents/me/config` — fetch full config
2. Write `soul.md`, `skills_config.json` to disk (same as `load_platform_config()`)
3. Update in-memory settings (`model_id`, `model_provider`)
4. The existing file watcher detects changes and triggers `create_agent()` automatically

### Backend: Remove Restart on Config Updates

Modify `platform_agents.py` endpoints:

- `POST /{agent_id}/soul` — remove `_restart_live_service_if_needed()` call
- `POST /{agent_id}/model` — remove `_restart_live_service_if_needed()` call
- `POST /{agent_id}/skills-config` — remove `_restart_live_service_if_needed()` call

These endpoints now only update the DB. The daemon picks up changes via polling.

### Frontend: No More UPDATING State for Config Changes

Since config changes no longer restart the container, the agent stays `LIVE` throughout. The frontend can show a brief "Saving..." indicator on the button, then done.

The `UPDATING` status is reserved for actual container restarts (code deploys, explicit restart).

### Config Endpoint Enhancement

The existing `GET /v2/agents/me/config` endpoint needs to return `soul_md` in the response. Currently it does not include it — the daemon only receives soul.md at initial startup via `load_platform_config()`. Add `soul_md` (from `agent.data["instructions"]`) to the config response.

---

## Part 2: Rolling Deploys

### ECS Deployment Configuration Change

Current (in `launch_service.py` or container_manager):
```python
deploymentConfiguration={
    "maximumPercent": 100,
    "minimumHealthyPercent": 0,
}
```

New:
```python
deploymentConfiguration={
    "maximumPercent": 200,
    "minimumHealthyPercent": 100,
}
```

This means:
- ECS starts the new task **before** killing the old one
- Old task only stops after new task passes health check
- Up to 2 tasks can run simultaneously during transition

### Telegram Handoff Strategy

**Problem:** Two containers cannot long-poll the same Telegram bot simultaneously (duplicate message processing, offset conflicts).

**Solution:** Delayed Telegram start + graceful SIGTERM shutdown.

#### New Container Startup Sequence

1. Load platform config from backend
2. Create Agno agent
3. Start health server on `:8000` (reports healthy immediately)
4. **Wait for startup grace period** (configurable, default 20s)
5. Start Telegram polling

The grace period gives ECS time to:
- See the new task as healthy
- Send SIGTERM to the old task
- Wait for old task to stop

#### Old Container Shutdown Sequence

On receiving SIGTERM:
1. Stop Telegram dispatcher (`dp.stop_polling()`)
2. Finish processing any in-flight message (aiogram handles this gracefully)
3. Close bot session
4. Report final status to backend
5. Exit

#### Implementation in `telegram.py`

```python
import signal

_shutdown_event = asyncio.Event()

def _handle_sigterm(*_):
    logger.info("SIGTERM received — initiating graceful shutdown")
    _shutdown_event.set()

signal.signal(signal.SIGTERM, _handle_sigterm)

async def main():
    # ... existing setup ...

    # Delay Telegram start in platform mode (rolling deploy)
    if settings.platform_mode:
        startup_delay = int(os.environ.get("TELEGRAM_START_DELAY", "20"))
        logger.info(f"Platform mode: waiting {startup_delay}s before starting Telegram")
        await asyncio.sleep(startup_delay)

    # Start polling with shutdown hook
    await dp.start_polling(_bot, ...)
```

#### Health Check

The existing health server on `:8000` already exists. ECS uses this to determine task health. The health check should report healthy as soon as config is loaded and agent is created (before Telegram starts), so ECS can proceed with the rolling update.

### Explicit Restart Command

The `POST /{agent_id}/restart` endpoint (manual restart from frontend) continues to use `forceNewDeployment`. The rolling deploy handles the transition automatically.

---

## Status Behavior Changes

| Action | Current Status | New Status |
|--------|---------------|------------|
| Update soul.md | LIVE → UPDATING → LIVE | LIVE (stays LIVE) |
| Update model | LIVE → UPDATING → LIVE | LIVE (stays LIVE) |
| Update skills | LIVE → UPDATING → LIVE | LIVE (stays LIVE) |
| Code deploy | LIVE → UPDATING → LIVE | LIVE → UPDATING → LIVE (but shorter) |
| Manual restart | LIVE → UPDATING → LIVE | LIVE → UPDATING → LIVE (rolling) |
| Pause/Resume | Unchanged | Unchanged |

---

## Files to Modify

### Backend (Python)

1. **`platform_agents.py`** — New `/me/config-hash` endpoint; remove `_restart_live_service_if_needed()` from soul/model/skills endpoints; add `soul_md` to config response
2. **`container_manager.py`** — Update ECS deployment configuration (`maximumPercent: 200, minimumHealthyPercent: 100`)
3. **`launch_service.py`** — Ensure service creation uses new deployment config

### Daemon

4. **`telegram.py`** — Add `poll_config_updates()` background task; add SIGTERM handler for graceful shutdown; add configurable Telegram start delay for platform mode
5. **`config.py`** — Add `fetch_config_hash()` and `reload_config_from_backend()` functions; add `telegram_start_delay` setting

### Frontend

6. **`useAgents.ts`** — Already fixed (polling added). The `UPDATING` state will now only appear during actual container restarts, which will be shorter.

---

## Risks and Mitigations

**Risk: Daemon misses a config update during the 5s poll window.**
Mitigation: Acceptable latency for config changes. User sees "Saved" immediately; agent picks it up within 5s.

**Risk: Two containers poll Telegram simultaneously during rolling deploy.**
Mitigation: 20s startup delay on new container + graceful SIGTERM on old. The overlap window is designed to be zero. If timing is off, Telegram sends updates to the newest poller — at worst a single message is processed by the old container before it shuts down.

**Risk: SIGTERM not handled properly, old container killed after 30s.**
Mitigation: ECS default stop timeout is 30s. Our graceful shutdown completes in <5s. Add explicit `stopTimeout: 30` to task definition as safety net.

**Risk: Config hash endpoint adds load.**
Mitigation: Negligible — it's a single DB read returning a hash. Could be cached with a 2s TTL if needed.
