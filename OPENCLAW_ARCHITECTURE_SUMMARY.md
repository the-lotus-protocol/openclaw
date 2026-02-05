# OpenClaw: How It Works — Architecture Summary

A distilled view of the OpenClaw codebase for quick reference.

---

## 1. What OpenClaw Is

**OpenClaw** is a **personal AI assistant** you run locally. One **Gateway** process owns all messaging surfaces and runs the agent. Clients (CLI, WebChat, macOS app, iOS/Android nodes) connect over **WebSocket** to that Gateway. The “brain” is an **embedded Pi agent** (from pi-mono: `createAgentSession`, tool execution, streaming). Channels (WhatsApp, Telegram, Slack, Discord, etc.) deliver messages **into** the Gateway; the Gateway routes them to sessions and runs the agent; replies go **back out** over the same channel.

- **Single Gateway per host** — one WebSocket server (default `ws://127.0.0.1:18789`), one HTTP server for Control UI + WebChat + optional Canvas host.
- **Channels** = transports (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, WebChat, extensions like BlueBubbles, Matrix, Zalo, etc.).
- **Agent** = one embedded Pi runtime per run; sessions and workspace are OpenClaw-owned.

---

## 2. High-Level Flow

```
[Channels] → Gateway (WS + HTTP) → [Routing] → Session → [Queue] → runEmbeddedPiAgent → [Tools] → Reply
     ↑                                                                                        │
     └────────────────────────── Outbound send (same channel) ◀──────────────────────────────┘
```

- **Inbound:** Channel adapter receives message → allowlist/mention/group gating → **routing** (agent + session key) → **session key** (e.g. `agent:main:main` for DMs, `agent:main:telegram:group:-100…` for groups) → **queue** (lane per session, then global lane) → **auto-reply pipeline** → `getReplyFromConfig` → `runEmbeddedPiAgent`.
- **Outbound:** Agent returns payloads → **dispatcher** (typing, block streaming, etc.) → **channel dock** `send` → back to user on the same channel/thread.

---

## 3. Entry Points & Main Paths

### 3.1 Gateway startup

- **CLI:** `openclaw gateway` (or daemon) → `createGatewayRuntimeState` in `src/gateway/server-runtime-state.ts`.
- **HTTP:** `createGatewayHttpServer` (`server-http.ts`) → `listenGatewayHttpServer` (bind host + port).
- **WebSocket:** `new WebSocketServer({ noServer: true })`; upgrade handled in `attachGatewayUpgradeHandler`. First frame must be **connect** (handshake); then req/res and events.
- **Canvas host** (optional): same HTTP server, path `CANVAS_HOST_PATH`; A2UI for agent-driven UI.

### 3.2 Clients connecting

- **Operators:** CLI, Web UI, macOS app — connect with `role: "operator"`, scopes like `operator.read`, `operator.write`. Send `req: agent`, `req: send`, `req: health`, etc.
- **Nodes:** iOS/Android/macOS node — connect with `role: "node"`, declare `caps` and `commands` (e.g. `camera.snap`, `canvas.navigate`). Gateway invokes them via `node.invoke`.
- **WebChat:** Uses same WS; chat methods (e.g. history, send) are part of the protocol.

### 3.3 Agent runs: two ways

1. **Via Gateway (WS `agent` method)**  
   Client sends `req: agent` with message, sessionKey/sessionId, options.  
   → `server-methods/agent.ts` → `agentCommand` (in `commands/agent.ts`) → same path as CLI agent (runEmbeddedPiAgent or runCliAgent depending on context).  
   Used by WebChat, macOS app, and any WS client.

2. **Via auto-reply (inbound from channel)**  
   Channel receives message → channel-specific monitor (e.g. WhatsApp `monitorWebInbox`, Telegram/Slack/Discord monitors) → **routing** (`resolveAgentRoute` → agentId + sessionKey) → **inbound debouncer/queue** (per session key, then global lane) → **process-message** (e.g. `web/auto-reply/monitor/process-message.ts`) → **dispatchReplyWithBufferedBlockDispatcher** → **dispatchReplyFromConfig** → **getReplyFromConfig** → **runPreparedReply** → **runReplyAgent** → **runEmbeddedPiAgent**.

So: **WS `agent`** and **channel inbound** both end up at the same agent run (either `runEmbeddedPiAgent` or, for CLI-only path, `runCliAgent`). The difference is who triggers it (client vs channel pipeline).

---

## 4. Session & Routing

- **Session key** identifies a conversation bucket. Examples:
  - DM (main): `agent:<agentId>:main` (or configured mainKey).
  - Group: `agent:<agentId>:<channel>:group:<id>`; threads add `:thread:<id>`; Telegram topics `:topic:<id>`.
  - Cron: `cron:<jobId>`; webhook: `hook:<uuid>`.
- **Routing** (see `docs/concepts/channel-routing.md`): For each inbound message, one agent is chosen by bindings (peer, guild, team, account, channel, default). That agent’s workspace and session store are used.
- **Session store:** On gateway host, `~/.openclaw/agents/<agentId>/sessions/sessions.json` (key → sessionId, etc.). Transcripts: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`.
- **Gateway is source of truth** for session state; UIs ask the gateway (e.g. session list, token counts).

---

## 5. Queue & Concurrency

- **Command queue** (`src/process/command-queue.ts`): Lane-based. Runs are enqueued by **session key** (one active run per session) and by a **global lane** (e.g. `main`) to cap concurrency (`agents.defaults.maxConcurrent`).
- **Queue modes** (per session/channel): `collect` (coalesce), `followup`, `steer` (inject into current run), `steer-backlog`, etc. Configured via `messages.queue` and `/queue` command.
- **Serialization:** Prevents multiple agent runs from fighting over the same session and keeps global parallelism bounded.

---

## 6. Pi Agent (Embedded)

- **No subprocess:** OpenClaw imports pi packages (`@mariozechner/pi-agent-core`, `@mariozechner/pi-coding-agent`, etc.) and calls **createAgentSession** / **SessionManager** in process.
- **Entry:** `runEmbeddedPiAgent` in `src/agents/pi-embedded-runner/run.ts`. It:
  - Resolves model (provider/model), workspace, session (sessionId/sessionKey).
  - Enqueues to session lane then global lane.
  - Calls **runEmbeddedAttempt** (`run/attempt.ts`): loads/creates SessionManager, builds system prompt (workspace files, SOUL, AGENTS, TOOLS, skills), injects bootstrap files, limits history, runs **streamSimple** (or equivalent) with tools.
- **Tools:** OpenClaw injects **createOpenClawCodingTools()** (and channel-specific tools). Includes bash, read/write/edit, browser, canvas, nodes, cron, sessions, etc. Tool policy and sandbox are applied per session/agent.
- **Streaming:** Events (e.g. text_delta, tool_call) go to **subscribeEmbeddedPiSession**; handlers drive typing, block chunking, and delivery. Final payloads are returned and then sent to the channel by the **dispatcher**.

---

## 7. Channels (Inbound → Agent)

- **Core channels** (in `src/`): WhatsApp (Baileys in `web/`), Telegram (`telegram/`), Slack (`slack/`), Discord (`discord/`), Signal (`signal/`), iMessage (`imessage/`), WebChat (gateway WS).
- **Extensions** (in `extensions/`): e.g. BlueBubbles, MSteams, Matrix, Zalo, Zalo Personal — loaded as plugins, registered in channel **registry**.
- **Channel dock** (`channels/dock.ts`): Per-channel capabilities (allowlists, groups, mentions, threading, agent prompt). Inbound flow uses dock + **routing** to get agentId + sessionKey, then pushes into the **auto-reply** pipeline (debouncer → process-message → getReplyFromConfig → runEmbeddedPiAgent).
- **Outbound:** Dispatcher calls channel-specific send (e.g. Telegram send, WhatsApp send) with reply payload; chunking and policies (e.g. block streaming, textChunkLimit) applied per channel.

---

## 8. Config & Workspace

- **Config:** `~/.openclaw/openclaw.json` (or path from env). Loaded by `loadConfig()` in `config/config.js`. Contains agents, session, channels, gateway, tools, etc.
- **Workspace:** Per agent, e.g. `agents.defaults.workspace` (often `~/.openclaw/workspace`). Contains **AGENTS.md**, **SOUL.md**, **TOOLS.md**, **BOOTSTRAP.md**, **IDENTITY.md**, **USER.md** — injected into system prompt. Skills live under workspace `skills/` and/or `~/.openclaw/skills` and bundled skills.
- **Credentials:** e.g. `~/.openclaw/credentials` for channel logins; pairing/allowlist stores for DMs.

---

## 9. Security & Pairing

- **Gateway auth:** Optional token (`OPENCLAW_GATEWAY_TOKEN` / `gateway.auth`). All connections (local/remote) can require token.
- **Pairing:** New device IDs (e.g. new CLI instance, new node) get a challenge; after approval they receive a **device token** and can reconnect. Local (loopback) can be auto-approved.
- **Channels:** DM policy (e.g. pairing vs open), allowlists (`allowFrom`, `groupAllowFrom`). Run `openclaw doctor` to check risky configs.
- **Sandbox:** Non-main sessions can run in Docker or restricted tool policy; main session typically has full tools on host.

---

## 10. Key Files (Quick Map)

| Area               | Key files                                                                                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CLI entry**      | `src/index.ts` → `buildProgram()` (cli/program.js)                                                                                                   |
| **Gateway server** | `server-runtime-state.ts`, `server-http.ts`, `server/ws-connection.ts`, `server/http-listen.ts`                                                      |
| **WS protocol**    | `gateway/protocol/`, `server-methods/agent.ts`, `server-methods.ts`                                                                                  |
| **Agent command**  | `commands/agent.ts` → `agentCommand`; gateway calls this for `req: agent`                                                                            |
| **Embedded Pi**    | `agents/pi-embedded-runner/run.ts`, `run/attempt.ts`, `pi-tools.ts`, `pi-embedded-subscribe.ts`                                                      |
| **Auto-reply**     | `auto-reply/reply/get-reply.ts`, `reply/get-reply-run.ts`, `reply/agent-runner.ts`, `reply/dispatch-from-config.ts`, `dispatch.ts`                   |
| **Channels**       | `channels/dock.ts`, `channels/registry.ts`; `web/inbound/monitor.ts`, `web/auto-reply/monitor/process-message.ts`; `telegram/`, `slack/`, `discord/` |
| **Routing**        | `routing/resolve-route.ts`, `routing/session-key.ts`                                                                                                 |
| **Sessions**       | `config/sessions.js`, `gateway/session-utils.ts`, `gateway/sessions-resolve.ts`                                                                      |
| **Queue**          | `process/command-queue.ts`; queue mode in `auto-reply/` and config `messages.queue`                                                                  |

---

## 11. Docs (in repo)

- **Architecture:** `docs/concepts/architecture.md`, `docs/gateway/protocol.md`
- **Agent:** `docs/concepts/agent.md`, `docs/pi.md`
- **Sessions:** `docs/concepts/session.md`, `docs/concepts/channel-routing.md`
- **Queue:** `docs/concepts/queue.md`
- **Streaming:** `docs/concepts/streaming.md`
- **Config:** `docs/gateway/configuration.md`, reference in `docs/reference/`

This file is a **summary**; for exact behavior and config keys, rely on the code and the official docs above.
