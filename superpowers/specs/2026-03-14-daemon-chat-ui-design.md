# Daemon Agent Chat UI — Design Spec

## Overview

Replace the "Chat on Telegram" placeholder in the daemon agent detail page (`/agents/[id]`, Chat tab) with a full chat interface. The UI displays the agent's work Claude Code-style: tool calls appear as collapsible pills (collapsed by default, errors auto-expand), and the agent's final response streams below them.

Single continuous session per agent. Messages persist server-side via Agno's session DB. Client keeps messages in React state for instant tab switching; on page reload, fetches the last 10 messages from a new history endpoint with scroll-up pagination for older messages.

## Architecture

### Request Flow (existing, no changes)

```
Frontend fetch() POST
  → NestJS /v1/superagents/:id/chat  { stream: true, session_id }
    → Python /v2/agents/:id/chat
      → Daemon container /chat (aiohttp, SSE response)
```

### Message History Flow (new)

```
Frontend fetch() GET
  → NestJS /v1/superagents/:id/chat/history?session_id&limit&before
    → Python /v2/agents/:id/chat/history
      → Daemon container /chat/history (reads Agno SQLite session DB)
```

### State Management

- **React state** (`useState` in hook) holds messages for current session. Survives tab switches within the page.
- **localStorage** stores `session_id` per agent (key: `superagent_session_{agentId}`). Created on first message, reused forever.
- **Page reload**: Read `session_id` from localStorage, fetch last 10 messages from history endpoint, populate state.
- **Scroll up**: Paginate with `before=<oldest_message_created_at>` parameter.
- **New messages**: Appended to React state in real-time from SSE stream. Server persists automatically via Agno.

## Agno SSE Event Format

The daemon uses Agno `Agent.arun(stream=True, stream_events=True)`. Each event is serialized via `event.to_dict()` and sent as `data: {json}\n\n`.

### Event Types

Agno emits `ModelResponse` objects with an `event` field from the `ModelResponseEvent` enum. The exact event names in the `to_dict()` output use PascalCase values. **Note:** Agno versions may vary — during implementation, log a few SSE events from a live stream to confirm exact field names. The content streaming event may appear as `AssistantResponse` or a run-level event like `RunContent` depending on version.

| Event | Key Fields | UI Mapping |
|-------|-----------|------------|
| `ToolCallStarted` | `tool_calls[].function.{name, arguments}`, `tool_calls[].id` | Add pill with spinner + tool name. Use `tool_calls[].id` to correlate with completion. |
| `ToolCallCompleted` | `tool_executions[].{tool_call_id, tool_name, tool_args, result, tool_call_error}` | Match by `tool_call_id` (or by `tool_name` + index fallback). Spinner → green dot (or red if `tool_call_error`). Result available on expand. |
| `AssistantResponse` | `content` (text chunk) | Append to streaming content below tool calls |
| Other events | Various (e.g. `ModelRequestStarted`, `CompressionStarted`) | Ignored in UI |
| `stream_complete` | `session_id` (custom sentinel from daemon) | Remove cursor, finalize message |
| `error` | `message` (custom sentinel from daemon) | Show error inline |

**Tool call matching strategy:** Prefer `tool_call_id` if present in both `ToolCallStarted` and `ToolCallCompleted` events. Fallback: match by position (Nth started tool → Nth completed tool) since Agno processes tool calls sequentially.

### Custom sentinel (existing)

```json
{"event": "stream_complete", "session_id": "..."}
```

## New Backend Endpoints

### 1. Daemon Container: `GET /chat/history`

Added to the aiohttp server in `telegram.py` alongside existing `/health` and `/chat`.

**Query params:**
- `session_id` (required) — the session to fetch
- `limit` (optional, default 10, max 50) — number of messages to return
- `before` (optional) — ISO timestamp cursor for pagination (return messages older than this)

**Response:**
```json
{
  "session_id": "abc-123",
  "messages": [
    {
      "role": "user",
      "content": "Check ETH funding rates",
      "created_at": "2026-03-14T10:00:00Z"
    },
    {
      "role": "assistant",
      "content": "ETH funding is currently...",
      "tool_calls": [
        {
          "tool_name": "get_funding_rates",
          "tool_args": {"pair": "ETH-USD"},
          "result": "...",
          "error": false
        }
      ],
      "created_at": "2026-03-14T10:00:05Z"
    }
  ],
  "has_more": true
}
```

**Implementation:** Read from Agno's `daemon_sessions` SQLite table. Agno stores data as *runs* (not flat messages). Each run in the `runs` JSON column is a `RunOutput` containing:
- `input.input_content` — the user's message
- `content` — the assistant's response text
- `tools` — list of `ToolExecution` objects (`{tool_call_id, tool_name, tool_args, result, tool_call_error}`)
- `created_at` — unix timestamp of the run

**Flattening runs to messages:** Each run produces 2 messages (user + assistant). The `created_at` for the user message uses the run's `created_at`; the assistant message uses `created_at + 1s` (or a separate timestamp if available). Tool calls are extracted from `tools` and attached to the assistant message.

**Pagination:** The `before` cursor is an ISO timestamp. Filter runs where `created_at < before_timestamp`, take the last `limit/2` runs (since each run = 2 messages), return in chronological order.

**Implementation approach:** Use Agno's `AsyncSqliteDb` API to load the session by `session_id`, then parse the `runs` list in Python. This is more stable than raw SQL since it handles Agno's serialization format.

### 2. Python Proxy: `GET /v2/agents/{agent_id}/chat/history`

Added to `platform_agents.py` router. Proxies to daemon container `GET /chat/history` with same query params.

**Note:** The existing `_proxy_to_daemon()` helper's GET branch does not forward query parameters. Extend it to accept and forward `params` dict for GET requests, or use a direct `httpx.get()` call with `params=` in this route handler.

### 3. NestJS Proxy: `GET /v1/superagents/:id/chat/history`

Added to `ApiSuperagentController`. Same pattern as existing chat endpoint:
- Extract `req.user.id` from API key guard
- Verify agent ownership via `resolveOwnedAgent()`
- Proxy to Python backend
- Track usage

## Frontend Components

### New Files

| File | Purpose |
|------|---------|
| `agents/[id]/components/DaemonChatTab.tsx` | Main chat tab — message list, input, SSE handling |
| `agents/[id]/components/ToolCallPill.tsx` | Collapsible tool call display |
| `agents/[id]/hooks/useDaemonChat.ts` | Hook: SSE streaming, state, history, session management |

### DaemonChatTab

Replaces the Telegram placeholder in `agents/[id]/page.tsx` (the `{activeTab === 'chat' && (...)}` block that renders `FaTelegram` icon and "Chat on Telegram" CTA).

**Layout:**
- Flex column, full height of the tab content area
- Scrollable message list (flex: 1, overflow-y: auto)
- Fixed input area at bottom (border-top separator)

**Message list:**
- User messages: avatar + name + text content
- Assistant messages: agent avatar + name + tool call pills + response content
- Loading skeleton on initial history fetch
- "Load earlier messages" button at top (appears when `has_more` is true from history API)
- Auto-scroll to bottom on new messages (skip if user has scrolled up)

**Input area:**
- Textarea with auto-resize (reuse pattern from Brain chat page)
- Send button (arrow up icon)
- Disabled state when agent isn't `live` — show "Start agent to chat" message
- Disabled during streaming (prevent double-send)
- Enter to send, Shift+Enter for newline

**Props:** `agentId: string`, `agentName: string`, `agentStatus: string`

### ToolCallPill

Displays a single tool call from the agent's response.

**Collapsed state (default):**
- One-line horizontal layout in a rounded pill
- Status indicator: spinner (in-progress), green dot (success), red dot (error)
- Tool name in monospace
- Brief args summary on the right (truncated)
- Chevron indicator (▸)

**Expanded state (click to toggle):**
- Pill becomes header of an expanded panel
- Panel shows full args + result in a code block
- Syntax-highlighted for shell output, code, JSON
- Chevron flipped (▾)

**Auto-expand rules:**
- Errors always auto-expand
- Everything else starts collapsed

**Styling:**
- Background: `#0d0d0d`, border: `1px solid #1a1a1a`
- Error state: background `#1a0d0d`, border `#2a1515`, red dot
- Success dot: `#32F98F` (brand green)
- Error dot: `#ff6b6b`
- Font: monospace for tool name and expanded content

### useDaemonChat Hook

```typescript
interface DaemonChatMessage {
  id: string
  role: 'user' | 'assistant'
  content: string
  toolCalls?: ToolCallData[]
  created_at: string
  pending?: boolean  // true while streaming
}

interface ToolCallData {
  id: string
  toolName: string
  toolArgs: Record<string, any>
  result?: string
  error?: boolean
  status: 'running' | 'completed'
}

interface UseDaemonChatReturn {
  messages: DaemonChatMessage[]
  isStreaming: boolean
  isLoadingHistory: boolean
  hasMore: boolean
  sendMessage: (text: string) => Promise<void>
  loadMoreHistory: () => Promise<void>
}
```

**`sendMessage(text)`:**
1. If no `sessionId` yet, generate via `crypto.randomUUID()`, store in localStorage
2. Generate message ID, append user message to state
3. Create pending assistant message (empty content, `pending: true`)
4. `fetch()` POST to `/daemon-chat/:id` with JWT Bearer auth, body `{ message: text, session_id, stream: true }`
5. Read SSE via `response.body.getReader()` + `TextDecoder` (same pattern as Brain chat `useChatV2.ts`)
6. Parse each `data: {json}` line, switch on `event` field:
   - `ToolCallStarted`: Add `ToolCallData` with `status: 'running'` to current assistant message. Extract `tool_call_id` from `tool_calls[].id` if present.
   - `ToolCallCompleted`: Match by `tool_call_id` (or by index fallback). Update to `status: 'completed'`, store result. If `tool_call_error` is true, set `error: true`.
   - `AssistantResponse` (or content event): Append `content` chunk to assistant message
   - `session_info`: Confirm/update stored session ID
   - `stream_complete`: Set `pending: false`, finalize
   - `error`: Set error content on assistant message
   - Unknown events: Ignore silently (forward-compatible with future Agno versions)
7. Auto-scroll to bottom on each update

**`loadMoreHistory()`:**
1. Set `isLoadingHistory: true`
2. Apollo `useQuery`/`useLazyQuery` for `chatHistory(agentId, sessionId, limit: 10, before)` via GraphQL
3. Parse response, prepend messages to state (generate stable IDs: `{role}-{created_at_unix}-{index}`)
4. Update `hasMore` from response
5. Preserve scroll position (measure scrollHeight before/after, adjust scrollTop)

**Session management:**
- On mount: check `localStorage` for `superagent_session_{agentId}`
- If exists: call `loadMoreHistory()` for initial load (no `before` param = latest 10)
- If not: wait for first `sendMessage()` to create session ID via `crypto.randomUUID()`, store in localStorage immediately before sending the request
- The frontend always generates the session ID (UUIDv4 format). Both NestJS and daemon container accept valid UUIDs and pass them through. The NestJS proxy validates UUID format (`/^[0-9a-f-]{36}$/i`) and generates a new one only if the format is invalid — so a properly generated UUID will always be preserved.
- Handle `session_info` SSE event as a confirmation — if the returned `session_id` differs from what was sent (shouldn't happen with valid UUID), update localStorage

**Message IDs for history:**
- History messages from the API need stable IDs for React keys
- Generate client-side: `{role}-{created_at_unix}-{index}` (e.g. `user-1710403200-0`, `assistant-1710403201-0`)
- Live-streamed messages use `{role}-{Date.now()}-{crypto.randomUUID()}` (same as Brain chat)

**Concurrent tabs (accepted limitation):**
- Two browser tabs with the same agent will share the localStorage session ID and could send concurrent messages. Agno may handle this gracefully (sequential run processing) or produce interleaved results. Accepted for v1 — single-tab usage is the expected flow.

## Integration Points

### Page Modification

In `agents/[id]/page.tsx`, replace the `{activeTab === 'chat' && (...)}` block (currently rendering the Telegram icon + "Chat on Telegram" CTA) with:

Also clean up unused imports: remove `FaTelegram` from `react-icons/fa` and `ExternalLink` from `lucide-react` if no longer used elsewhere in the file.

```tsx
{activeTab === 'chat' && (
  <DaemonChatTab
    agentId={id}
    agentName={agent.name}
    agentStatus={agent.agent_status}
  />
)}
```

### Auth for Dashboard Calls

The frontend dashboard uses JWT auth (Bearer token from `localStorage`), not API keys. The existing `ApiSuperagentController` uses `@UseGuards(ApiKeyGuard)` and is intended for the public API / CLI. The `AgentRuntimeController` also uses `ApiKeyGuard` (agent-scoped keys, not user JWT) — it is for daemon containers to sync config, NOT for dashboard users.

**Decision: Two-pronged approach matching existing patterns.**

**1. Chat history — via GraphQL** (same auth as all other dashboard operations):

Add `chatHistory` query to `AgentPlatformResolver` (which uses `@UseGuards(GqlAuthGuard)` for JWT auth). The resolver calls `AgentPlatformService` which proxies to Python backend.

```graphql
type Query {
  chatHistory(agentId: String!, sessionId: String!, limit: Int, before: String): ChatHistoryResponse!
}

type ChatHistoryResponse {
  session_id: String!
  messages: [ChatMessage!]!
  has_more: Boolean!
}

type ChatMessage {
  id: String!
  role: String!
  content: String!
  tool_calls: [ChatToolCall!]
  created_at: String!
}

type ChatToolCall {
  tool_name: String!
  tool_args: JSON
  result: String
  error: Boolean
}
```

Add `getChatHistory()` method to `AgentPlatformService` that calls `GET /v2/agents/{id}/chat/history`.

**2. Streaming chat — via REST with JWT auth** (same pattern as Brain chat's `/custom_run`):

The Brain chat page already calls `POST /custom_run` with a Bearer JWT token (not an API key). Follow the same pattern: add a new JWT-authed REST endpoint for daemon agent chat streaming.

Create a new controller `DaemonChatController` with `@UseGuards(JwtAuthGuard)`:
- `POST /daemon-chat/:id` — proxies to Python `/v2/agents/:id/chat` with SSE streaming
- Uses JWT to identify user, verifies agent ownership via `UserAgentService`

This keeps the `ApiSuperagentController` purely for API key auth (public API / CLI) and the dashboard uses JWT-authed endpoints exclusively.

**Frontend calls:**
- History: Apollo `useQuery(GET_CHAT_HISTORY, { variables: { agentId, sessionId, limit } })`
- Send message: `fetch('/daemon-chat/:id', { headers: { Authorization: 'Bearer ' + jwt }, body: { message, session_id, stream: true } })`

## Error Handling

- **Agent not live**: Input disabled, message "Agent must be running to chat"
- **Network error during SSE**: Show error message in assistant bubble, allow retry
- **Agent timeout** (>15 min): Agno emits `error` event with timeout message, displayed inline
- **History fetch failure**: Show "Failed to load history" with retry button
- **Empty session** (first visit): Show empty state with "Send a message to start chatting" prompt

## Scope Boundaries

**In scope:**
- Chat tab UI with tool call pills + streaming content
- History endpoint (daemon + Python proxy + NestJS proxy/GraphQL)
- Session persistence (localStorage session ID + server-side Agno sessions)
- Scroll-up pagination for older messages

**Out of scope (future work):**
- Multiple sessions per agent
- Message search
- File/image attachments in chat
- Telegram bot token optional (separate PR, prerequisite for public API)
- Public API endpoints (separate PR, after this + TG optional)
