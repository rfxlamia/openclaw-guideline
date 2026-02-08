# Multi-Agent Architecture Reference

OpenClaw's multi-agent routing, sub-agents, and orchestration patterns.

---

## Core Concepts

### Agent Components

Each agent has isolated:
- **Workspace**: `SOUL.md`, `AGENTS.md`, `USER.md`, `skills/`
- **State Directory**: `agentDir` (auth profiles, model config)
- **Session Store**: Chat history, routing state

**Principles**: Full isolation by default; no shared credentials; customizable tool policies per-agent.

### Agents vs Sub-Agents

| Aspect | Multi-Agent | Sub-Agent |
|--------|-------------|-----------|
| **Purpose** | Independent personas/workspaces | Parallel task execution |
| **Persistence** | Permanent | Temporary, task-scoped |
| **Session Key** | `agent:<id>:<mainKey>` | `agent:<id>:subagent:<uuid>` |
| **Workspace** | Separate per agent | Shares parent workspace |

---

## Agent Isolation Model

### File System

```
~/.openclaw/
├── openclaw.json              # Global config
├── workspace-<agentId>/       # Per-agent workspace
└── agents/<agentId>/
    ├── agent/
    │   ├── auth-profiles.json  # Per-agent credentials
    │   └── model-registry.json # Per-agent model config
    └── sessions/<sessionId>.jsonl
```

**Auth profiles are per-agent and NOT shared.** To share credentials:
```bash
cp ~/.openclaw/agents/home/agent/auth-profiles.json \
   ~/.openclaw/agents/work/agent/auth-profiles.json
```

### Workspace Files

- `SOUL.md` - Agent identity
- `AGENTS.md` - Operational instructions
- `USER.md` - Handler preferences
- `MEMORY.md` - Long-term knowledge
- `TOOLS.md` - External tool docs
- `skills/` - Agent-specific skills

**Shared Skills**: Global at `~/.openclaw/skills/` (symlinked to all agents).

---

## Multi-Agent Routing

### Routing Flow

```
Inbound Message → Gateway → Binding Resolution → Target Agent → Lane Queue → Agent Runner
```

Routing is **deterministic** (most-specific wins):
1. Peer match (exact DM/group/channel ID)
2. Guild ID (Discord)
3. Team ID (Slack)
4. Account ID (WhatsApp "personal" vs "biz")
5. Channel-level match (`accountId: "*"`)
6. Fallback (default agent)

### Configuration Examples

**Two WhatsApp Accounts → Two Agents:**
```javascript
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home",
        agentDir: "~/.openclaw/agents/home/agent" },
      { id: "work", workspace: "~/.openclaw/workspace-work",
        agentDir: "~/.openclaw/agents/work/agent" }
    ]
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
    // Override: specific group to work agent
    { agentId: "work", match: { channel: "whatsapp", accountId: "personal",
        peer: { kind: "group", id: "[email protected]" } } }
  ],
  channels: { whatsapp: { accounts: { personal: {}, biz: {} } } }
}
```

**Channel-Based Model Split:**
```javascript
{
  agents: {
    list: [
      { id: "fast", model: "anthropic/claude-sonnet-4-5" },
      { id: "deep", model: "anthropic/claude-opus-4-6" }
    ]
  },
  bindings: [
    { agentId: "fast", match: { channel: "whatsapp" } },
    { agentId: "deep", match: { channel: "telegram" } }
  ]
}
```

**DM-Level Routing (Multi-User):**
```javascript
{
  bindings: [
    { agentId: "alex", match: { channel: "whatsapp", peer: { kind: "dm", id: "+15551230001" } } },
    { agentId: "mia", match: { channel: "whatsapp", peer: { kind: "dm", id: "+15551230002" } } }
  ],
  channels: { whatsapp: { dmPolicy: "allowlist", allowFrom: ["+15551230001", "+15551230002"] } }
}
```

**Note**: DM access control is global per WhatsApp account; replies come from the same number.

### CLI Helpers

```bash
openclaw agents add work          # Add new agent
openclaw agents list --bindings   # List with bindings
openclaw status                   # In-chat: check handler
```

---

## Sub-Agents for Parallel Execution

Sub-agents are background runs spawned for parallel tasks.

**Characteristics:**
- Session key: `agent:<agentId>:subagent:<uuid>`
- Isolated from parent chat history
- Non-blocking, runs in background
- Announces results when complete
- **Cannot spawn nested sub-agents**

### Tool: `sessions_spawn`

```javascript
sessions_spawn({
  task: "Research top 5 competitors",
  label: "competitor-research",
  agentId: "research",               // Optional: target agent
  model: "anthropic/claude-sonnet-4-5",
  runTimeoutSeconds: 300,
  cleanup: true
})
// Returns: { status: "accepted", runId: "...", childSessionKey: "..." }
```

### Managing Sub-Agents

```bash
/subagents              # List active
/subagents info <id>    # Detailed info
/subagents send <id>    # Send message
/subagents stop <id>    # Stop sub-agent
/stop                   # Stop all from current session
```

### Configuration

```javascript
{
  agents: {
    defaults: {
      subagents: {
        model: "anthropic/claude-sonnet-4-5",
        thinking: "low",
        maxConcurrent: 5,
        allowAgents: ["*"]  // or ["research", "writer"]
      }
    },
    list: [{
      id: "main",
      subagents: {
        model: "openrouter/deepseek/deepseek-r1",
        allowAgents: ["research"]
      }
    }]
  }
}
```

### Tool Policy

**Allowed**: `read`, `write`, `edit`, `exec`, `bash`, `browser`, `web_search`, `cron`, `message`

**Denied**: `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status`

Override:
```javascript
{ tools: { subagents: { allow: ["sessions_list"], deny: ["browser"] } } }
```

---

## Session Management

### Session Key Format

```
agent:<agentId>:<contextType>:<identifier>[:topic:<topicId>]

Examples:
- agent:main:main                          (direct DM)
- agent:work:telegram:group:-1001234567890 (Telegram group)
- agent:home:subagent:4215c864             (sub-agent)
- agent:main:telegram:group:123:topic:456  (topic thread)
```

**Resolution:** Direct chats collapse to `agent:<id>:main`; groups include channel/chatType/chatID.

### Session Store

Location: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

Format (JSONL, one event per line):
```jsonl
{"role":"user","content":"Hello","timestamp":"2026-02-08T10:00:00Z"}
{"role":"assistant","content":"Hi!","timestamp":"2026-02-08T10:00:05Z"}
```

### Session Tools

```javascript
sessions_list({ kinds: ["main", "group"], limit: 20, activeMinutes: 60 })
sessions_history({ sessionKey: "agent:main:main", limit: 50, includeTools: true })
sessions_send({ sessionKey: "agent:work:main", message: "Status?", timeoutSeconds: 30 })
session_status({ sessionKey: "agent:main:main", model: "..." })
```

### Agent-to-Agent Messaging

```javascript
{ tools: { agentToAgent: { enabled: true, allow: ["home", "work"], maxPingPongTurns: 3 } } }
```

Reply `REPLY_SKIP` to stop ping-pong; reply `ANNOUNCE_SKIP` to suppress announcement.

### Reset Triggers

| Method | Command |
|--------|---------|
| Manual | `/new [model]`, `/reset` |
| Auto | Daily at 4:00 AM; idle expiry (configurable) |
| Manual file | Delete `.jsonl` transcript |

---

## Tool Policies & Sandbox Configuration

### Per-Agent Tool Policies

```javascript
{
  agents: {
    list: [{
      id: "public",
      tools: {
        allow: ["read", "sessions_list", "message"],
        deny: ["write", "edit", "exec", "browser"]
      }
    }]
  }
}
```

**Tool Groups**: `group:fs`, `group:runtime`, `group:sessions`, `group:memory`, `group:ui`, `group:automation`, `group:messaging`, `group:nodes`, `group:openclaw`

### Sandbox Configuration

```javascript
{
  agents: {
    list: [{
      id: "public",
      sandbox: {
        mode: "all",           // "off" | "all"
        scope: "agent",        // "agent" | "session" | "shared"
        workspaceAccess: "ro", // "none" | "ro" | "rw"
        docker: {
          image: "openclaw/agent:latest",
          setupCommand: "apt-get update && apt-get install -y git",
          binds: ["~/.ssh:/root/.ssh:ro", "/data/shared:/shared:rw"]
        }
      }
    }]
  }
}
```

**Critical**: `setupCommand` runs once on container creation; binds with `/var/run/docker.sock` give host control.

### Elevated Mode

Allows sandboxed agents to run `exec` on host:

```javascript
{ tools: { elevated: { enabled: true, allowFrom: { whatsapp: ["+15551234567"] } } } }
```

In-chat: `/elevated on`, `/elevated full` (skip approvals), `/elevated off`

---

## Real-World Patterns

### 1. Principal-Specialist Hierarchy

```javascript
{
  agents: {
    list: [
      { id: "principal", model: "anthropic/claude-opus-4-6",
        subagents: { allowAgents: ["blog", "app-dev"] } },
      { id: "blog", model: "anthropic/claude-sonnet-4-5" },
      { id: "app-dev", model: "anthropic/claude-opus-4-6" }
    ]
  },
  bindings: [{ agentId: "principal", match: { channel: "telegram" } }]
}
```

Principal delegates to specialists via `sessions_spawn`.

### 2. Research Fan-Out

```javascript
for (const company of ["A", "B", "C"]) {
  sessions_spawn({
    task: `Research ${company}`,
    model: "openrouter/deepseek/deepseek-r1"
  });
}
```

### 3. Family/Public Agent (Restricted)

```javascript
{
  agents: {
    list: [{
      id: "family",
      groupChat: { mentionPatterns: ["@family"] },
      sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
      tools: { allow: ["read", "web_search"], deny: ["write", "exec"] }
    }]
  }
}
```

### 4. Multi-Tenant Gateway

```javascript
{
  agents: {
    list: [
      { id: "user1", workspace: "~/.openclaw/workspace-user1",
        agentDir: "~/.openclaw/agents/user1/agent" },
      { id: "user2", workspace: "~/.openclaw/workspace-user2",
        agentDir: "~/.openclaw/agents/user2/agent" }
    ]
  },
  bindings: [
    { agentId: "user1", match: { channel: "whatsapp", peer: { kind: "dm", id: "+1555USER1" } } },
    { agentId: "user2", match: { channel: "whatsapp", peer: { kind: "dm", id: "+1555USER2" } } }
  ]
}
```

---

## Cost Optimization

### Multi-Model Routing

```javascript
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-6",
      heartbeat: { model: "google/gemini-flash-lite-2", enabled: true },
      subagents: { model: "openrouter/deepseek/deepseek-r1", thinking: "low" }
    },
    list: [{
      id: "main",
      model: "anthropic/claude-opus-4-6",
      fallback: ["openai/gpt-5.2", "anthropic/claude-sonnet-4-5"]
    }]
  }
}
```

**Impact**: $2,750/month (all Opus) → $1,000/month (64% reduction).

### Concurrency Limits

```javascript
{ agents: { defaults: { subagents: { maxConcurrent: 5 } } } }
```

Prevents token cost explosions and rate limit violations.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Session key collisions (shared history) | Reused `agentDir` | Use unique `agentDir` per agent |
| Sub-agent "(no output)" | Announce step failed | Check transcript: `~/.openclaw/agents/main/sessions/<id>.jsonl` |
| Discord ignores group messages | Message Content Intent disabled | Enable in Discord Developer Portal |
| Telegram topics = orphaned sessions | By design (topic isolation) | Use single group chat for continuity |
| Can't access other agent's credentials | Per-agent auth by design | Copy `auth-profiles.json` or use `openclaw auth set` |
| Sub-agent doesn't announce | Gateway restart or `ANNOUNCE_SKIP` | Check transcript for announce step |
| Wrong agent handles message | Binding order wrong | Most-specific first; use `/status` to debug |
| Sandbox fails | Docker issues | `openclaw sandbox explain`; `docker ps -a \| grep openclaw` |

### Diagnostic Commands

```bash
openclaw agents list --bindings
openclaw sandbox explain [--agent <id>]
openclaw doctor [--deep]
openclaw logs --follow
```

---

**Version:** 1.0 (2026-02-08) | **Compatibility:** OpenClaw v2026.1.6+
