# LiteLLM Consumer LLM Integration Design

## Goal

Let GigaBrain daemon agents use Claude Max and ChatGPT Pro consumer subscriptions via a centralized LiteLLM proxy, alongside the existing direct API key providers.

## Architecture

```
┌─ Agent Containers ────────────────────────────────────┐
│                                                        │
│  daemon (llm_provider: "litellm")                      │
│    └─ Agno OpenAIChat(base_url=LITELLM_URL, key=vk)   │
│                                                        │
│  daemon (llm_provider: "anthropic")                    │
│    └─ Agno Claude(api_key=user_key)  ← unchanged      │
│                                                        │
└────────────────────┬───────────────────────────────────┘
                     │ (only litellm provider)
                     ▼
┌─ LiteLLM Proxy (Railway, shared) ─────────────────────┐
│                                                        │
│  Per-user virtual keys for spend tracking              │
│                                                        │
│  Routes by model prefix:                               │
│  ├─ chatgpt/*   → ChatGPT subscription (OAuth)        │
│  ├─ claude/*    → Claude Max (OAuth token forwarding)  │
│  ├─ anthropic/* → Anthropic API (passthrough)          │
│  └─ openai/*    → OpenAI API (passthrough)             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## How It Works

### Provider Selection

LiteLLM is just another provider in the existing `llm_provider` enum:

- `openrouter` — existing, direct to OpenRouter API
- `anthropic` — existing, direct to Anthropic API
- `openai` — existing, direct to OpenAI API
- `xai` — existing, direct to xAI API
- `venice` — existing, direct to Venice API
- **`litellm` — NEW, routes through centralized LiteLLM proxy**

When `llm_provider == "litellm"`, the daemon creates an `OpenAIChat` model pointed at the LiteLLM proxy URL. All other providers remain unchanged.

### User Flow

1. **Connect subscription**: User goes to Settings, selects "Connect Claude Max" or "Connect ChatGPT Pro"
2. **Paste token**: User extracts session token from browser DevTools, pastes into GigaBrain
3. **Backend validates**: Backend validates token via LiteLLM test call, stores encrypted
4. **Virtual key created**: Backend calls LiteLLM Admin API to create a virtual key linked to the user's subscription token
5. **Model selection**: User sees LiteLLM models in the model picker (e.g., `litellm/claude-opus-4.6`)
6. **Agent uses it**: Daemon configured with `llm_provider: "litellm"`, `model_id: "litellm/claude-opus-4.6"`, API key = virtual key

### Token Types

**ChatGPT Pro/Plus:**
- OAuth device code flow (LiteLLM handles this natively)
- Stores `access_token` + `account_id` in LiteLLM's auth store
- Models: `chatgpt/gpt-5.4`, `chatgpt/gpt-5.4-pro`, etc.

**Claude Max:**
- OAuth token from claude.ai session
- Forwarded via `Authorization` header (`forward_client_headers_to_llm_api: true`)
- Models: `claude/claude-opus-4.6`, `claude/claude-sonnet-4.6`, etc.

## Changes Required

### 1. Daemon Config (`config.py`)

Add one new setting:

```python
litellm_base_url: str = ""  # e.g., https://litellm.gigabrain.gg
```

### 2. Daemon Agent (`agent.py` — `_create_model()`)

Add one new case:

```python
case "litellm":
    from agno.models.openai import OpenAIChat
    return OpenAIChat(
        id=settings.model_id,
        api_key=settings.openrouter_api_key,  # holds litellm virtual key
        base_url=settings.litellm_base_url,
    )
```

### 3. Python Backend — Integration Endpoint

Extend `POST /integrations/llm` to handle LiteLLM provider:

- Accept `provider: "litellm"` with subscription type (`claude_max` or `chatgpt_pro`)
- Validate token by making a test completion via LiteLLM
- Call LiteLLM Admin API (`POST /key/generate`) to create a virtual key for the user
- Store: virtual key (encrypted) + subscription type + masked token

### 4. Python Backend — Config Endpoint

When `model_provider == "litellm"`, the config endpoint returns:
- `api_key`: The user's LiteLLM virtual key
- `litellm_base_url`: The proxy URL (from env var)

### 5. Python Backend — Models Endpoint

Extend `POST /integrations/llm/models` for `provider: "litellm"`:
- Call LiteLLM's `/model/info` endpoint to list available models
- Filter by what the user's subscription supports
- Return model list to frontend

### 6. Frontend — Settings Tab

Add to provider integration UI:
- "Connect Claude Max" option with token paste modal
- "Connect ChatGPT Pro" option with token paste modal
- Instructions for extracting tokens from browser DevTools
- Show connected status + subscription type

### 7. LiteLLM Proxy Config (`config.yaml`)

Hosted on Railway:

```yaml
general_settings:
  forward_client_headers_to_llm_api: true
  master_key: "sk-master-xxx"

model_list:
  # ChatGPT subscription models
  - model_name: chatgpt/gpt-5.4
    model_info:
      mode: responses
    litellm_params:
      model: chatgpt/gpt-5.4

  # Claude Max models (forwarded OAuth)
  - model_name: claude/claude-opus-4.6
    litellm_params:
      model: anthropic/claude-opus-4-6

  # Standard API passthrough
  - model_name: anthropic/claude-sonnet-4.6
    litellm_params:
      model: anthropic/claude-sonnet-4-6

  - model_name: openai/gpt-4o
    litellm_params:
      model: openai/gpt-4o
```

## What Stays the Same

- All existing providers (`openrouter`, `anthropic`, `openai`, `xai`, `venice`) — zero changes
- Existing API key storage and encryption — reused for LiteLLM virtual keys
- Existing model picker UI — just adds a new provider section
- Existing config hot-reload — works for LiteLLM provider too
- Agent containers — no new processes, just a different base URL

## ToS Considerations

- **Claude Max OAuth**: Anthropic's ToS restricts OAuth to Claude Code and claude.ai. They have been actively blocking third-party usage since Jan 2026. Users should be informed this may stop working.
- **ChatGPT subscriptions**: Similar gray area. OAuth device flow is designed for Codex CLI.
- **Mitigation**: Show clear warning in the UI when connecting consumer tokens. Frame as "experimental" or "beta".

## Infrastructure

- **LiteLLM hosting**: Railway (user deploys)
- **LiteLLM URL**: Stored as `LITELLM_BASE_URL` env var in backend
- **LiteLLM master key**: Stored as `LITELLM_MASTER_KEY` env var in backend (for admin API calls)
- **Virtual key management**: Backend creates/rotates via LiteLLM Admin API

## Implementation Order

1. Host LiteLLM on Railway with config (user does this)
2. Add `litellm` provider to daemon `_create_model()` + config
3. Add LiteLLM virtual key management to backend integration endpoints
4. Add config endpoint support for `litellm_base_url`
5. Add frontend UI for connecting consumer subscriptions
6. Add model listing for LiteLLM provider
