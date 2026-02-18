# OGClaw

Autonomous enterprise cloud operations platform. Go-based multi-agent system that manages, operates, and optimizes cloud infrastructure via LLM-driven agents and MCP (Model Context Protocol).

## What

OGClaw connects Claude to your cloud infrastructure through MCP servers. No AWS SDK code in OGClaw itself — cloud capabilities are pluggable MCP servers (AWS IAM, CloudWatch, Cost Explorer, Terraform, etc.). The LLM is the sole decision-maker; OGClaw never decides anything on its own.

An agent = **system prompt + conversation history + tool definitions**.

## Architecture

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

## Tech Stack

| Component | Choice |
|-----------|--------|
| Language | Go 1.22+ |
| LLM SDK | `anthropic-sdk-go` |
| MCP SDK | `modelcontextprotocol/go-sdk` |
| TUI | `charmbracelet/bubbletea` |
| Config | YAML |
| Sessions | JSONL (append-only) |
| Cloud tools | AWS MCP servers (stdio) |

## Use Cases

- **Compliance auditing** — CIS benchmarks, IAM policy review, MFA checks
- **Cost optimization** — spend analysis, waste identification, right-sizing
- **Monitoring** — CloudWatch alarms, CloudTrail anomaly detection
- **Multi-agent** — parallel sub-agent spawning for concurrent operations

## Getting Started

```bash
export ANTHROPIC_API_KEY=sk-...
go run main.go
```

## Status

Under active development. See [docs/PLAN.md](docs/PLAN.md) for the full implementation roadmap.

## License

MIT
