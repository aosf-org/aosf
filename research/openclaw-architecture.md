# OpenClaw Architecture Document

> **Version:** February 2026 · **Source:** Local docs, GitHub, docs.openclaw.ai  
> **Reading time:** ~12 minutes

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Major Capabilities](#2-major-capabilities)
3. [Privacy & Data Safety](#3-privacy--data-safety)
4. [Use Case Walkthrough](#4-use-case-walkthrough)
5. [Conversation Privacy Model](#5-conversation-privacy-model)

---

## 1. Architecture Overview

OpenClaw is a **local-first, single-user personal AI assistant** that you run on your own machine. It bridges your existing messaging channels (WhatsApp, Telegram, Discord, Slack, Signal, iMessage, and more) to a coding-agent runtime (Pi), with a single long-lived **Gateway** process acting as the control plane.

### High-Level Architecture

```
 ┌─────────────────────────────────────────────────────────┐
 │                   MESSAGING CHANNELS                    │
 │  WhatsApp · Telegram · Discord · Slack · Signal         │
 │  iMessage · Google Chat · MS Teams · Matrix · WebChat   │
 └────────────────────────┬────────────────────────────────┘
                          │ Inbound messages
                          ▼
 ┌────────────────────────────────────────────────────────┐
 │                    GATEWAY DAEMON                      │
 │            ws://127.0.0.1:18789 (loopback)             │
 │                                                        │
 │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ │
 │  │ Channel  │ │ Session  │ │  Cron /  │ │  Pairing  │ │
 │  │ Manager  │ │  Store   │ │ Scheduler│ │  & Auth   │ │
 │  └──────────┘ └──────────┘ └──────────┘ └───────────┘ │
 │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ │
 │  │  Agent   │ │  Tools   │ │  Skills  │ │  Config   │ │
 │  │  Router  │ │  Engine  │ │  Loader  │ │  Schema   │ │
 │  └──────────┘ └──────────┘ └──────────┘ └───────────┘ │
 └────────┬──────────┬───────────┬──────────┬────────────┘
          │          │           │          │
          ▼          ▼           ▼          ▼
   ┌──────────┐ ┌─────────┐ ┌────────┐ ┌────────────┐
   │ Pi Agent │ │  CLI    │ │ macOS  │ │ iOS/Android│
   │  (RPC)   │ │ Client  │ │  App   │ │   Nodes    │
   └──────────┘ └─────────┘ └────────┘ └────────────┘
```

### Core Components

| Component | Role |
|-----------|------|
| **Gateway** | Single long-lived daemon. Owns all channel connections, WebSocket control plane, session store, cron scheduler, and config. Binds to loopback by default. |
| **Pi Agent** | Embedded agent runtime (from pi-mono). Runs in RPC mode with tool streaming and block streaming. Handles LLM calls. |
| **Channels** | Protocol adapters: Baileys (WhatsApp), grammY (Telegram), discord.js, Bolt (Slack), signal-cli, etc. Each channel normalizes messages into a common envelope. |
| **Sessions** | Per-conversation state. DMs collapse into a shared "main" session; group chats get isolated session keys. Transcripts stored as JSONL files. |
| **Tools/Skills** | First-class tool system: exec, read/write, browser, canvas, nodes, cron, web search/fetch. Skills are discoverable `SKILL.md` folders that teach the agent how to use tools. |
| **Nodes** | Companion devices (macOS/iOS/Android) that connect over WebSocket and expose device commands (camera, canvas, screen recording, location, system.run). |

### Message Flow

```
 User                Channel             Gateway              Agent            LLM Provider
  │                    │                    │                    │                    │
  │  "Research X"      │                    │                    │                    │
  │───────────────────>│                    │                    │                    │
  │                    │  Normalized msg    │                    │                    │
  │                    │───────────────────>│                    │                    │
  │                    │                    │  Route to agent    │                    │
  │                    │                    │  (session lookup)  │                    │
  │                    │                    │───────────────────>│                    │
  │                    │                    │                    │  API call          │
  │                    │                    │                    │  (system prompt +  │
  │                    │                    │                    │   context + msg)   │
  │                    │                    │                    │───────────────────>│
  │                    │                    │                    │                    │
  │                    │                    │                    │<───────────────────│
  │                    │                    │                    │  Response/tools    │
  │                    │                    │<───────────────────│                    │
  │                    │  Formatted reply   │                    │                    │
  │                    │<───────────────────│                    │                    │
  │  Reply arrives     │                    │                    │                    │
  │<───────────────────│                    │                    │                    │
```

### WebSocket Protocol

The Gateway exposes a typed WebSocket API. All clients — CLI, macOS app, Control UI, nodes — connect to the same endpoint.

```
Client                          Gateway
  │                                │
  │──── req:connect ──────────────>│   (auth token + device identity)
  │<─── res:connect (hello-ok) ────│   (presence + health snapshot)
  │                                │
  │<─── event:presence ────────────│   (push events)
  │<─── event:tick ────────────────│
  │                                │
  │──── req:agent ────────────────>│   (message to process)
  │<─── event:agent ───────────────│   (streaming blocks)
  │<─── res:agent (final) ─────────│   (complete response)
  │                                │
```

- **Transport:** WebSocket, text frames with JSON payloads
- **First frame must be `connect`** with auth token (if configured)
- **Nodes** connect with `role: "node"` and declare their capabilities
- **Events** are server-push (presence, agent streaming, health)

---

## 2. Major Capabilities

### 2.1 Multi-Channel Support

OpenClaw acts as a unified inbox across messaging platforms. Each channel has its own protocol adapter:

```
 ┌─────────────────────────────────────────────────┐
 │              Channel Adapters                    │
 │                                                  │
 │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
 │  │ WhatsApp │  │ Telegram │  │ Discord  │      │
 │  │ (Baileys)│  │ (grammY) │  │(discordjs│      │
 │  └──────────┘  └──────────┘  └──────────┘      │
 │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
 │  │  Slack   │  │  Signal  │  │ iMessage │      │
 │  │  (Bolt)  │  │(signal-  │  │(BlueBub- │      │
 │  │          │  │  cli)    │  │  bles)   │      │
 │  └──────────┘  └──────────┘  └──────────┘      │
 │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
 │  │ Google   │  │  MS      │  │  Matrix  │      │
 │  │  Chat    │  │  Teams   │  │          │      │
 │  └──────────┘  └──────────┘  └──────────┘      │
 │  ┌──────────┐  ┌──────────┐                     │
 │  │  WebChat │  │  Zalo    │                     │
 │  │          │  │          │                     │
 │  └──────────┘  └──────────┘                     │
 └─────────────────────────────────────────────────┘
```

**Key features:**
- DMs and groups are handled per-channel with configurable policies
- Group chats support mention-based activation (bot only responds when @mentioned)
- Media pipeline handles images, audio, video, and documents across channels
- Voice notes can be auto-transcribed
- Block streaming sends completed text blocks as they finish (Telegram supports draft-style streaming)

### 2.2 Agent System

OpenClaw supports **multi-agent routing** — multiple isolated agents sharing one Gateway:

```
                    ┌──────────────────┐
                    │     Gateway      │
                    │                  │
                    │   ┌──────────┐   │
                    │   │ Bindings │   │  Route by: channel, accountId,
                    │   │  Router  │   │  peer id, guild/team
                    │   └────┬─────┘   │
                    │        │         │
                    └────────┼─────────┘
                   ┌─────────┼─────────┐
                   ▼         ▼         ▼
            ┌──────────┐ ┌──────────┐ ┌──────────┐
            │ Agent A  │ │ Agent B  │ │ Agent C  │
            │ "Home"   │ │ "Work"   │ │ "Family" │
            │          │ │          │ │          │
            │ Own:     │ │ Own:     │ │ Own:     │
            │ ·Workspace│ │ ·Workspace│ │ ·Workspace│
            │ ·Sessions│ │ ·Sessions│ │ ·Sessions│
            │ ·Auth    │ │ ·Auth    │ │ ·Auth    │
            │ ·Skills  │ │ ·Skills  │ │ ·Skills  │
            │ ·Persona │ │ ·Persona │ │ ·Persona │
            └──────────┘ └──────────┘ └──────────┘
```

Each agent has its own:
- **Workspace** (AGENTS.md, SOUL.md, USER.md, memory files)
- **Session store** (conversation history)
- **Auth profiles** (API keys, OAuth tokens)
- **Skills** (workspace-level overrides)
- **Persona** (identity, boundaries, tone)

**Sub-agents** can be spawned for parallel/background work:
- Run in isolated sessions (`agent:<agentId>:subagent:<uuid>`)
- Cannot spawn further sub-agents (no recursive fan-out)
- Announce results back to the requester channel
- Auto-archive after configurable timeout
- Can use a different (cheaper) model than the main agent

### 2.3 Tool & Skill System

```
 ┌─────────────────────────────────────────────────────┐
 │                 TOOL SYSTEM                          │
 │                                                      │
 │  Built-in Tools         Extended Tools               │
 │  ─────────────          ──────────────               │
 │  · read/write/edit      · browser (Chrome/Chromium)  │
 │  · exec/process         · canvas (A2UI)              │
 │  · web_search/fetch     · nodes (camera, screen)     │
 │  · image analysis       · cron (scheduler)           │
 │  · message (channels)   · sessions (spawn/send)      │
 │  · tts (text-to-speech) · memory_search/get          │
 │                                                      │
 │  Skills (SKILL.md folders)                           │
 │  ─────────────────────────                           │
 │  · Bundled:   shipped with install                   │
 │  · Managed:   ~/.openclaw/skills                     │
 │  · Workspace: <workspace>/skills                     │
 │  · ClawHub:   public registry (clawhub.com)          │
 │                                                      │
 │  Precedence: workspace > managed > bundled           │
 └─────────────────────────────────────────────────────┘
```

Skills are **AgentSkills-compatible** folders containing a `SKILL.md` with YAML frontmatter. They can be gated by:
- Required binaries on PATH
- Required environment variables
- Required config flags
- OS restrictions

### 2.4 Memory & Context Management

OpenClaw memory is **plain Markdown in the workspace**:

```
 ~/.openclaw/workspace/
 ├── AGENTS.md          ← Operating instructions
 ├── SOUL.md            ← Persona, tone, boundaries
 ├── USER.md            ← User profile
 ├── IDENTITY.md        ← Agent name/vibe
 ├── TOOLS.md           ← Tool-specific notes
 ├── MEMORY.md          ← Curated long-term memory
 ├── BOOTSTRAP.md       ← One-time first-run ritual
 └── memory/
     ├── 2026-02-03.md  ← Daily log (yesterday)
     └── 2026-02-04.md  ← Daily log (today)
```

**Context** = everything sent to the model per run:
- System prompt (rules, tools, skills list, runtime info)
- Injected workspace files (AGENTS.md, SOUL.md, etc.)
- Conversation history (session JSONL)
- Tool call results and attachments

**Memory features:**
- **Vector memory search** — semantic search over MEMORY.md and daily logs using embeddings (OpenAI, Gemini, or local GGUF model)
- **Hybrid search** — BM25 keyword + vector similarity for best results
- **Pre-compaction flush** — silent agent turn to write durable memories before context is compacted
- **Session memory** (experimental) — index session transcripts for cross-session recall

### 2.5 Cron & Scheduling

The Gateway has a built-in scheduler:

- **One-shot reminders** (`schedule.kind = "at"`) — "remind me in 20 minutes"
- **Recurring jobs** (`schedule.kind = "cron"` or `"every"`) — "every morning at 7am"
- **Main session** jobs enqueue system events into the heartbeat loop
- **Isolated** jobs run dedicated agent turns in their own session (`cron:<jobId>`)
- Jobs persist to `~/.openclaw/cron/jobs.json` — survive restarts
- Can deliver output to any channel (WhatsApp, Telegram, etc.)
- Support model and thinking-level overrides per job

### 2.6 Browser Automation

OpenClaw can control a **dedicated, isolated Chrome/Chromium profile**:

- Separate from your personal browser (orange accent profile named "openclaw")
- Agent can open tabs, navigate, click, type, drag, select
- Snapshot DOM for accessibility-tree-based interaction
- Screenshot and PDF generation
- Two profiles: `openclaw` (managed, isolated) or `chrome` (extension relay to your system browser)
- Browser control runs through a local service inside the Gateway (loopback only)

### 2.7 Node Pairing (Companion Devices)

```
 ┌──────────────┐     WebSocket      ┌──────────────┐
 │   Gateway    │◄───────────────────►│  macOS Node  │
 │   (Host)     │                     │  (Menu Bar)  │
 └──────────────┘                     └──────────────┘
        ▲                                    
        │ WebSocket                          
        ▼                                    
 ┌──────────────┐                     ┌──────────────┐
 │  iOS Node    │                     │ Android Node │
 │  (App)       │                     │  (App)       │
 └──────────────┘                     └──────────────┘

 Node Capabilities:
 ─────────────────
 · canvas.present / canvas.snapshot / canvas.eval (A2UI)
 · camera.snap (front/back/both) / camera.clip (video)
 · screen.record (mp4, up to 60s)
 · location.get (lat/lon/accuracy)
 · system.run / system.notify (macOS)
 · sms.send (Android)
```

Nodes connect over WebSocket with `role: "node"` and require **device pairing** (approval via CLI or UI). Exec approvals gate which commands can run on each node.

### 2.8 Canvas (UI Presentation)

Canvas is an agent-driven visual workspace:

- **A2UI** (Agent-to-UI) push/reset for rendering agent-generated content
- Hosted on a separate HTTP port (default 18793)
- Available on macOS, iOS, and Android nodes
- Supports eval (run JavaScript), snapshot (capture rendered state), and navigation
- Enables rich visual output beyond text replies

---

## 3. Privacy & Data Safety

### 3.1 Local-First Architecture

```
 ┌──────────────────────────────────────────────────────┐
 │              YOUR MACHINE (Local)                     │
 │                                                       │
 │  ┌─────────────────────────────────────────────────┐  │
 │  │           ~/.openclaw/                          │  │
 │  │                                                  │  │
 │  │  openclaw.json          ← Config (JSON5)        │  │
 │  │  workspace/             ← Agent workspace       │  │
 │  │  agents/<id>/sessions/  ← Conversation history  │  │
 │  │  agents/<id>/agent/     ← Auth profiles         │  │
 │  │  credentials/           ← Channel credentials   │  │
 │  │  cron/                  ← Scheduled jobs         │  │
 │  │  memory/<id>.sqlite     ← Vector search index   │  │
 │  │  skills/                ← Managed skills         │  │
 │  │  extensions/            ← Installed plugins      │  │
 │  └─────────────────────────────────────────────────┘  │
 │                                                       │
 │  Gateway daemon (ws://127.0.0.1:18789)               │
 │  Canvas host   (http://127.0.0.1:18793)              │
 └──────────────────────┬───────────────────────────────┘
                        │
                        │ Only outbound connection:
                        │ API calls to LLM provider
                        ▼
              ┌───────────────────┐
              │   LLM Provider    │
              │ (Anthropic/OpenAI)│
              │                   │
              │ Receives: system  │
              │ prompt + context  │
              │ + current message │
              └───────────────────┘
```

**Everything stays local** except the LLM API call:
- Conversation history → local JSONL files
- Memory files → local Markdown
- Config & credentials → local JSON files
- Session store → local JSON
- Cron jobs → local JSON
- Vector search index → local SQLite

**The Gateway binds to loopback (127.0.0.1) by default** — not accessible from the network unless you explicitly configure remote access (Tailscale, SSH tunnel).

### 3.2 Credential & API Key Management

```
 Credential Storage Map
 ──────────────────────────────────────────────────────
 WhatsApp creds    → ~/.openclaw/credentials/whatsapp/<accountId>/creds.json
 Telegram token    → config/env or channels.telegram.tokenFile
 Discord token     → config/env
 Slack tokens      → config/env
 Pairing allowlists→ ~/.openclaw/credentials/<channel>-allowFrom.json
 Model auth        → ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
 Gateway token     → config (gateway.auth.token) or env
 ──────────────────────────────────────────────────────

 Recommended permissions:
 · ~/.openclaw/          → 700 (user only)
 · ~/.openclaw/openclaw.json → 600 (user read/write)
 · credentials/*.json    → 600
 · auth-profiles.json    → 600
```

- Auth profiles are **per-agent** — each agent has its own API keys/OAuth tokens
- Gateway auth token protects the WebSocket control plane
- `openclaw security audit` checks for exposed credentials and permission issues
- Skills inject env vars scoped to agent runs (not globally)

### 3.3 Session Isolation Model

```
 ┌────────────────────────────────────────────────────────────┐
 │                    SESSION ISOLATION                        │
 │                                                             │
 │  DM Sessions (dmScope = "main"):                           │
 │  ┌─────────────────────────────────────────────┐           │
 │  │  agent:main:main                            │           │
 │  │  (All DMs share one session for continuity) │           │
 │  └─────────────────────────────────────────────┘           │
 │                                                             │
 │  Group Sessions (always isolated):                          │
 │  ┌──────────────────────┐ ┌──────────────────────┐         │
 │  │agent:main:whatsapp:  │ │agent:main:telegram:  │         │
 │  │group:<group-id>      │ │group:<chat-id>       │         │
 │  └──────────────────────┘ └──────────────────────┘         │
 │  ┌──────────────────────┐ ┌──────────────────────┐         │
 │  │agent:main:discord:   │ │agent:main:slack:     │         │
 │  │channel:<channel-id>  │ │channel:<channel-id>  │         │
 │  └──────────────────────┘ └──────────────────────┘         │
 │                                                             │
 │  Sub-Agent Sessions:                                        │
 │  ┌──────────────────────────────────────┐                  │
 │  │ agent:main:subagent:<uuid>           │                  │
 │  │ (Fully isolated, fresh each run)     │                  │
 │  └──────────────────────────────────────┘                  │
 │                                                             │
 │  Cron Sessions:                                             │
 │  ┌──────────────────────────────────────┐                  │
 │  │ cron:<jobId>                         │                  │
 │  │ (Fresh session ID per run)           │                  │
 │  └──────────────────────────────────────┘                  │
 └────────────────────────────────────────────────────────────┘
```

### 3.4 Access Control: DM Pairing

```
 Stranger                     Gateway                    Owner
    │                            │                          │
    │  "Hello bot"               │                          │
    │───────────────────────────>│                          │
    │                            │  DM policy = "pairing"  │
    │  "Please pair: CODE-1234"  │                          │
    │<───────────────────────────│                          │
    │                            │                          │
    │        (message NOT processed)                        │
    │                            │                          │
    │                            │  openclaw pairing        │
    │                            │  approve CODE-1234       │
    │                            │<─────────────────────────│
    │                            │                          │
    │  "Hello bot" (retry)       │  Sender now in allowlist │
    │───────────────────────────>│───────> Process normally │
```

**DM policies:**
- `pairing` (default) — unknown senders get a pairing code; ignored until approved
- `allowlist` — only configured senders; no pairing handshake
- `open` — anyone (requires explicit `"*"` in allowlist)
- `disabled` — ignore all inbound DMs

### 3.5 Configuration Security

- **Strict schema validation** — Gateway refuses to start with invalid config
- **`openclaw security audit`** — flags risky DM/group policies, network exposure, permission issues
- **`openclaw security audit --fix`** — auto-tightens common footguns
- **Sandboxing** — optional Docker-based tool isolation per agent
- **Tool allow/deny lists** — restrict which tools each agent can use
- **Exec approvals** — allowlist-based gating for shell command execution on nodes

---

## 4. Use Case Walkthrough

### Scenario: Chris asks Micky to research a topic via Telegram

Here's what happens step by step when Chris sends "Research the latest developments in WebAssembly" to Micky on Telegram:

```
 ┌─────────┐          ┌──────────┐         ┌──────────┐
 │  Chris  │          │ Telegram │         │ Gateway  │
 │(Phone)  │          │ Bot API  │         │(Chris's  │
 │         │          │          │         │  Mac)    │
 └────┬────┘          └────┬─────┘         └────┬─────┘
      │                    │                    │
  1.  │ "Research the      │                    │
      │  latest WebAssembly│                    │
      │  developments"     │                    │
      │───────────────────>│                    │
      │                    │                    │
  2.  │                    │  grammY adapter    │
      │                    │  normalizes msg    │
      │                    │───────────────────>│
      │                    │                    │
  3.  │                    │         ┌──────────┤
      │                    │         │ Session  │
      │                    │         │ lookup:  │
      │                    │         │ agent:   │
      │                    │         │ main:main│
      │                    │         └──────────┤
      │                    │                    │
  4.  │                    │         ┌──────────┤
      │                    │         │ Build    │
      │                    │         │ context: │
      │                    │         │ ·System  │
      │                    │         │  prompt  │
      │                    │         │ ·AGENTS  │
      │                    │         │  .md     │
      │                    │         │ ·SOUL.md │
      │                    │         │ ·History │
      │                    │         │ ·Skills  │
      │                    │         └──────────┤
      │                    │                    │
  5.  │                    │                    │──── API call ────>  Anthropic
      │                    │                    │                     (Claude)
      │                    │                    │                        │
      │                    │                    │<─── Response ──────────│
      │                    │                    │     (with tool calls)  │
      │                    │                    │                        │
  6.  │                    │         ┌──────────┤
      │                    │         │ Execute  │
      │                    │         │ tools:   │
      │                    │         │ ·web_    │
      │                    │         │  search  │
      │                    │         │ ·web_    │
      │                    │         │  fetch   │
      │                    │         └──────────┤
      │                    │                    │
  7.  │                    │                    │──── API call ────>  Anthropic
      │                    │                    │  (tool results +    (Claude)
      │                    │                    │   ask for summary)     │
      │                    │                    │<─── Final reply ───────│
      │                    │                    │
  8.  │                    │  Formatted reply   │
      │                    │<───────────────────│
      │                    │                    │
  9.  │  Research summary  │                    │
      │  arrives in chat   │                    │
      │<───────────────────│                    │
      │                    │                    │
 10.  │                    │         ┌──────────┤
      │                    │         │ Append   │
      │                    │         │ to JSONL │
      │                    │         │ transcript│
      │                    │         └──────────┘
```

### Privacy at each step:

| Step | What happens | Privacy status |
|------|-------------|----------------|
| 1-2 | Message goes Telegram → Gateway | Telegram sees the message (standard Telegram behavior). Gateway runs on Chris's Mac. |
| 3 | Session lookup | Local only. Session store is a file on Chris's Mac. |
| 4 | Context assembly | Local only. Workspace files, history — all read from local disk. |
| 5 | LLM API call | **Only data that leaves the machine.** System prompt + context + message sent to Anthropic's API. |
| 6 | Tool execution | Local only. Web search/fetch runs from Gateway process on Chris's Mac. |
| 7 | Follow-up LLM call | Tool results sent to Anthropic for summarization. |
| 8-9 | Reply delivery | Reply sent via Telegram Bot API back to Chris. |
| 10 | Transcript saved | Local only. JSONL file on Chris's Mac at `~/.openclaw/agents/main/sessions/`. |

---

## 5. Conversation Privacy Model

### 5.1 What Stays Local vs. What Leaves

```
 ┌──────────────────────────────────────────────────────────┐
 │                STAYS ON YOUR MACHINE                      │
 │                                                           │
 │  ✓ All conversation history (JSONL transcripts)          │
 │  ✓ Memory files (MEMORY.md, daily logs)                  │
 │  ✓ Agent workspace files (AGENTS.md, SOUL.md, etc.)      │
 │  ✓ Configuration (openclaw.json)                         │
 │  ✓ Channel credentials (WhatsApp session, bot tokens)    │
 │  ✓ Model auth profiles (API keys, OAuth tokens)          │
 │  ✓ Pairing allowlists                                    │
 │  ✓ Cron job definitions and run history                  │
 │  ✓ Vector search index (SQLite)                          │
 │  ✓ Session metadata and routing state                    │
 │  ✓ Tool execution output and logs                        │
 └──────────────────────────────────────────────────────────┘

 ┌──────────────────────────────────────────────────────────┐
 │              LEAVES YOUR MACHINE                          │
 │                                                           │
 │  → LLM API calls: system prompt + conversation context   │
 │    + current message → sent to Anthropic/OpenAI          │
 │                                                           │
 │  → Channel traffic: replies sent through messaging APIs   │
 │    (Telegram Bot API, WhatsApp Web, Discord, etc.)        │
 │                                                           │
 │  → Web search/fetch: outbound HTTP when using those tools │
 │                                                           │
 │  → Embedding API calls (if using remote vector search)    │
 └──────────────────────────────────────────────────────────┘
```

### 5.2 DM Isolation

By default, all DMs share one main session for **continuity** — your conversation flows naturally across devices and channels. This is safe because the default DM policy is `pairing`, meaning only **you** (or people you explicitly approve) can talk to the bot.

For multi-user setups, session isolation prevents cross-user context leakage:

```
 dmScope: "main" (default — single user)
 ────────────────────────────────────────
 WhatsApp DM ──┐
 Telegram DM ──┤──► agent:main:main (shared)
 Discord DM  ──┘

 dmScope: "per-channel-peer" (multi-user)
 ────────────────────────────────────────
 Alice (WhatsApp) ──► agent:main:whatsapp:dm:+1555...
 Alice (Telegram) ──► agent:main:telegram:dm:12345
 Bob   (WhatsApp) ──► agent:main:whatsapp:dm:+1666...
```

With `session.identityLinks`, you can link the same person across channels so they share one DM session.

### 5.3 Group Chat Isolation

Every group chat automatically gets its own isolated session:

```
 Family group (WhatsApp) ──► agent:main:whatsapp:group:120363...@g.us
 Work channel (Slack)    ──► agent:main:slack:channel:C0123456
 Dev server (Discord)    ──► agent:main:discord:channel:98765432

 No context leaks between:
 · Your DM conversations and any group
 · Different groups
 · Different channels
```

Group chats also support **mention gating** — the bot only activates when @mentioned, preventing it from processing every message in busy groups.

### 5.4 Sub-Agent Isolation

```
 Main Session                    Sub-Agent Session
 ┌────────────────────┐          ┌────────────────────────────┐
 │ agent:main:main    │  spawn   │ agent:main:subagent:<uuid> │
 │                    │─────────>│                            │
 │ Full context:      │          │ Minimal context:           │
 │ · AGENTS.md        │          │ · AGENTS.md                │
 │ · SOUL.md          │          │ · TOOLS.md                 │
 │ · USER.md          │          │ (No SOUL.md, USER.md,      │
 │ · IDENTITY.md      │          │  IDENTITY.md, MEMORY.md)   │
 │ · MEMORY.md        │          │                            │
 │ · Full history     │          │ · Fresh session, no history│
 │                    │          │ · No session tools          │
 │                    │ announce │ · Cannot spawn sub-agents   │
 │                    │<─────────│                            │
 └────────────────────┘          └────────────────────────────┘
```

### 5.5 Multi-Agent Privacy

When multiple agents share one Gateway, each agent is a **fully isolated brain**:

```
 ┌───────────────────────────────────────────────────────┐
 │                    GATEWAY                             │
 │                                                        │
 │  Agent "personal"          Agent "family"              │
 │  ┌────────────────┐        ┌────────────────┐         │
 │  │ Workspace A    │        │ Workspace B    │         │
 │  │ Sessions A     │        │ Sessions B     │         │
 │  │ Auth A         │        │ Auth B         │         │
 │  │ Skills A       │  ╳──╳  │ Skills B       │         │
 │  │ Memory A       │ (no    │ Memory B       │         │
 │  │ Persona A      │ cross- │ Persona B      │         │
 │  └────────────────┘ talk)  └────────────────┘         │
 │                                                        │
 │  Different sandbox policies possible:                  │
 │  · Personal: full access, no sandbox                  │
 │  · Family: sandboxed + read-only tools                │
 └───────────────────────────────────────────────────────┘
```

### 5.6 Security Checklist Summary

| Layer | Default | Configurable |
|-------|---------|--------------|
| Gateway binding | Loopback only (127.0.0.1) | Can expose via Tailscale/SSH |
| Gateway auth | Required (token) | Token or password mode |
| DM policy | Pairing (code-based approval) | Allowlist, open, disabled |
| Group activation | Mention-required | Always-on (not recommended) |
| Session isolation | DMs shared; groups isolated | Per-channel-peer DM isolation |
| Tool execution | Host process | Docker sandbox, per-agent policies |
| Node commands | Pairing + exec approvals | Allowlist-based |
| File permissions | User-only (700/600) | `openclaw security audit --fix` |
| mDNS discovery | Minimal mode | Off, minimal, or full |

---

## Appendix: Quick Reference

### File Locations

```
~/.openclaw/
├── openclaw.json                              ← Main config (JSON5)
├── workspace/                                 ← Default agent workspace
│   ├── AGENTS.md, SOUL.md, USER.md, ...       ← Bootstrap files
│   ├── MEMORY.md                              ← Long-term memory
│   ├── memory/                                ← Daily logs
│   └── skills/                                ← Workspace skills
├── agents/<agentId>/
│   ├── agent/auth-profiles.json               ← Per-agent auth
│   └── sessions/
│       ├── sessions.json                      ← Session store
│       └── <SessionId>.jsonl                  ← Transcripts
├── credentials/                               ← Channel creds
├── cron/
│   ├── jobs.json                              ← Scheduled jobs
│   └── runs/                                  ← Run history
├── skills/                                    ← Managed skills
├── memory/<agentId>.sqlite                    ← Vector index
└── extensions/                                ← Plugins
```

### Default Ports

| Service | Port | Binding |
|---------|------|---------|
| Gateway (WS + HTTP) | 18789 | 127.0.0.1 (loopback) |
| Canvas host | 18793 | 127.0.0.1 (loopback) |

### Key CLI Commands

```bash
openclaw onboard              # Guided setup wizard
openclaw gateway              # Start the Gateway daemon
openclaw doctor               # Diagnose + fix issues
openclaw security audit       # Security audit
openclaw status               # Show current state
openclaw sessions --json      # List all sessions
openclaw nodes status         # List paired nodes
openclaw cron list            # List scheduled jobs
openclaw pairing list         # Show pending pairing requests
openclaw agents list          # List configured agents
```

---

*Generated by Micky · February 2026*
