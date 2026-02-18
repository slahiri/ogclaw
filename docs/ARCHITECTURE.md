# OpenClaw Agent Architecture

## Overview

OpenClaw is a self-hosted, local-first personal AI assistant platform. It is **not** a traditional agentic framework with planners, state machines, or DAGs. Instead, it relies entirely on the **raw power of the LLM** to orchestrate tool calls. The "agent" is a **prompt assembly recipe + configuration wrapper** around an LLM API call.

An agent = **System Prompt (assembled from markdown files + hardcoded sections) + Conversation History (JSONL) + Tool Definitions (filtered by policy)**. The LLM is the only decision-maker. OpenClaw never decides anything on its own.

---

## Tech Stack

- **Core platform**: TypeScript (ESM), Node 22+, pnpm
- **macOS/iOS apps**: Swift (407 files)
- **Android app**: Kotlin (77 files)
- **Tooling**: Go (docs i18n), Python (skill scripts), Shell (build/deploy)
- **Testing**: Vitest with V8 coverage
- **Linting**: Oxlint + Oxfmt
- **Versioning**: Date-based (`v2026.2.17`)
- **Agent runtime**: `@mariozechner/pi-agent-core` and `@mariozechner/pi-coding-agent`

---

## High-Level Data Flow

```
Messaging Channels (WhatsApp / Telegram / Discord / Slack / Signal / iMessage / ...)
                                   |
                                   v
                          Channel Plugin
                    (parse, normalize, extract threading)
                                   |
                                   v
                        Session Key Resolution
                 "agent:main:telegram:default:direct:user123"
                  (encodes: agent + channel + account + peer + thread)
                                   |
                                   v
                          Agent Command Handler
               (load history, resolve model, build prompt, select auth)
                                   |
                                   v
                            LLM Provider
                  (Anthropic / OpenAI / Gemini / Ollama / ...)
                     Streams events: lifecycle, tool, assistant, error
                                   |
                     +-------------+-------------+
                     v             v             v
                WebSocket     Node Subs      Channel Hook
                Broadcast     (Mobile)       (Reply back)
               (Control UI)
```

---

## Full Agent Architecture

```
+==============================================================================+
|                         OPENCLAW AGENT ARCHITECTURE                           |
+==============================================================================+

+------------------------------------------------------------------------------+
|                           INBOUND MESSAGE                                     |
|  (WhatsApp / Telegram / Discord / Slack / Signal / CLI / WebChat / ...)      |
+--------------------------------------+---------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                     1. AGENT RESOLUTION                                       |
|                                                                              |
|  Session Key --> Agent ID --> Config Merge (agent entry + global defaults)   |
|  "agent:main:telegram:default:direct:user123"                                |
|                                                                              |
|  Resolves: model, tools policy, skills, sandbox, identity, workspace         |
+--------------------------------------+---------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                 2. CONTEXT ASSEMBLY (System Prompt)                           |
|                                                                              |
|  +----------------------------------------------------------------------+   |
|  |                    WORKSPACE FILES (Markdown)                         |   |
|  |                                                                      |   |
|  |  SOUL.md --------- Personality, tone, behavioral guidelines          |   |
|  |  IDENTITY.md ----- Name, emoji, avatar, creature type                |   |
|  |  AGENTS.md ------- Multi-agent instructions, Red Lines, constraints  |   |
|  |  BOOTSTRAP.md ---- Dynamic codebase/project context                  |   |
|  |  USER.md --------- User preferences and context                      |   |
|  |  TOOLS.md -------- Tool usage guidance                                |   |
|  |  HEARTBEAT.md ---- Periodic task instructions                         |   |
|  |  MEMORY.md ------- Long-term memory entries                           |   |
|  |                                                                      |   |
|  |  Budget: 20K chars/file, 150K chars total                            |   |
|  |  Truncation: Head 70% + Tail 20% when over limit                    |   |
|  +----------------------------------------------------------------------+   |
|                              +                                               |
|  +----------------------------------------------------------------------+   |
|  |                    HARDCODED PROMPT SECTIONS                          |   |
|  |                                                                      |   |
|  |  - Identity line ("You are a personal assistant...")                  |   |
|  |  - Tool listing with summaries                                        |   |
|  |  - Safety guardrails                                                  |   |
|  |  - CLI quick reference                                                |   |
|  |  - Skills instructions                                                |   |
|  |  - Memory recall instructions                                        |   |
|  |  - Messaging/reply protocol                                          |   |
|  |  - Sandbox info (if containerized)                                   |   |
|  |  - Runtime info (model, OS, channel, timezone)                       |   |
|  |  - Silent reply / heartbeat protocol                                 |   |
|  +----------------------------------------------------------------------+   |
|                              =                                               |
|                    ONE BIG SYSTEM PROMPT STRING                              |
+--------------------------------------+---------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                    3. SESSION HISTORY (JSONL)                                 |
|                                                                              |
|  ~/.openclaw/state/agents/{agentId}/sessions/{sessionKey}.jsonl              |
|                                                                              |
|  Line 1: {"type":"session","version":2,"id":"...","cwd":"..."}              |
|  Line 2: {"type":"message","message":{"role":"user","content":"..."}}        |
|  Line 3: {"type":"message","message":{"role":"assistant","content":"..."}}   |
|  Line N: {"type":"message","message":{"role":"toolResult","content":"..."}}  |
|                                                                              |
|  - Append-only (one JSON line per message)                                   |
|  - History limits per channel/DM configurable                                |
|  - Provider-specific turn validation (Anthropic vs Gemini)                   |
|  - Oversized tool results auto-truncated                                     |
+--------------------------------------+---------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                 4. LLM EXECUTION (PI Framework)                              |
|                                                                              |
|     system_prompt + session_history + user_message                           |
|                        |                                                     |
|                        v                                                     |
|               +----------------+                                             |
|               |   LLM API      |<-- Auth Profile Rotation                    |
|               |  (streaming)   |    (cooldown-aware cycling)                 |
|               +-------+--------+                                             |
|                       |                                                      |
|              +--------+--------+                                             |
|              v                 v                                              |
|        Text Response     Tool Call(s)                                        |
|              |                 |                                              |
|              |                 v                                              |
|              |          +------------+                                        |
|              |          | Execute    |                                        |
|              |          | Tool       |                                        |
|              |          | (sandbox?) |                                        |
|              |          +------+-----+                                        |
|              |                 |                                              |
|              |                 v                                              |
|              |          Tool Result --> Append to history                     |
|              |                 |                                              |
|              |                 v                                              |
|              |          Back to LLM (loop until text response)               |
|              |                                                               |
|              v                                                               |
|        Final Response --> Append to JSONL                                    |
|                                                                              |
|  Model Fallback: Primary --> Fallback 1 --> Fallback 2 --> Error             |
+--------------------------------------+---------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                      5. TOOLS AVAILABLE TO LLM                               |
|                                                                              |
|  +-------------+  +-------------+  +-------------+  +--------------+        |
|  | File I/O    |  | Shell       |  | Browser     |  | Messaging    |        |
|  | read        |  | exec        |  | browse      |  | message      |        |
|  | write       |  | process     |  | screenshot  |  | sessions_*   |        |
|  | edit        |  |             |  | navigate    |  | subagents    |        |
|  | apply_patch |  |             |  |             |  |              |        |
|  | grep / find |  |             |  |             |  |              |        |
|  | ls          |  |             |  |             |  |              |        |
|  +-------------+  +-------------+  +-------------+  +--------------+        |
|  +-------------+  +-------------+  +-------------+  +--------------+        |
|  | Web         |  | Memory      |  | Scheduling  |  | System       |        |
|  | web_search  |  | memory_     |  | cron        |  | gateway      |        |
|  | web_fetch   |  |   search    |  |             |  | session_     |        |
|  |             |  | memory_get  |  |             |  |   status     |        |
|  |             |  |             |  |             |  | canvas       |        |
|  |             |  |             |  |             |  | nodes        |        |
|  |             |  |             |  |             |  | image        |        |
|  +-------------+  +-------------+  +-------------+  +--------------+        |
|                                                                              |
|  Filtered by: tool policy (minimal/coding/messaging/full) + allow/deny      |
+--------------------------------------+---------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                    6. MEMORY SYSTEM (Long-Term)                              |
|                                                                              |
|  MEMORY.md -----------+                                                      |
|  memory/*.md ---------+-- Chunk (400 tokens, 80 overlap)                    |
|  session transcripts--+          |                                           |
|                                  v                                           |
|                    +------------------------+                                |
|                    |  Embedding Provider     |                                |
|                    |  (OpenAI / Gemini /     |                                |
|                    |   Voyage / Local)       |                                |
|                    +-----------+------------+                                |
|                                |                                             |
|                                v                                             |
|                    +------------------------+                                |
|                    |   SQLite Database       |                                |
|                    |  - files (metadata)     |                                |
|                    |  - chunks (vectors)     |                                |
|                    |  - fts_chunks (BM25)    |                                |
|                    |  - embedding_cache      |                                |
|                    +-----------+------------+                                |
|                                |                                             |
|              +-----------------+------------------+                          |
|              v                 v                   v                          |
|       Vector Search      BM25 Full-Text      Hybrid Fusion                  |
|       (cosine sim)       (keyword match)     (0.7v + 0.3t)                  |
|                                                                              |
|  Sync modes: onSessionStart | onSearch | watch | intervalMinutes            |
+--------------------------------------+---------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                 7. COMPACTION (Context Overflow Recovery)                    |
|                                                                              |
|  Triggered when: context window exceeded (token limit hit)                  |
|                                                                              |
|  STEP 1: Memory Flush (optional, pre-compaction)                            |
|  - Prompt agent: "save durable memories to memory/YYYY-MM-DD.md"            |
|  - Agent writes important context before history is lost                    |
|                                                                              |
|  STEP 2: Safeguard Collection                                               |
|  - Collect last 8 tool failure summaries                                    |
|  - Track file operations (read/modified files)                              |
|  - Extract Red Lines from AGENTS.md                                         |
|                                                                              |
|  STEP 3: Chunk & Summarize                                                  |
|  - Split history into balanced token-share chunks                           |
|  - Summarize each chunk independently                                       |
|  - Merge partial summaries into final summary                               |
|  - Adaptive chunk ratio: 15%-40% of context window                         |
|                                                                              |
|  STEP 4: Replace History                                                    |
|  - Old messages removed from JSONL                                          |
|  - Summary injected as first message of new session                         |
|  - Recent messages preserved after summary                                  |
|                                                                              |
|  Max 3 compaction attempts per run                                          |
+--------------------------------------+---------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                    8. MULTI-AGENT (Sub-Agent Spawning)                       |
|                                                                              |
|  Parent Session                    Child Session                             |
|  +------------------+             +------------------+                       |
|  | agent:main:...   |---spawn--->| agent:X:subagent  |                      |
|  |                  |             |   :{uuid}         |                      |
|  |                  |             |                  |                       |
|  | calls            |  limits:    | - Own history    |                       |
|  | sessions_spawn() |  depth=1    | - Own tools      |                       |
|  |                  |  children=5 | - Own context    |                       |
|  |                  |  concurrent | - Minimal prompt |                       |
|  |                  |   =8        |   mode           |                       |
|  |                  |<--announce--+                  |                       |
|  | receives result  |             | completes & auto-|                       |
|  | in context       |             | announces back   |                       |
|  +------------------+             +------------------+                       |
|                                                                              |
|  No shared state. No agent-to-agent protocol. Just isolated LLM sessions.  |
+--------------------------------------+---------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                       9. OUTBOUND RESPONSE                                   |
|                                                                              |
|  Final text --> Channel formatting --> Chunking --> Channel API send         |
|                 (mentions, threading,   (4000 char     (Telegram,            |
|                  reply tags)            limit etc.)    Discord, etc.)        |
+------------------------------------------------------------------------------+
```

---

## Agent Definition

An agent is defined in `config.yaml` under `agents.list[]`:

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4"
      fallbacks: ["openai/gpt-4"]
    workspace: ~/.openclaw/workspace
    tools:
      profile: "full"
    subagents:
      maxSpawnDepth: 1
      maxConcurrent: 8
      maxChildrenPerAgent: 5

  list:
    - id: main
      default: true
      workspace: ~/my-workspace
      model:
        primary: "anthropic/claude-opus-4"
      skills: ["coding-agent", "github"]

    - id: pr-reviewer
      name: "PR Reviewer"
      workspace: /repos/project
      model: "anthropic/claude-3-sonnet"
      skills: ["review-pr"]
      identity:
        name: "PR Bot"
        emoji: "🔍"
      tools:
        profile: "coding"
      subagents:
        allowAgents: ["main"]
```

### Agent Properties (AgentConfig)

| Property | Purpose |
|----------|---------|
| `id` | Unique identifier (lowercase alphanumeric/dash, max 64 chars) |
| `default` | Whether this is the default agent |
| `name` | Human-readable display name |
| `workspace` | Working directory for this agent |
| `agentDir` | State directory (auth profiles, session store) |
| `model` | Primary model + fallback chain |
| `skills` | Allowlist of skills this agent can use |
| `identity` | Name, emoji, avatar, vibe |
| `tools` | Tool profile (minimal/coding/messaging/full) + allow/deny lists |
| `sandbox` | Docker isolation settings |
| `subagents` | Which agents it can spawn, model for children |
| `heartbeat` | Periodic background task config |
| `memorySearch` | Vector memory/RAG config |
| `humanDelay` | Simulated typing delays |
| `groupChat` | Group chat behavior settings |

### What Differentiates One Agent From Another

| What changes | Effect |
|---|---|
| Different `SOUL.md` | Different personality/instructions |
| Different `model` | Different LLM (Opus vs Sonnet vs GPT-4) |
| Different `tools.profile` | Different tool access (minimal vs full) |
| Different `skills` | Different skill allowlist |
| Different `workspace` | Different working directory |
| Different `sandbox` | Sandboxed vs host access |
| Different channel binding | Which chat platforms it handles |

---

## Agent Routing via Bindings

Different agents handle different channels/peers:

```yaml
bindings:
  - agentId: main
    match:
      channel: slack

  - agentId: pr-reviewer
    match:
      channel: discord
      guildId: "123456"
      peer:
        kind: "channel"
        id: "pr-review-channel"
```

---

## Agent Lifecycle (Detailed)

### Phase 1: Initialization
- Workspace resolution (user-specified or default `~/.openclaw/workspace`)
- Model selection + fallback chain setup
- Auth profile rotation init (cooldown-aware cycling)
- Hook execution (`before_model_resolve`, `before_agent_start`)
- Context window validation (min 16K tokens, warn at 32K)

### Phase 2: Context Assembly
- Load bootstrap files (AGENTS.md, SOUL.md, BOOTSTRAP.md, TOOLS.md, IDENTITY.md, USER.md, HEARTBEAT.md, MEMORY.md)
- Build system prompt (runtime info, tools, skills, safety, messaging, voice)
- Resolve tools + apply policies (per-agent, per-channel, owner-only)
- Initialize session manager (load JSONL history)
- Apply history limits (per-channel/DM configurable)
- Validate turn ordering (provider-specific: Anthropic vs Gemini)

### Phase 3: Extension Setup
- Register compaction-safeguard extension (if enabled)
- Register context-pruning extension (if cache-ttl mode)

### Phase 4: Model Execution Loop
- **Auth profile rotation**: Try profile N, if auth error check cooldown, try N+1
- **API call**: Send system_prompt + history + tools to LLM (streaming)
- **Tool execution loop**: Parse tool calls, execute, append results, call LLM again
- **Message persistence**: Append user/assistant/toolResult messages to JSONL

### Phase 5: Overflow Detection & Compaction
- If context_overflow_error: trigger compaction (max 3 attempts)
- Optional memory flush: prompt agent to save memories before compaction
- Chunk history by token share, summarize in stages, replace old history

### Phase 6: Return & Cleanup
- Report usage metrics (input, output, cache tokens)
- Return final assistant response
- Clean up temp files, restore workspace directory

---

## Memory System Details

### Storage
- **Files**: MEMORY.md + memory/*.md (user-editable markdown)
- **Index**: SQLite database with vector embeddings + BM25 full-text search
- **Chunking**: 400 tokens per chunk, 80 token overlap

### Embedding Providers
- OpenAI (text-embedding-3-small)
- Google Gemini (gemini-embedding-001)
- Voyage (voyage-4-large)
- Local model (requires download)

### Search
- **Hybrid fusion**: 0.7 vector weight + 0.3 text weight (configurable)
- **Max results**: 6 (default)
- **Min score**: 0.35 threshold
- **Optional**: MMR for diversity, temporal decay for recency bias

### Sync Strategies
- `onSessionStart` - sync on agent startup
- `onSearch` - sync before searches
- `watch` - file system watch with 1500ms debounce
- `intervalMinutes` - periodic background sync

### Memory Tools (LLM-accessible)
- `memory_search` - semantic search over memory files, returns snippets with paths + line numbers
- `memory_get` - safe line-range extraction from memory files

### Memory Flush (Pre-Compaction)
- Triggered when tokens approach context limit
- Prompts the agent: "save durable memories to memory/YYYY-MM-DD.md"
- Agent appends important context before history is summarized away
- Runs only once per compaction cycle

---

## Compaction Details

### Trigger
- Context overflow error from LLM API (token limit exceeded)
- Max 3 compaction attempts per agent run

### Process
1. **Safeguard collection**: last 8 tool failures, file operations, Red Lines from AGENTS.md
2. **Adaptive chunking**: split messages by token share (15%-40% of context window)
3. **Multi-stage summarization**: summarize each chunk, merge partial summaries
4. **History replacement**: old messages removed, summary injected as first message
5. **Fallback**: if summarization fails, note message counts/sizes as minimal context

### Context Pruning (In-Memory Only)
- Separate from compaction; runs on each request
- Removes old/prunable tool results from current context window
- Does NOT modify persisted JSONL file
- Mode: "cache-ttl" based on elapsed time since last cache touch

---

## Gateway (Control Plane)

The Gateway is the central WebSocket + HTTP server (`ws://127.0.0.1:18789`):

- **Channel management**: starts/stops/restarts channels with exponential backoff
- **Message routing**: routes inbound messages to correct agent session
- **Event streaming**: broadcasts agent events to connected clients (web UI, mobile, CLI)
- **HTTP APIs**: OpenAI-compatible `POST /v1/chat/completions`, extended `POST /v1/responses`
- **Auth**: API keys, rate limiting, CORS, allowlists
- **Discovery**: Bonjour/mDNS (LAN), Tailscale (WAN)
- **Node subscriptions**: remote nodes subscribe to session events

---

## Channel Abstraction

Channels use an adapter pattern with a "Dock" spec:

| Adapter | Purpose |
|---------|---------|
| `ChannelMessagingAdapter` | Send/receive messages |
| `ChannelThreadingAdapter` | Thread/reply-to support |
| `ChannelGroupAdapter` | Group chat handling |
| `ChannelMentionAdapter` | @-mention normalization |
| `ChannelHeartbeatAdapter` | Keep-alive pings |
| `ChannelSecurityAdapter` | DM pairing, security policies |
| `ChannelAgentToolFactory` | Channel-specific agent tools |

**Core channels**: Telegram (grammY), WhatsApp (Baileys), Discord (discord.js), Slack (Bolt), Signal, iMessage

**Extension channels**: Matrix, Teams, IRC, LINE, Mattermost, Nostr, Feishu, Tlon, Twitch, Zalo

---

## Plugin System

10 extension points:

| Extension | API | Purpose |
|-----------|-----|---------|
| Tools | `api.registerTool()` | Add agent capabilities |
| Hooks | `api.on()` | Intercept 18+ lifecycle events |
| Channels | `api.registerChannel()` | New messaging platforms |
| Providers | `api.registerProvider()` | New LLM backends |
| CLI | `api.registerCli()` | Extend CLI commands |
| Commands | `api.registerCommand()` | Direct slash commands (bypass LLM) |
| Services | `api.registerService()` | Background daemons |
| Gateway Methods | `api.registerGatewayMethod()` | WebSocket handlers |
| HTTP Routes | `api.registerHttpRoute()` | Custom HTTP endpoints |
| HTTP Handlers | `api.registerHttpHandler()` | Request interceptors |

### Key Hooks (18 total)

- `before_model_resolve` - override model/provider selection
- `before_prompt_build` - inject system prompt or context
- `message_received` / `message_sending` / `message_sent` - message pipeline
- `before_tool_call` / `after_tool_call` - modify or block tool use
- `before_compaction` / `after_compaction` - history summarization
- `agent_end` - post-run cleanup
- `session_start` / `session_end` - session lifecycle
- `gateway_start` / `gateway_stop` - server lifecycle

---

## Skills System

50+ bundled skills in `skills/` directory:

- **Productivity**: 1Password, Apple Notes, Apple Reminders, Bear Notes, Notion, Obsidian, Things, Trello, GitHub Issues
- **Communication**: Discord, Slack, iMessage, BlueBubbles
- **Media**: Spotify Player, OpenAI Image Gen, GifGrep, Video Frames, Camera Snap
- **Development**: Coding Agent, GitHub, tmux, Session Logs
- **AI/Automation**: ClawHub, MCPorter (MCP bridge), Oracle, Skill Creator, Summarize
- **Utilities**: Weather, Food Order, Health Check, Model Usage, PDF

Skills are curated tool/prompt bundles published to ClawHub (`clawhub.ai`).

---

## Security Model

- **Default**: tools run directly on host for main session (full access for single user)
- **Groups/channels**: can be sandboxed in per-session Docker containers
- **DM pairing**: unknown senders receive a pairing code, must be approved via `openclaw pairing approve`
- **Tool policies**: minimal/coding/messaging/full profiles + explicit allow/deny lists
- **SSRF protection**: private IP blocking on web_fetch
- **Owner-only tools**: high-risk tools restricted to owner numbers
- **Session file integrity**: JSONL format prevents partial writes, write locks prevent concurrent modification

---

## Key Source Files

| File | Purpose |
|------|---------|
| `src/gateway/server.impl.ts` | Main gateway server initialization |
| `src/gateway/server-chat.ts` | Agent event handler (delta/final/error) |
| `src/gateway/server-channels.ts` | Channel lifecycle management |
| `src/commands/agent.ts` | Main agent command entry point |
| `src/agents/agent-scope.ts` | Agent resolution logic |
| `src/agents/system-prompt.ts` | System prompt assembly (~640 lines) |
| `src/agents/pi-embedded-runner/run.ts` | Main agent execution loop |
| `src/agents/pi-embedded-runner/run/attempt.ts` | Single attempt execution |
| `src/agents/pi-embedded-runner/compact.ts` | Compaction logic |
| `src/agents/pi-extensions/compaction-safeguard.ts` | Compaction context provider |
| `src/agents/pi-extensions/context-pruning/extension.ts` | In-memory pruning |
| `src/agents/tools/memory-tool.ts` | memory_search & memory_get tools |
| `src/agents/workspace.ts` | Workspace file discovery |
| `src/agents/identity-file.ts` | IDENTITY.md parser |
| `src/agents/subagent-spawn.ts` | Sub-agent spawning |
| `src/routing/session-key.ts` | Session key encoding/decoding |
| `src/channels/dock.ts` | Channel behavior specs |
| `src/channels/plugins/types.plugin.ts` | Channel plugin contract |
| `src/plugins/types.ts` | Complete plugin API surface |
| `src/plugins/loader.ts` | Plugin loading mechanics |
| `src/plugins/hooks.ts` | Hook execution engine |
| `src/config/types.agents.ts` | Agent configuration types |
| `src/config/types.memory.ts` | Memory configuration types |
| `src/config/zod-schema.agent-runtime.ts` | Agent validation schemas |
| `src/plugin-sdk/index.ts` | Public SDK exports |
