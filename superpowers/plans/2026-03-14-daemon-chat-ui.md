# Daemon Agent Chat UI — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the "Chat on Telegram" placeholder in the daemon agent detail page with a Claude Code-style chat interface that shows tool calls as collapsible pills and streams agent responses in real-time.

**Architecture:** Four-layer implementation: (1) daemon container gets a `/chat/history` endpoint reading from Agno's session DB, (2) Python proxy forwards history requests, (3) NestJS exposes GraphQL query for history + new REST controller for SSE streaming with JWT auth, (4) frontend hook + components render the chat with tool call pills.

**Tech Stack:** Agno (Python agent framework), aiohttp, FastAPI, NestJS (TypeScript), Next.js 16, React, Apollo GraphQL, SSE streaming via fetch ReadableStream.

**Spec:** `docs/superpowers/specs/2026-03-14-daemon-chat-ui-design.md`

---

## File Structure

### New Files

| File | Responsibility |
|------|---------------|
| `intelligence-monorepo/services/daemon/daemon-core/daemon/history.py` | Chat history endpoint logic — reads Agno session DB, flattens runs to messages |
| `gigabrain-backend/src/user-agent/daemon-chat.controller.ts` | JWT-authed REST controller for daemon SSE chat streaming |
| `gigabrain-frontend/src/app/(dashboard)/agents/[id]/hooks/useDaemonChat.ts` | Hook: SSE streaming, message state, session management, history loading |
| `gigabrain-frontend/src/app/(dashboard)/agents/[id]/components/DaemonChatTab.tsx` | Main chat tab — message list + input |
| `gigabrain-frontend/src/app/(dashboard)/agents/[id]/components/ToolCallPill.tsx` | Collapsible tool call display component |

### Modified Files

| File | Change |
|------|--------|
| `intelligence-monorepo/services/daemon/daemon-core/daemon/telegram.py` | Register `/chat/history` route in aiohttp server |
| `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py` | Add `GET /v2/agents/{id}/chat/history` proxy route, extend `_proxy_to_daemon` with query param support |
| `gigabrain-backend/src/user-agent/agent-platform.service.ts` | Add `getChatHistory()` method |
| `gigabrain-backend/src/user-agent/agent-platform.resolver.ts` | Add `chatHistory` GraphQL query |
| `gigabrain-backend/src/user-agent/user-agent.module.ts` | Register `DaemonChatController` |
| `gigabrain-frontend/src/app/(dashboard)/agents/[id]/page.tsx` | Replace Telegram placeholder with `DaemonChatTab` |

---

## Chunk 1: Backend — Daemon Container History Endpoint

### Task 1: Create history module that reads Agno session DB

**Files:**
- Create: `intelligence-monorepo/services/daemon/daemon-core/daemon/history.py`

- [ ] **Step 1: Create history.py with session reading logic**

This module reads from Agno's `daemon_sessions` SQLite table, extracts runs from the session, and flattens them into a user/assistant message list.

```python
"""Chat history — reads Agno's session DB and returns flattened messages."""

from __future__ import annotations

import json
import logging
import sqlite3
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

from daemon.config import settings

logger = logging.getLogger(__name__)


def _open_db() -> sqlite3.Connection:
    db_path = str(settings.session_db_path)
    return sqlite3.connect(db_path)


def get_chat_history(
    session_id: str,
    limit: int = 10,
    before: str | None = None,
) -> dict[str, Any]:
    """Return flattened messages for a session, newest-last.

    Each Agno "run" produces two messages (user input + assistant response).
    We parse the `memory` JSON column which stores the full session state
    including all messages exchanged with the LLM.

    Args:
        session_id: The session to fetch.
        limit: Max messages to return (default 10, max 50).
        before: ISO timestamp cursor — return messages older than this.

    Returns:
        {"session_id": str, "messages": [...], "has_more": bool}
    """
    limit = min(max(limit, 1), 50)

    try:
        conn = _open_db()
        cursor = conn.cursor()

        # Agno stores sessions with session_id as primary key.
        # The `memory` column holds JSON with `runs` array.
        cursor.execute(
            "SELECT memory FROM daemon_sessions WHERE session_id = ?",
            (session_id,),
        )
        row = cursor.fetchone()
        conn.close()

        if not row or not row[0]:
            return {"session_id": session_id, "messages": [], "has_more": False}

        memory = json.loads(row[0]) if isinstance(row[0], str) else row[0]
        runs = memory.get("runs", [])

        # Flatten runs into messages
        messages: list[dict] = []
        for i, run in enumerate(runs):
            created_at = run.get("created_at")
            if created_at is None:
                # Fallback: use index-based offset from epoch
                created_at = 1700000000 + i

            # Convert unix timestamp to ISO string
            if isinstance(created_at, (int, float)):
                ts = datetime.fromtimestamp(created_at, tz=timezone.utc).isoformat()
            else:
                ts = str(created_at)

            # User message from run input
            user_content = ""
            run_input = run.get("input")
            if isinstance(run_input, dict):
                user_content = run_input.get("input_content", "") or run_input.get("message", "")
            elif isinstance(run_input, str):
                user_content = run_input

            if user_content:
                messages.append({
                    "id": f"user-{i}",
                    "role": "user",
                    "content": user_content,
                    "created_at": ts,
                })

            # Assistant message from run output
            assistant_content = run.get("content", "")
            tool_calls = []

            # Extract tool executions
            tools = run.get("tools") or run.get("tool_executions") or []
            for j, tool in enumerate(tools):
                tc: dict[str, Any] = {
                    "tool_name": tool.get("tool_name", "unknown"),
                    "tool_args": tool.get("tool_args", {}),
                    "result": tool.get("result"),
                    "error": bool(tool.get("tool_call_error", False)),
                }
                tool_calls.append(tc)

            if assistant_content or tool_calls:
                msg: dict[str, Any] = {
                    "id": f"assistant-{i}",
                    "role": "assistant",
                    "content": assistant_content or "",
                    "created_at": ts,
                }
                if tool_calls:
                    msg["tool_calls"] = tool_calls
                messages.append(msg)

        # Apply `before` filter
        if before:
            messages = [m for m in messages if m["created_at"] < before]

        # Return last `limit` messages (chronological, newest at end)
        has_more = len(messages) > limit
        if has_more:
            messages = messages[-limit:]

        return {
            "session_id": session_id,
            "messages": messages,
            "has_more": has_more,
        }

    except Exception as e:
        logger.error(f"Failed to read chat history for session {session_id}: {e}")
        return {"session_id": session_id, "messages": [], "has_more": False}
```

- [ ] **Step 2: Verify the Agno session DB schema**

Before wiring this up, we need to verify the actual column names in the `daemon_sessions` table. Run inside a daemon container or dev environment:

```bash
cd intelligence-monorepo/services/daemon/daemon-core
python3 -c "
import sqlite3, json
conn = sqlite3.connect('data/sessions.db')
cursor = conn.cursor()
cursor.execute('PRAGMA table_info(daemon_sessions)')
print('Columns:', cursor.fetchall())
cursor.execute('SELECT * FROM daemon_sessions LIMIT 1')
row = cursor.fetchone()
if row:
    for i, col in enumerate(cursor.description):
        val = row[i]
        if isinstance(val, str) and len(val) > 200:
            val = val[:200] + '...'
        print(f'{col[0]}: {val}')
conn.close()
"
```

Expected: Table has columns including `session_id` and a JSON column (`memory` or `session_data`) containing runs. **If column names differ, update `history.py` accordingly.**

- [ ] **Step 3: Commit**

```bash
git add intelligence-monorepo/services/daemon/daemon-core/daemon/history.py
git commit -m "feat(daemon): add chat history module for reading Agno session DB"
```

### Task 2: Register history endpoint in daemon aiohttp server

**Files:**
- Modify: `intelligence-monorepo/services/daemon/daemon-core/daemon/telegram.py`

- [ ] **Step 1: Add the history route handler**

In `telegram.py`, find the `_start_http_server()` function (around line 620). Add a new handler before the `app = web.Application()` line:

```python
    async def chat_history_handler(request: web.Request) -> web.Response:
        # Auth: same as chat_handler
        auth_header = request.headers.get("Authorization", "")
        expected_token = settings.gigabrain_api_key
        if not expected_token or not auth_header.startswith("Bearer "):
            return web.json_response({"error": "Unauthorized"}, status=401)
        if auth_header[7:] != expected_token:
            return web.json_response({"error": "Unauthorized"}, status=401)

        session_id = request.query.get("session_id")
        if not session_id:
            return web.json_response({"error": "session_id is required"}, status=400)

        limit = int(request.query.get("limit", "10"))
        before = request.query.get("before")

        from daemon.history import get_chat_history
        result = get_chat_history(session_id=session_id, limit=limit, before=before)
        return web.json_response(result)
```

- [ ] **Step 2: Register the route**

Find the line `app.router.add_post("/chat", chat_handler)` (around line 690) and add after it:

```python
    app.router.add_get("/chat/history", chat_history_handler)
```

- [ ] **Step 3: Commit**

```bash
git add intelligence-monorepo/services/daemon/daemon-core/daemon/telegram.py
git commit -m "feat(daemon): register /chat/history endpoint in aiohttp server"
```

---

## Chunk 2: Backend — Python & NestJS Proxy Layer

### Task 3: Extend Python proxy with query param support + history route

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py`

- [ ] **Step 1: Add `params` support to `_proxy_to_daemon`**

Find `_proxy_to_daemon` (around line 992). Add a `params` parameter:

Change the function signature from:
```python
async def _proxy_to_daemon(
    agent_id: str,
    user_id: str,
    method: str,
    path: str,
    body: dict | None = None,
    timeout: int = 180,
) -> dict:
```

To:
```python
async def _proxy_to_daemon(
    agent_id: str,
    user_id: str,
    method: str,
    path: str,
    body: dict | None = None,
    params: dict | None = None,
    timeout: int = 180,
) -> dict:
```

And update the GET branch from:
```python
                resp = await client.get(url, headers=headers)
```
To:
```python
                resp = await client.get(url, headers=headers, params=params)
```

- [ ] **Step 2: Add the chat history route**

Find the chat endpoint route (search for `@router.post("/{agent_id}/chat")`). Add above it:

```python
@router.get("/{agent_id}/chat/history")
async def get_chat_history(
    agent_id: str,
    session_id: str,
    limit: int = 10,
    before: str | None = None,
    user_id: str = Depends(get_current_user_id),
):
    """Get chat history for a daemon agent session."""
    # Verify ownership
    agent = await _get_owned_agent(agent_id, user_id)
    if agent.runtime_type != "daemon":
        raise HTTPException(status_code=400, detail="Chat history only available for daemon agents")

    params = {"session_id": session_id, "limit": str(limit)}
    if before:
        params["before"] = before

    return await _proxy_to_daemon(
        agent_id=agent_id,
        user_id=user_id,
        method="GET",
        path="/chat/history",
        params=params,
    )
```

**Note:** Check how `get_current_user_id` and `_get_owned_agent` work in this file — they may have different names. Search for existing patterns in the same file:
- `Depends(...)` for user ID extraction
- Agent ownership checks used by other endpoints

- [ ] **Step 3: Commit**

```bash
git add intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py
git commit -m "feat(python): add chat history proxy route + query param support"
```

### Task 4: Add NestJS GraphQL query for chat history

**Files:**
- Modify: `gigabrain-backend/src/user-agent/agent-platform.service.ts`
- Modify: `gigabrain-backend/src/user-agent/agent-platform.resolver.ts`

- [ ] **Step 1: Add `getChatHistory` to AgentPlatformService**

Open `agent-platform.service.ts`. Add after the `getAgentFiles` method (around line 201):

```typescript
  async getChatHistory(
    agentId: string,
    userId: string,
    sessionId: string,
    limit: number = 10,
    before?: string,
  ): Promise<any> {
    const params: Record<string, string> = {
      user_id: userId,
      session_id: sessionId,
      limit: String(limit),
    };
    if (before) params.before = before;

    const { data } = await axios.get(this.url(`/${agentId}/chat/history`), {
      params,
      headers: this.headers,
    });
    return data;
  }
```

- [ ] **Step 2: Add `chatHistory` query to AgentPlatformResolver**

Open `agent-platform.resolver.ts`. Find the existing query methods (search for `@Query`). Add a new query:

First, check the existing imports and add any missing ones. You'll need `Args` from `@nestjs/graphql` (likely already imported).

Add the query method:

```typescript
  @Query(() => GraphQLJSON, { name: 'chatHistory' })
  @UseGuards(GqlAuthGuard)
  async chatHistory(
    @Context() context,
    @Args('agentId') agentId: string,
    @Args('sessionId') sessionId: string,
    @Args('limit', { nullable: true, defaultValue: 10 }) limit: number,
    @Args('before', { nullable: true }) before: string,
  ): Promise<any> {
    const user = context.req.user;
    return this.platformService.getChatHistory(
      agentId,
      user.id,
      sessionId,
      limit,
      before,
    );
  }
```

**Note:** Check how other queries in this file return data. If they use typed return objects instead of `GraphQLJSON`, follow that pattern. The resolver likely already imports `GraphQLJSON` — check the top of the file.

- [ ] **Step 3: Commit**

```bash
git add gigabrain-backend/src/user-agent/agent-platform.service.ts gigabrain-backend/src/user-agent/agent-platform.resolver.ts
git commit -m "feat(nestjs): add chatHistory GraphQL query for daemon agent sessions"
```

### Task 5: Create NestJS REST controller for daemon SSE chat streaming

**Files:**
- Create: `gigabrain-backend/src/user-agent/daemon-chat.controller.ts`
- Modify: `gigabrain-backend/src/user-agent/user-agent.module.ts`

- [ ] **Step 1: Create DaemonChatController**

This follows the exact pattern of `custom-run.controller.ts` (SSE streaming via axios + pipe) but is specifically for daemon agent chat. Uses `GqlAuthGuard` (which supports both HTTP and GraphQL contexts, as used by `custom_run`).

```typescript
import {
  Controller,
  Post,
  Body,
  Res,
  Param,
  Request,
  UseGuards,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Response } from 'express';
import { GqlAuthGuard } from '../auth/guards/gql-auth.guard';
import { UserAgentService } from './user-agent.service';
import axios from 'axios';
import * as crypto from 'crypto';

interface DaemonChatDto {
  message: string;
  session_id?: string;
  stream?: boolean;
}

@Controller('daemon-chat')
@UseGuards(GqlAuthGuard)
export class DaemonChatController {
  private readonly brainApiUrl: string;
  private readonly brainApiKey: string | undefined;

  constructor(
    private readonly userAgentService: UserAgentService,
    private readonly configService: ConfigService,
  ) {
    this.brainApiUrl =
      this.configService.get<string>('BRAIN_INTERNAL_API_BASE_URL') ||
      'https://brain-internal-api.gigabrain.cloud';
    this.brainApiKey = this.configService.get<string>('BRAIN_INTERNAL_API_KEY');
  }

  @Post(':id')
  async chat(
    @Request() req,
    @Param('id') agentId: string,
    @Body() body: DaemonChatDto,
    @Res() response: Response,
  ) {
    const user = req.user;
    const raw = body.session_id?.trim();
    const sessionId =
      raw && /^[0-9a-f-]{36}$/i.test(raw) ? raw : crypto.randomUUID();

    try {
      if (!body.message || !body.message.trim()) {
        throw new HttpException('message is required', HttpStatus.BAD_REQUEST);
      }

      // Verify ownership
      const agent = await this.userAgentService.getUserAgent(agentId);
      if (!agent) {
        throw new HttpException('Agent not found', HttpStatus.NOT_FOUND);
      }
      if (agent.user_id !== user.id) {
        throw new HttpException('Access denied', HttpStatus.FORBIDDEN);
      }

      const headers: Record<string, string> = {
        'Content-Type': 'application/json',
      };
      if (this.brainApiKey) {
        headers['x-api-key'] = this.brainApiKey;
      }

      const chatPayload = {
        user_id: user.id,
        message: body.message,
        session_id: sessionId,
        stream: true,
      };

      const streamResponse = await axios.post(
        `${this.brainApiUrl}/v2/agents/${agentId}/chat`,
        chatPayload,
        { headers, responseType: 'stream', timeout: 930_000 },
      );

      response.setHeader('Content-Type', 'text/event-stream');
      response.setHeader('Cache-Control', 'no-cache');
      response.setHeader('Connection', 'keep-alive');
      response.setHeader('X-Session-Id', sessionId);

      // Send session_info as first event
      response.write(
        `data: ${JSON.stringify({ event: 'session_info', session_id: sessionId })}\n\n`,
      );

      // Pipe upstream SSE to client
      streamResponse.data.on('data', (chunk) => {
        response.write(chunk);
      });

      streamResponse.data.on('end', () => {
        response.end();
      });

      streamResponse.data.on('error', (err) => {
        console.error('Daemon SSE stream error:', err.message);
        response.end();
      });
    } catch (error) {
      if (error instanceof HttpException) {
        throw error;
      }

      const status = axios.isAxiosError(error)
        ? error.response?.status || HttpStatus.INTERNAL_SERVER_ERROR
        : HttpStatus.INTERNAL_SERVER_ERROR;

      const message = axios.isAxiosError(error)
        ? error.response?.data?.detail ||
          error.response?.data?.error ||
          error.message
        : error.message || 'Internal server error';

      throw new HttpException(message, status);
    }
  }
}
```

- [ ] **Step 2: Register in UserAgentModule**

Open `user-agent.module.ts`. Add the import and register the controller:

Add import at top:
```typescript
import { DaemonChatController } from './daemon-chat.controller';
```

Add to the `controllers` array:
```typescript
  controllers: [
    UserAgentController,
    ApiAgentController,
    ApiSuperagentController,
    AgentRuntimeController,
    SharedAgentController,
    DaemonChatController,  // <-- add this
  ],
```

- [ ] **Step 3: Verify NestJS compiles**

```bash
cd gigabrain-backend && npx tsc --noEmit
```

Expected: No errors.

- [ ] **Step 4: Commit**

```bash
git add gigabrain-backend/src/user-agent/daemon-chat.controller.ts gigabrain-backend/src/user-agent/user-agent.module.ts
git commit -m "feat(nestjs): add DaemonChatController for JWT-authed SSE streaming"
```

---

## Chunk 3: Frontend — Hook & Components

### Task 6: Create useDaemonChat hook

**Files:**
- Create: `gigabrain-frontend/src/app/(dashboard)/agents/[id]/hooks/useDaemonChat.ts`

- [ ] **Step 1: Create the hook**

This hook manages session state, SSE streaming, and history loading. Pattern follows `useChatV2.ts` for SSE parsing.

```typescript
'use client'

import { useState, useCallback, useRef, useEffect } from 'react'
import { gql, useLazyQuery } from '@apollo/client'
import { isTokenValid } from '@/utils/auth'

// --- Types ---

export interface ToolCallData {
  id: string
  toolName: string
  toolArgs: Record<string, any>
  result?: string
  error?: boolean
  status: 'running' | 'completed'
}

export interface DaemonChatMessage {
  id: string
  role: 'user' | 'assistant'
  content: string
  toolCalls?: ToolCallData[]
  created_at: string
  pending?: boolean
}

interface UseDaemonChatReturn {
  messages: DaemonChatMessage[]
  isStreaming: boolean
  isLoadingHistory: boolean
  hasMore: boolean
  sendMessage: (text: string) => Promise<void>
  loadMoreHistory: () => Promise<void>
}

// --- GraphQL ---

const GET_CHAT_HISTORY = gql`
  query ChatHistory($agentId: String!, $sessionId: String!, $limit: Int, $before: String) {
    chatHistory(agentId: $agentId, sessionId: $sessionId, limit: $limit, before: $before)
  }
`

// --- Helpers ---

function getSessionKey(agentId: string): string {
  return `superagent_session_${agentId}`
}

function getStoredSessionId(agentId: string): string | null {
  if (typeof window === 'undefined') return null
  return localStorage.getItem(getSessionKey(agentId))
}

function storeSessionId(agentId: string, sessionId: string): void {
  if (typeof window !== 'undefined') {
    localStorage.setItem(getSessionKey(agentId), sessionId)
  }
}

function getAccessToken(): string | null {
  if (typeof window === 'undefined') return null
  const token = localStorage.getItem('accessToken')
  return token && isTokenValid(token) ? token : null
}

// --- Hook ---

export function useDaemonChat(agentId: string): UseDaemonChatReturn {
  const [messages, setMessages] = useState<DaemonChatMessage[]>([])
  const [isStreaming, setIsStreaming] = useState(false)
  const [isLoadingHistory, setIsLoadingHistory] = useState(false)
  const [hasMore, setHasMore] = useState(false)
  const sessionIdRef = useRef<string | null>(getStoredSessionId(agentId))
  const initialLoadDone = useRef(false)

  const [fetchHistory] = useLazyQuery(GET_CHAT_HISTORY, {
    fetchPolicy: 'network-only',
  })

  // Load initial history on mount
  useEffect(() => {
    if (initialLoadDone.current) return
    initialLoadDone.current = true

    const sid = getStoredSessionId(agentId)
    if (sid) {
      sessionIdRef.current = sid
      loadHistory(sid)
    }
  }, [agentId]) // eslint-disable-line react-hooks/exhaustive-deps

  const loadHistory = useCallback(
    async (sessionId: string, before?: string) => {
      setIsLoadingHistory(true)
      try {
        const { data } = await fetchHistory({
          variables: { agentId, sessionId, limit: 10, before },
        })

        const history = data?.chatHistory
        if (!history?.messages?.length) {
          setHasMore(false)
          return
        }

        const parsed: DaemonChatMessage[] = history.messages.map(
          (m: any, i: number) => ({
            id: m.id || `${m.role}-${Date.parse(m.created_at)}-${i}`,
            role: m.role,
            content: m.content || '',
            toolCalls: m.tool_calls?.map((tc: any, j: number) => ({
              id: `tc-${i}-${j}`,
              toolName: tc.tool_name,
              toolArgs: tc.tool_args || {},
              result: tc.result,
              error: tc.error || false,
              status: 'completed' as const,
            })),
            created_at: m.created_at,
          }),
        )

        setHasMore(history.has_more)

        if (before) {
          // Prepend older messages
          setMessages((prev) => [...parsed, ...prev])
        } else {
          // Initial load
          setMessages(parsed)
        }
      } catch (err) {
        console.error('Failed to load chat history:', err)
      } finally {
        setIsLoadingHistory(false)
      }
    },
    [agentId, fetchHistory],
  )

  const loadMoreHistory = useCallback(async () => {
    if (!sessionIdRef.current || isLoadingHistory || !hasMore) return
    const oldest = messages[0]?.created_at
    await loadHistory(sessionIdRef.current, oldest)
  }, [hasMore, isLoadingHistory, messages, loadHistory])

  const sendMessage = useCallback(
    async (text: string) => {
      if (!text.trim() || isStreaming) return

      const accessToken = getAccessToken()
      if (!accessToken) return

      // Ensure session ID exists
      if (!sessionIdRef.current) {
        const newId = crypto.randomUUID()
        sessionIdRef.current = newId
        storeSessionId(agentId, newId)
      }

      const sessionId = sessionIdRef.current!

      // Add user message to state
      const userMsg: DaemonChatMessage = {
        id: `user-${Date.now()}-${crypto.randomUUID()}`,
        role: 'user',
        content: text,
        created_at: new Date().toISOString(),
      }

      // Add assistant placeholder
      const assistantMsg: DaemonChatMessage = {
        id: `assistant-${Date.now()}-${crypto.randomUUID()}`,
        role: 'assistant',
        content: '',
        toolCalls: [],
        created_at: new Date().toISOString(),
        pending: true,
      }

      setMessages((prev) => [...prev, userMsg, assistantMsg])
      setIsStreaming(true)

      try {
        const apiUrl = process.env.NEXT_PUBLIC_API_URL || ''
        const resp = await fetch(`${apiUrl}/daemon-chat/${agentId}`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${accessToken}`,
          },
          body: JSON.stringify({
            message: text,
            session_id: sessionId,
            stream: true,
          }),
        })

        if (!resp.ok) {
          let errorMsg = `HTTP ${resp.status}`
          try {
            const errBody = await resp.json()
            errorMsg = errBody?.message || errBody?.detail || errorMsg
          } catch {}
          throw new Error(errorMsg)
        }

        if (!resp.body) throw new Error('No response body')

        const reader = resp.body.getReader()
        const decoder = new TextDecoder()
        let buffer = ''
        let toolCallIndex = 0

        while (true) {
          const { value, done } = await reader.read()
          if (done) break

          buffer += decoder.decode(value, { stream: true })
          const lines = buffer.split('\n')
          buffer = lines.pop() || ''

          for (const line of lines) {
            const trimmed = line.trim()
            if (!trimmed.startsWith('data: ')) continue

            const eventData = trimmed.slice(6)
            if (!eventData || eventData === '[DONE]') continue

            try {
              const event = JSON.parse(eventData)
              const eventType = event.event

              switch (eventType) {
                case 'session_info': {
                  // Confirm/update session ID
                  if (event.session_id && event.session_id !== sessionId) {
                    sessionIdRef.current = event.session_id
                    storeSessionId(agentId, event.session_id)
                  }
                  break
                }

                case 'ToolCallStarted': {
                  const toolCalls = event.tool_calls || []
                  const newCalls: ToolCallData[] = toolCalls.map(
                    (tc: any, i: number) => ({
                      id: tc.id || `tc-live-${toolCallIndex + i}`,
                      toolName: tc.function?.name || 'unknown',
                      toolArgs: tc.function?.arguments
                        ? typeof tc.function.arguments === 'string'
                          ? JSON.parse(tc.function.arguments)
                          : tc.function.arguments
                        : {},
                      status: 'running' as const,
                    }),
                  )
                  toolCallIndex += toolCalls.length

                  setMessages((prev) => {
                    const updated = [...prev]
                    const last = updated[updated.length - 1]
                    if (last?.role === 'assistant') {
                      updated[updated.length - 1] = {
                        ...last,
                        toolCalls: [...(last.toolCalls || []), ...newCalls],
                      }
                    }
                    return updated
                  })
                  break
                }

                case 'ToolCallCompleted': {
                  const executions = event.tool_executions || []
                  setMessages((prev) => {
                    const updated = [...prev]
                    const last = updated[updated.length - 1]
                    if (last?.role === 'assistant' && last.toolCalls) {
                      const updatedCalls = [...last.toolCalls]

                      for (const exec of executions) {
                        // Match by tool_call_id or by name (first unresolved)
                        const idx = updatedCalls.findIndex(
                          (tc) =>
                            tc.status === 'running' &&
                            (exec.tool_call_id
                              ? tc.id === exec.tool_call_id
                              : tc.toolName === exec.tool_name),
                        )
                        if (idx >= 0) {
                          updatedCalls[idx] = {
                            ...updatedCalls[idx],
                            status: 'completed',
                            result: exec.result,
                            error: Boolean(exec.tool_call_error),
                          }
                        }
                      }

                      updated[updated.length - 1] = {
                        ...last,
                        toolCalls: updatedCalls,
                      }
                    }
                    return updated
                  })
                  break
                }

                case 'AssistantResponse':
                case 'RunContent':
                case 'RunResponseContent': {
                  // Content chunk — append to assistant message
                  const chunk = event.content || ''
                  if (chunk) {
                    setMessages((prev) => {
                      const updated = [...prev]
                      const last = updated[updated.length - 1]
                      if (last?.role === 'assistant') {
                        updated[updated.length - 1] = {
                          ...last,
                          content: last.content + chunk,
                        }
                      }
                      return updated
                    })
                  }
                  break
                }

                case 'stream_complete': {
                  setMessages((prev) => {
                    const updated = [...prev]
                    const last = updated[updated.length - 1]
                    if (last?.role === 'assistant') {
                      updated[updated.length - 1] = {
                        ...last,
                        pending: false,
                      }
                    }
                    return updated
                  })
                  break
                }

                case 'error': {
                  setMessages((prev) => {
                    const updated = [...prev]
                    const last = updated[updated.length - 1]
                    if (last?.role === 'assistant') {
                      updated[updated.length - 1] = {
                        ...last,
                        content:
                          last.content +
                          `\n\n**Error:** ${event.message || 'Unknown error'}`,
                        pending: false,
                      }
                    }
                    return updated
                  })
                  break
                }

                default:
                  // Ignore unknown events (forward-compatible)
                  break
              }
            } catch {
              // Ignore unparseable events
            }
          }
        }
      } catch (err: any) {
        // Set error on assistant message
        setMessages((prev) => {
          const updated = [...prev]
          const last = updated[updated.length - 1]
          if (last?.role === 'assistant') {
            updated[updated.length - 1] = {
              ...last,
              content: `Failed to get response: ${err.message}`,
              pending: false,
            }
          }
          return updated
        })
      } finally {
        setIsStreaming(false)
      }
    },
    [agentId, isStreaming],
  )

  return {
    messages,
    isStreaming,
    isLoadingHistory,
    hasMore,
    sendMessage,
    loadMoreHistory,
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add gigabrain-frontend/src/app/\(dashboard\)/agents/\[id\]/hooks/useDaemonChat.ts
git commit -m "feat(frontend): add useDaemonChat hook with SSE streaming + history"
```

### Task 7: Create ToolCallPill component

**Files:**
- Create: `gigabrain-frontend/src/app/(dashboard)/agents/[id]/components/ToolCallPill.tsx`

- [ ] **Step 1: Create the component**

```tsx
'use client'

import { useState } from 'react'
import { LuLoader2 } from 'react-icons/lu'
import type { ToolCallData } from '../hooks/useDaemonChat'

/** Pretty-print tool args as a short summary string */
function summarizeArgs(args: Record<string, any>): string {
  const vals = Object.values(args)
  if (vals.length === 0) return ''
  const summary = vals
    .map((v) => (typeof v === 'string' ? v : JSON.stringify(v)))
    .join(', ')
  return summary.length > 50 ? summary.slice(0, 47) + '...' : summary
}

export default function ToolCallPill({ tool }: { tool: ToolCallData }) {
  const [expanded, setExpanded] = useState(tool.error === true)
  const isRunning = tool.status === 'running'
  const isError = tool.error === true

  return (
    <div className="mb-1">
      {/* Pill header */}
      <button
        onClick={() => !isRunning && setExpanded((e) => !e)}
        className={`w-full flex items-center gap-2 px-2.5 py-1.5 rounded-md text-left transition-colors ${
          expanded ? 'rounded-b-none' : ''
        } ${
          isError
            ? 'bg-red-500/5 border border-red-500/20 hover:bg-red-500/10'
            : 'bg-[#0d0d0d] border border-[#1a1a1a] hover:bg-[#111]'
        }`}
      >
        {/* Status indicator */}
        {isRunning ? (
          <LuLoader2 className="w-3 h-3 text-brand animate-spin flex-shrink-0" />
        ) : (
          <div
            className={`w-1.5 h-1.5 rounded-full flex-shrink-0 ${
              isError ? 'bg-red-400' : 'bg-brand'
            }`}
          />
        )}

        {/* Tool name */}
        <span
          className={`text-xs font-mono truncate ${
            isError ? 'text-red-300' : isRunning ? 'text-[#ccc]' : 'text-[#888]'
          }`}
        >
          {tool.toolName}
        </span>

        {/* Args summary */}
        {!expanded && (
          <span className="text-[11px] text-[#444] ml-auto truncate max-w-[200px]">
            {summarizeArgs(tool.toolArgs)}
          </span>
        )}

        {/* Chevron */}
        {!isRunning && (
          <span className="text-[10px] text-[#444] ml-auto flex-shrink-0">
            {expanded ? '▾' : '▸'}
          </span>
        )}
      </button>

      {/* Expanded content */}
      {expanded && !isRunning && (
        <div
          className={`border border-t-0 rounded-b-md px-3 py-2.5 font-mono text-[11px] leading-relaxed max-h-[200px] overflow-y-auto ${
            isError
              ? 'bg-red-500/[0.03] border-red-500/20'
              : 'bg-[#080808] border-[#1a1a1a]'
          }`}
        >
          {/* Args */}
          {Object.keys(tool.toolArgs).length > 0 && (
            <>
              <div className="text-[#555] mb-1">// Input</div>
              <pre className="text-[#aaa] whitespace-pre-wrap break-all mb-2">
                {JSON.stringify(tool.toolArgs, null, 2)}
              </pre>
            </>
          )}

          {/* Result */}
          {tool.result && (
            <>
              <div className="text-[#555] mb-1">
                {isError ? '// Error' : '// Result'}
              </div>
              <pre
                className={`whitespace-pre-wrap break-all ${
                  isError ? 'text-red-300' : 'text-[#aaa]'
                }`}
              >
                {tool.result}
              </pre>
            </>
          )}
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add gigabrain-frontend/src/app/\(dashboard\)/agents/\[id\]/components/ToolCallPill.tsx
git commit -m "feat(frontend): add ToolCallPill component with collapse/expand"
```

### Task 8: Create DaemonChatTab component

**Files:**
- Create: `gigabrain-frontend/src/app/(dashboard)/agents/[id]/components/DaemonChatTab.tsx`

- [ ] **Step 1: Create the component**

```tsx
'use client'

import { useState, useRef, useEffect, useCallback } from 'react'
import { LuLoader2 } from 'react-icons/lu'
import { ArrowUp } from 'lucide-react'
import { useDaemonChat } from '../hooks/useDaemonChat'
import ToolCallPill from './ToolCallPill'
import ChatMarkdownFormatter from '../../../chat/components/ChatMarkdownFormatter'

interface DaemonChatTabProps {
  agentId: string
  agentName: string
  agentStatus: string
}

export default function DaemonChatTab({
  agentId,
  agentName,
  agentStatus,
}: DaemonChatTabProps) {
  const {
    messages,
    isStreaming,
    isLoadingHistory,
    hasMore,
    sendMessage,
    loadMoreHistory,
  } = useDaemonChat(agentId)

  const [input, setInput] = useState('')
  const messagesEndRef = useRef<HTMLDivElement>(null)
  const scrollContainerRef = useRef<HTMLDivElement>(null)
  const inputRef = useRef<HTMLTextAreaElement>(null)
  const isAtBottomRef = useRef(true)

  const isLive = agentStatus === 'live'
  const canSend = isLive && !isStreaming && input.trim().length > 0

  // Auto-scroll to bottom when new messages arrive (if user is at bottom)
  useEffect(() => {
    if (isAtBottomRef.current) {
      messagesEndRef.current?.scrollIntoView({ behavior: 'auto' })
    }
  }, [messages])

  // Track scroll position to know if user is at bottom
  const handleScroll = useCallback(() => {
    const el = scrollContainerRef.current
    if (!el) return
    const threshold = 100
    isAtBottomRef.current =
      el.scrollHeight - el.scrollTop - el.clientHeight < threshold
  }, [])

  // Auto-resize textarea
  useEffect(() => {
    const textarea = inputRef.current
    if (textarea) {
      textarea.style.height = 'auto'
      const maxHeight = 200
      textarea.style.height =
        Math.min(textarea.scrollHeight, maxHeight) + 'px'
    }
  }, [input])

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (!canSend) return

    const text = input.trim()
    setInput('')
    isAtBottomRef.current = true
    await sendMessage(text)
  }

  const handleKeyDown = (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault()
      handleSubmit(e)
    }
  }

  // Load more on scroll to top
  const handleScrollToTop = useCallback(() => {
    const el = scrollContainerRef.current
    if (!el || isLoadingHistory || !hasMore) return
    if (el.scrollTop < 50) {
      const prevHeight = el.scrollHeight
      loadMoreHistory().then(() => {
        // Preserve scroll position after prepending
        requestAnimationFrame(() => {
          el.scrollTop = el.scrollHeight - prevHeight
        })
      })
    }
  }, [isLoadingHistory, hasMore, loadMoreHistory])

  useEffect(() => {
    const el = scrollContainerRef.current
    if (!el) return
    el.addEventListener('scroll', handleScrollToTop)
    return () => el.removeEventListener('scroll', handleScrollToTop)
  }, [handleScrollToTop])

  return (
    <div className="flex flex-col h-[calc(100vh-280px)] min-h-[400px]">
      {/* Messages */}
      <div
        ref={scrollContainerRef}
        onScroll={handleScroll}
        className="flex-1 overflow-y-auto px-4 py-4"
      >
        {/* Load more indicator */}
        {isLoadingHistory && (
          <div className="flex justify-center py-3">
            <LuLoader2 className="w-4 h-4 text-[#555] animate-spin" />
          </div>
        )}

        {/* Empty state */}
        {messages.length === 0 && !isLoadingHistory && (
          <div className="flex flex-col items-center justify-center h-full text-center">
            <div className="w-12 h-12 rounded-full bg-[#111] flex items-center justify-center mb-4">
              <span className="text-xl">🧠</span>
            </div>
            <p className="text-[#888] text-sm mb-1">
              {isLive
                ? 'Send a message to start chatting'
                : 'Start the agent to begin chatting'}
            </p>
            <p className="text-[#555] text-xs">
              Your conversation persists across page reloads
            </p>
          </div>
        )}

        {/* Message list */}
        {messages.map((msg) => (
          <div key={msg.id} className="flex gap-2.5 mb-5">
            {/* Avatar */}
            <div
              className={`w-6 h-6 rounded-full flex-shrink-0 flex items-center justify-center text-[11px] ${
                msg.role === 'user'
                  ? 'bg-[#1a1a1a] border border-[#333] text-[#888]'
                  : 'bg-gradient-to-br from-[#1a3a2a] to-[#0d1f16] border border-[#1a3a2a]'
              }`}
            >
              {msg.role === 'user' ? 'U' : '🧠'}
            </div>

            <div className="flex-1 min-w-0">
              {/* Name */}
              <div
                className={`text-xs mb-1.5 ${
                  msg.role === 'user' ? 'text-[#666]' : 'text-brand'
                }`}
              >
                {msg.role === 'user' ? 'You' : agentName}
              </div>

              {/* Tool calls (assistant only) */}
              {msg.role === 'assistant' && msg.toolCalls?.map((tc) => (
                <ToolCallPill key={tc.id} tool={tc} />
              ))}

              {/* Content */}
              {msg.content ? (
                <div className="text-sm text-[#e0e0e0] leading-relaxed">
                  <ChatMarkdownFormatter content={msg.content} />
                </div>
              ) : msg.pending ? (
                <div className="flex items-center gap-2 mt-2">
                  <LuLoader2 className="w-3 h-3 text-brand animate-spin" />
                  <span className="text-xs text-[#555]">Thinking...</span>
                </div>
              ) : null}
            </div>
          </div>
        ))}

        <div ref={messagesEndRef} />
      </div>

      {/* Input */}
      <div className="border-t border-[#1a1a1a] px-4 py-3">
        {!isLive ? (
          <div className="text-center py-2">
            <p className="text-[#555] text-sm">
              Agent must be running to chat
            </p>
          </div>
        ) : (
          <form onSubmit={handleSubmit}>
            <div className="flex items-end gap-2 bg-[#0d0d0d] border border-[#222] rounded-xl px-3 py-2.5">
              <textarea
                ref={inputRef}
                value={input}
                onChange={(e) => setInput(e.target.value)}
                onKeyDown={handleKeyDown}
                placeholder="Send a message..."
                rows={1}
                disabled={isStreaming}
                className="flex-1 bg-transparent text-sm text-white placeholder-[#555] resize-none outline-none min-h-[20px] max-h-[200px]"
              />
              <button
                type="submit"
                disabled={!canSend}
                className={`w-7 h-7 rounded-lg flex items-center justify-center flex-shrink-0 transition-colors ${
                  canSend
                    ? 'bg-brand text-black hover:bg-brand/90'
                    : 'bg-[#1a1a1a] text-[#555]'
                }`}
              >
                <ArrowUp className="w-4 h-4" />
              </button>
            </div>
          </form>
        )}
      </div>
    </div>
  )
}
```

**Note:** The import `ChatMarkdownFormatter` may need a path adjustment. Check where the component is exported from. If it doesn't accept a `content` prop directly, check its props interface in `gigabrain-frontend/src/app/(dashboard)/chat/components/ChatMarkdownFormatter.tsx` and adapt.

- [ ] **Step 2: Commit**

```bash
git add gigabrain-frontend/src/app/\(dashboard\)/agents/\[id\]/components/DaemonChatTab.tsx
git commit -m "feat(frontend): add DaemonChatTab component with message list + input"
```

### Task 9: Wire DaemonChatTab into the agent detail page

**Files:**
- Modify: `gigabrain-frontend/src/app/(dashboard)/agents/[id]/page.tsx`

- [ ] **Step 1: Add import**

At the top of `page.tsx`, add:

```typescript
import DaemonChatTab from './components/DaemonChatTab'
```

- [ ] **Step 2: Replace the Telegram placeholder**

Find the block:
```tsx
{activeTab === 'chat' && (
  <div className="flex flex-col items-center justify-center py-16">
    <div className="w-16 h-16 rounded-full bg-[#1a1a1a] flex items-center justify-center mb-5">
      <FaTelegram className="w-8 h-8 text-[#666]" />
    </div>
    ...
  </div>
)}
```

Replace the entire block with:
```tsx
{activeTab === 'chat' && (
  <DaemonChatTab
    agentId={id}
    agentName={agent.name || 'Agent'}
    agentStatus={agent.agent_status}
  />
)}
```

- [ ] **Step 3: Clean up unused imports**

Remove `FaTelegram` import if it's no longer used elsewhere in the file. Check if `ExternalLink` from lucide-react is still used — if not, remove it too.

Search the file: if `FaTelegram` only appeared in the removed block, remove:
```typescript
import { FaTelegram } from 'react-icons/fa'
```

And remove `ExternalLink` from the lucide-react import if unused.

- [ ] **Step 4: Verify build**

```bash
cd gigabrain-frontend && npx next build 2>&1 | tail -20
```

If the build fails due to the `ChatMarkdownFormatter` import path, fix the import path based on the error message.

- [ ] **Step 5: Commit**

```bash
git add gigabrain-frontend/src/app/\(dashboard\)/agents/\[id\]/page.tsx
git commit -m "feat(frontend): wire DaemonChatTab into agent detail page"
```

---

## Chunk 4: Verification & Polish

### Task 10: End-to-end verification

- [ ] **Step 1: Verify NestJS compiles**

```bash
cd gigabrain-backend && npx tsc --noEmit
```

Expected: No errors.

- [ ] **Step 2: Verify frontend builds**

```bash
cd gigabrain-frontend && npx next build 2>&1 | tail -30
```

Expected: Build succeeds.

- [ ] **Step 3: Log Agno SSE events to verify event names**

Before deploying, temporarily add logging to confirm the exact Agno event names. In `telegram.py`'s `_execute_agent_stream`, add a log before the yield:

```python
logger.info(f"SSE event: {payload.get('event', 'unknown')}")
```

Send a test message to the daemon and check logs. Verify:
- Content events: confirm the event name (`AssistantResponse` vs `RunContent` vs other)
- Tool call events: confirm `ToolCallStarted` and `ToolCallCompleted` appear
- Confirm `tool_calls[].id` field exists for matching

**If event names differ from what the frontend handles**, update the switch cases in `useDaemonChat.ts`.

Remove the temporary logging after verification.

- [ ] **Step 4: Test chat flow manually**

1. Navigate to `/agents/[id]` for a live daemon agent
2. Chat tab should show empty state: "Send a message to start chatting"
3. Send a message → should see tool call pills appearing with spinners
4. Tool calls complete → green dots, content streams in below
5. Switch to another tab (e.g., Feed) → switch back → messages still there (React state)
6. Reload the page → messages load from history (brief loading spinner, then messages appear)
7. Test with a paused agent → input should be disabled with "Agent must be running to chat"

- [ ] **Step 5: Test error states**

1. Send a message that triggers an error (e.g., ask the agent to do something impossible)
2. Verify error tool calls show red dot + auto-expand
3. Verify error messages display inline
4. Verify network errors show "Failed to get response: ..." message
