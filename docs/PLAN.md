# OGClaw: Implementation Plan

## Context

**What**: A Go-based multi-agent platform that enables enterprises to fully manage, operate, and optimize cloud infrastructure autonomously — zero manual implementation, configuration, or ongoing operations.

**Why**: Enterprises spend massive resources on cloud ops teams doing repetitive work — compliance audits, cost reviews, incident response, monitoring. An LLM-driven agent with access to cloud APIs via MCP can do this continuously, consistently, and at scale.

**Core philosophy** (borrowed from OpenClaw): An agent is NOT a state machine or DAG. It's **system prompt + conversation history + tool definitions**. The LLM is the sole decision-maker. OGClaw never decides anything on its own.

**Key architectural decision**: Cloud tools are accessed via **MCP (Model Context Protocol)**. OGClaw is an MCP host/client that connects to MCP servers (AWS, Terraform, etc.) over stdio. This means:
- No AWS SDK code in OGClaw itself
- Cloud capabilities are pluggable MCP servers
- Adding a new cloud = adding a new MCP server config
- Enterprise can control exactly which tools each agent has access to

---

## Tech Stack

| Component | Choice | Why |
|-----------|--------|-----|
| Language | Go 1.22+ | Learning goal, great for enterprise infra |
| LLM SDK | `github.com/anthropics/anthropic-sdk-go` | Official, streaming support |
| MCP SDK | `github.com/modelcontextprotocol/go-sdk` | Official (Anthropic+Google), has client/host mode |
| TUI | `github.com/charmbracelet/bubbletea` | 39.5k stars, Elm architecture, streaming |
| Config | `gopkg.in/yaml.v3` | Agent definitions |
| Session storage | JSONL files (append-only) | Simple, debuggable |
| AWS tools | `awslabs/mcp` servers (Python, stdio) | 69 official AWS MCP servers |
| Terraform | `hashicorp/terraform-mcp-server` (Go, stdio) | Official HashiCorp |
| State dir | `~/.ogclaw/` | Sessions, config, logs |

---

## Milestones

### Milestone 1: Hello Go — Chat with Claude in the Terminal

**Goal**: Get a Go project running. Call Claude API. Print a response.
**Go concepts**: modules, packages, structs, error handling, env vars, `fmt`, `bufio`

```
ogclaw/
├── go.mod
├── go.sum
├── main.go                    # Entry point, reads ANTHROPIC_API_KEY, starts chat loop
└── internal/
    └── llm/
        └── claude.go          # Wraps anthropic-sdk-go: send message, get response (non-streaming)
```

**Implementation details**:
- `main.go`: Read `ANTHROPIC_API_KEY` from environment, create LLM client, run interactive `you> ` prompt loop using `bufio.Scanner`
- `internal/llm/claude.go`: Wrap `anthropic-sdk-go` — `NewClient()`, `SendMessage(ctx, prompt) (string, error)` using `messages.New()` API
- Single-turn only (no history), hardcoded to `claude-sonnet-4-20250514`
- **Verify**: `go run main.go` → type "what is 2+2" → get a response → loop until Ctrl+C

---

### Milestone 2: Conversation Memory + JSONL Persistence

**Goal**: Multi-turn conversations that survive restarts.
**Go concepts**: slices, JSON marshal/unmarshal, file I/O, `encoding/json`, `os.OpenFile`

```
internal/
└── session/
    └── history.go             # Message type, []Message slice, JSONL read/write
```

**Implementation details**:
- `Message` struct: `Role string`, `Content string`, `Timestamp time.Time`
- `History` struct: holds `[]Message` in memory, knows its file path
- `NewHistory(path string) *History` — creates or loads from JSONL file
- `Add(msg Message)` — appends to in-memory slice AND appends one JSON line to file
- `Messages() []Message` — returns slice for building Claude API messages
- Modify `claude.go` to accept `[]Message` history and send full conversation
- Modify `main.go` to create history at `~/.ogclaw/sessions/default.jsonl`, pass to LLM client
- On startup, load existing JSONL file if present
- **Verify**: chat, quit, restart → conversation continues seamlessly

---

### Milestone 3: Streaming + Bubble Tea TUI

**Goal**: Rich terminal UI with streaming responses.
**Go concepts**: goroutines, channels, interfaces, Elm architecture (Model/Update/View)

```
internal/
├── llm/
│   └── claude.go              # Add streaming via Messages.NewStreaming()
└── tui/
    └── chat.go                # Bubble Tea: scrollable history, input field, streaming area
```

**Implementation details**:
- **Streaming in claude.go**: Add `StreamMessage(ctx, history, callback func(chunk string))` that uses the SDK's streaming API. Callback fires for each text delta.
- **TUI model** (`chat.go`):
  - `Model` struct: chat messages, input field (textinput), viewport (scrollable), streaming buffer, spinner
  - `Init()` → return nil (or set focus on input)
  - `Update(msg)` → handle key events (Enter to send, Ctrl+C to quit), streaming chunks (append to buffer), stream-done (finalize message)
  - `View()` → render viewport (history) + streaming area + input field
- **Goroutine bridge**: When user sends message, spawn goroutine that calls `StreamMessage`, sends chunks as Bubble Tea messages via `p.Send()`
- Spinner shows while waiting for first token
- **Verify**: ask a long question, watch text stream in token by token

---

### Milestone 4: MCP Host — Connect to External Tool Servers

**Goal**: OGClaw becomes an MCP host that can spawn and connect to MCP servers.
**Go concepts**: process spawning, `exec.Command`, context, interface design

```
internal/
├── mcp/
│   ├── host.go                # MCP host: spawn servers, manage connections
│   ├── registry.go            # Discovers tools from connected MCP servers
│   └── transport.go           # Stdio transport wrapper for spawned servers
└── agent/
    └── loop.go                # Core agent loop: LLM → tool_use → MCP execute → loop
```

**Implementation details**:
- **host.go**: `MCPHost` struct manages multiple MCP server connections
  - `AddServer(name, command string, args []string, env map[string]string)` — spawns process via `exec.Command`, connects stdin/stdout
  - `Shutdown()` — gracefully terminates all spawned processes
  - Uses `github.com/modelcontextprotocol/go-sdk` client mode
- **registry.go**: `ToolRegistry` calls `tools/list` on each connected server
  - Collects all available tools with their JSON schemas
  - `GetTools() []ToolDefinition` — returns all tools for Claude API
  - `FindServer(toolName string) *MCPServer` — routes tool calls to correct server
- **transport.go**: Wraps stdio transport for spawned processes
  - Handles JSON-RPC message framing over stdin/stdout
- **loop.go**: Core agent execution loop
  - Send messages + tool definitions to Claude
  - If Claude returns `tool_use` blocks, execute via MCP, collect results
  - Send tool results back to Claude, loop until text-only response
- **Config** (`config.yaml`):
  ```yaml
  mcp_servers:
    - name: filesystem
      command: npx
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
  ```
- Start with filesystem MCP server to prove the plumbing
- **Verify**: ask Claude "list files in /tmp" → it calls filesystem MCP server → shows files

---

### Milestone 5: Agent Config + System Prompt Assembly

**Goal**: Configurable agents with personality, model selection, tool policies.
**Go concepts**: YAML parsing, struct tags, file reading, string building

```
ogclaw/
├── config.yaml                # Agent definitions
├── workspace/
│   ├── SOUL.md                # Agent personality
│   └── IDENTITY.md            # Agent identity
└── internal/
    ├── agent/
    │   └── config.go          # Parse config, resolve agent settings
    └── prompt/
        └── builder.go         # Assemble system prompt from markdown files + sections
```

**Implementation details**:
- **config.go**: `AgentConfig` struct with YAML tags
  ```go
  type AgentConfig struct {
      ID        string            `yaml:"id"`
      Name      string            `yaml:"name"`
      Model     string            `yaml:"model"`
      Workspace string            `yaml:"workspace"`
      MCPServers []MCPServerConfig `yaml:"mcp_servers"`
      Tools     ToolPolicy        `yaml:"tools"`
  }
  ```
  - `LoadConfig(path string) (*Config, error)` — parses YAML, resolves defaults
  - `ResolveAgent(id string) (*AgentConfig, error)` — merges defaults + agent-specific
- **builder.go**: `BuildSystemPrompt(agent *AgentConfig) (string, error)`
  - Reads workspace markdown files (SOUL.md, IDENTITY.md)
  - Appends tool listing section
  - Appends runtime info (model, OS, time)
  - Returns assembled prompt string
- Config defines agents: id, model, workspace, MCP servers, tool policies
- **Verify**: edit SOUL.md to say "respond like a pirate" → agent personality changes

---

### Milestone 6: AWS Compliance Agent

**Goal**: First real enterprise use case — connect to AWS MCP servers, run compliance audit.
**Go concepts**: concurrent MCP connections, structured output handling

**MCP servers added to config**:
```yaml
mcp_servers:
  - name: aws-iam
    command: uv
    args: ["tool", "run", "awslabs.iam-mcp-server"]
    env:
      AWS_PROFILE: default
      AWS_REGION: us-east-1

  - name: aws-security
    command: uv
    args: ["tool", "run", "awslabs.well-architected-security-mcp-server"]
    env:
      AWS_PROFILE: default

  - name: aws-cloudwatch
    command: uv
    args: ["tool", "run", "awslabs.cloudwatch-mcp-server"]
    env:
      AWS_PROFILE: default
```

**Implementation details**:
- **Workspace file** (`SOUL.md`): Compliance audit instructions
  - You are a cloud security compliance agent
  - Run checks against CIS AWS Foundations Benchmark
  - Report findings with severity, affected resources, and remediation steps
  - Never make changes without explicit approval
- Agent connects to AWS MCP servers on startup
- User says "audit my AWS account for compliance issues"
- Agent calls IAM tools (list users, check MFA, review policies), Security Hub, CloudWatch
- Reports findings in structured format
- **Verify**: run against a real (or test) AWS account, get compliance findings

---

### Milestone 7: Multi-Agent — Sub-Agent Spawning

**Goal**: Agents can spawn child agents for parallel work.
**Go concepts**: goroutines, sync.WaitGroup, context.Context, cancellation

```
internal/
├── agent/
│   ├── spawn.go               # Sub-agent spawning
│   └── registry.go            # Agent registry: id → config
└── tools/
    └── subagent.go            # "spawn_agent" tool for LLM
```

**Implementation details**:
- **registry.go**: `AgentRegistry` maps agent IDs to configs
  - `Register(config AgentConfig)` — adds agent to registry
  - `Get(id string) (*AgentConfig, error)` — retrieves agent config
- **spawn.go**: `SpawnAgent(ctx, parentID, childID, task string) (string, error)`
  - Creates new agent instance from registry
  - Runs in separate goroutine with own history + MCP connections
  - Returns result string when child completes
- **subagent.go**: Defines `spawn_agent` tool for Claude API
  - Tool schema: `{agent_id: string, task: string}`
  - Executes SpawnAgent, returns result to parent
- Limits: max depth=1, max concurrent=5
- **Verify**: "audit compliance AND analyze costs" → spawns 2 agents in parallel

---

### Milestone 8: Cost + Monitoring Agents

**Goal**: Add cost optimization and monitoring use cases.

**Additional MCP servers**:
```yaml
  - name: aws-cost
    command: uv
    args: ["tool", "run", "awslabs.cost-explorer-mcp-server"]

  - name: aws-cloudtrail
    command: uv
    args: ["tool", "run", "awslabs.cloudtrail-mcp-server"]
```

**Implementation details**:
- **Cost agent** (workspace/cost/SOUL.md):
  - Analyzes spend patterns using Cost Explorer
  - Identifies waste (idle resources, oversized instances)
  - Suggests optimizations with estimated savings
- **Monitoring agent** (workspace/monitoring/SOUL.md):
  - Watches CloudWatch alarms and metrics
  - Analyzes CloudTrail logs for anomalies
  - Detects unusual API activity patterns
- **Verify**: "what am I spending the most on?" → detailed cost breakdown with recommendations

---

## Implementation Order

```
M1 (Hello Go) → M2 (History) → M3 (TUI) → M4 (MCP Host) → M5 (Config) → M6 (AWS Compliance) → M7 (Multi-Agent) → M8 (Cost+Monitoring)
```

Each milestone produces a runnable `go run main.go`. We don't move on until the current one works end-to-end.

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                    OGClaw (Go binary)                    │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │ Bubble   │  │ Agent    │  │ MCP Host             │  │
│  │ Tea TUI  │──│ Loop     │──│                      │  │
│  │          │  │          │  │  ┌─ AWS IAM Server   │  │
│  │ Input    │  │ History  │  │  ├─ AWS CW Server    │  │
│  │ Stream   │  │ Prompt   │  │  ├─ AWS Cost Server  │  │
│  │ Display  │  │ Tools    │  │  ├─ Terraform Server │  │
│  └──────────┘  └────┬─────┘  │  └─ ... (pluggable)  │  │
│                     │        └──────────────────────┘  │
│                     │                                   │
│              ┌──────┴──────┐                            │
│              │ Claude API  │                            │
│              │ (streaming) │                            │
│              └─────────────┘                            │
└─────────────────────────────────────────────────────────┘
```

---

## What We're NOT Building (Yet)

- Web UI / HTTP API (CLI-first)
- Multi-cloud (AWS first, Azure/GCP later via additional MCP servers)
- Custom MCP servers (use official awslabs ones first)
- Automated remediation (read-only first, write operations need approval gates)
- User auth / RBAC (single-user self-hosted first)
- Compaction / context overflow handling (later optimization)
