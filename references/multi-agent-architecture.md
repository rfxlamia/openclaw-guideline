# Multi-Agent Architecture Reference

**A comprehensive guide to OpenClaw's multi-agent routing, sub-agents, and orchestration patterns for production deployments.**

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Agent Isolation Model](#agent-isolation-model)
3. [Multi-Agent Routing](#multi-agent-routing)
4. [Sub-Agents for Parallel Execution](#sub-agents-for-parallel-execution)
5. [Session Management](#session-management)
6. [Tool Policies & Sandbox Configuration](#tool-policies--sandbox-configuration)
7. [Real-World Patterns](#real-world-patterns)
8. [Cost Optimization](#cost-optimization)
9. [Troubleshooting](#troubleshooting)

---

## Core Concepts

### What is "One Agent"?

An **agent** in OpenClaw is a fully scoped "brain" with its own:

```
Agent Components:
├── Workspace (SOUL.md, AGENTS.md, USER.md, skills/)
├── State Directory (agentDir: auth profiles, model config)
└── Session Store (chat history, routing state)
```

**Key Principles:**
- Each agent is **fully isolated** by default
- Agents do NOT share credentials unless explicitly configured
- Session keys determine conversation isolation
- Tool policies can be customized per-agent

### Critical Distinction: Agents vs Sub-Agents

| Aspect | Multi-Agent | Sub-Agent |
|--------|-------------|-----------|
| **Purpose** | Multiple independent personas/workspaces | Parallel task execution within one agent |
| **Persistence** | Permanent, survives restarts | Temporary, task-scoped |
| **Session Key** | `agent:<agentId>:<mainKey>` | `agent:<agentId>:subagent:<uuid>` |
| **Workspace** | Separate workspace per agent | Shares parent agent workspace |
| **Use Case** | Different users/contexts | Background work, research fan-out |

---

## Agent Isolation Model

### File System Paths

```
~/.openclaw/
├── openclaw.json              # Global configuration
├── workspace/                 # Default workspace (single-agent mode)
├── workspace-<agentId>/       # Per-agent workspaces (multi-agent)
└── agents/
    └── <agentId>/
        ├── agent/
        │   ├── auth-profiles.json  # Per-agent credentials
        │   └── model-registry.json # Per-agent model config
        └── sessions/
            └── <sessionId>.jsonl   # JSONL transcripts
```

### Authentication Isolation

**CRITICAL:** Auth profiles are **per-agent** and **NOT shared by default**.

```json
// Agent "home" credentials
~/.openclaw/agents/home/agent/auth-profiles.json

// Agent "work" credentials  
~/.openclaw/agents/work/agent/auth-profiles.json
```

**To share credentials across agents:**
```bash
# Manual copy (use with caution)
cp ~/.openclaw/agents/main/agent/auth-profiles.json \
   ~/.openclaw/agents/work/agent/auth-profiles.json
```

### Workspace Isolation

```javascript
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        // Personal SOUL.md, custom skills, etc.
      },
      {
        id: "work", 
        workspace: "~/.openclaw/workspace-work",
        // Work-specific persona, tools, guidelines
      }
    ]
  }
}
```

**Workspace Files (per agent):**
- `SOUL.md` - Agent identity and constraints
- `AGENTS.md` - Operational instructions
- `USER.md` - Handler preferences  
- `MEMORY.md` - Long-term curated knowledge
- `TOOLS.md` - External tool documentation
- `skills/` - Agent-specific skills

**Shared Skills:**
- Global skills: `~/.openclaw/skills/` (symlinked to all agents)
- Per-agent skills: `~/.openclaw/workspace-<agentId>/skills/`

---

## Multi-Agent Routing

### Routing Architecture

```
Inbound Message
    ↓
Gateway (WebSocket Control Plane)
    ↓
Binding Resolution (deterministic, most-specific wins)
    ↓
Target Agent (isolated workspace + session)
    ↓
Lane Queue (serial execution by default)
    ↓
Agent Runner (model selection, tool execution)
```

### Binding Rules (Priority Order)

Routing is **deterministic** and **most-specific wins**:

1. **Peer match** (exact DM/group/channel ID)
2. **Guild ID** (Discord)
3. **Team ID** (Slack)  
4. **Account ID** (e.g., WhatsApp account "personal" vs "biz")
5. **Channel-level match** (`accountId: "*"`)
6. **Fallback** (default agent or first in list)

### Configuration Examples

#### Example 1: Two WhatsApp Accounts → Two Agents

```javascript
{
  agents: {
    list: [
      {
        id: "home",
        default: true,
        workspace: "~/.openclaw/workspace-home",
        agentDir: "~/.openclaw/agents/home/agent",
      },
      {
        id: "work",
        workspace: "~/.openclaw/workspace-work", 
        agentDir: "~/.openclaw/agents/work/agent",
      }
    ]
  },
  
  bindings: [
    // Personal WhatsApp → home agent
    { 
      agentId: "home", 
      match: { channel: "whatsapp", accountId: "personal" } 
    },
    
    // Work WhatsApp → work agent
    { 
      agentId: "work", 
      match: { channel: "whatsapp", accountId: "biz" } 
    },
    
    // Override: specific group to work agent (even from personal account)
    {
      agentId: "work",
      match: {
        channel: "whatsapp",
        accountId: "personal", 
        peer: { kind: "group", id: "[email protected]" }
      }
    }
  ],
  
  channels: {
    whatsapp: {
      accounts: {
        personal: {},
        biz: {}
      }
    }
  }
}
```

#### Example 2: Split by Channel (WhatsApp → Fast, Telegram → Deep Work)

```javascript
{
  agents: {
    list: [
      {
        id: "chat",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-5",
      },
      {
        id: "opus",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-6",
      }
    ]
  },
  
  bindings: [
    { agentId: "chat", match: { channel: "whatsapp" } },
    { agentId: "opus", match: { channel: "telegram" } }
  ]
}
```

#### Example 3: DM Split on Same WhatsApp (Multiple Users)

```javascript
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" }
    ]
  },
  
  bindings: [
    { 
      agentId: "alex", 
      match: { 
        channel: "whatsapp", 
        peer: { kind: "dm", id: "+15551230001" } 
      } 
    },
    { 
      agentId: "mia", 
      match: { 
        channel: "whatsapp", 
        peer: { kind: "dm", id: "+15551230002" } 
      } 
    }
  ],
  
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"]
    }
  }
}
```

**Important Notes:**
- DM access control is **global per WhatsApp account**, not per-agent
- Replies come from the **same WhatsApp number** (no per-agent sender identity)
- Direct chats collapse to agent's **main session key** (true isolation requires one agent per person)

### Agent CLI Helper

```bash
# Add new agent (wizard-guided)
openclaw agents add work

# List all agents with bindings
openclaw agents list --bindings

# Check which agent handles a message
openclaw status  # in-chat command
```

---

## Sub-Agents for Parallel Execution

### What Are Sub-Agents?

**Sub-agents** are background agent runs spawned from a parent agent for **parallel task execution**.

```
Main Agent Session
    ├─ Sub-agent 1 (research competitor A)
    ├─ Sub-agent 2 (research competitor B)  
    └─ Sub-agent 3 (draft summary)
         ↓
    All complete independently
         ↓
    Announce results back to main chat
```

**Key Characteristics:**
- **Session key:** `agent:<agentId>:subagent:<uuid>`
- **Isolation:** Cannot see main chat history
- **Non-blocking:** Returns immediately, runs in background
- **Result delivery:** Announces back to requester chat when complete
- **No nesting:** Sub-agents **cannot spawn sub-agents**

### Tool: `sessions_spawn`

```javascript
// Spawn a sub-agent
sessions_spawn({
  task: "Research top 5 competitors in AI agent space",
  label: "competitor-research",      // Optional: human-readable label
  agentId: "research",               // Optional: target different agent
  model: "anthropic/claude-sonnet-4-5",  // Optional: override model
  runTimeoutSeconds: 300,            // Optional: 5-minute timeout
  cleanup: true                      // Optional: auto-delete transcript
})

// Returns immediately:
{
  status: "accepted",
  runId: "4215c864-27ef-41bd-bed2-e909ac7d34c1",
  childSessionKey: "agent:main:subagent:4215c864..."
}
```

### Sub-Agent Workflow

```
1. Main agent calls sessions_spawn
   ↓
2. Gateway creates isolated session
   ↓
3. Sub-agent runs task (parallel to main)
   ↓
4. Sub-agent completes → generates summary
   ↓
5. Announce step runs (can be suppressed with ANNOUNCE_SKIP)
   ↓
6. Result posted to requester chat
```

**Announce Message Format:**
```
✓ Sub-agent completed: competitor-research

Status: success
Result: Found 5 competitors with detailed analysis...

[Stats: 2m 15s • 12,450 tokens • $0.15 • sessionId: 4215c864...]
```

### Managing Sub-Agents

Use `/subagents` command in chat:

```bash
# List active sub-agents for current session
/subagents

# Get detailed info
/subagents info <id|#>

# Send message to running sub-agent
/subagents send <id|#> "Focus on pricing comparison"

# Stop sub-agent
/subagents stop <id|#>

# Stop all sub-agents from current session
/stop
```

### Sub-Agent Configuration

```javascript
{
  agents: {
    defaults: {
      subagents: {
        // Default model for sub-agents (cheaper than main)
        model: "anthropic/claude-sonnet-4-5",
        
        // Default thinking level
        thinking: "low",
        
        // Max concurrent sub-agents
        maxConcurrent: 5,
        
        // Which agents can be targeted via agentId
        allowAgents: ["*"]  // or ["research", "writer"]
      }
    },
    
    list: [
      {
        id: "main",
        model: "anthropic/claude-opus-4-6",
        
        // Per-agent sub-agent config (overrides defaults)
        subagents: {
          model: "openrouter/deepseek/deepseek-r1",
          allowAgents: ["research", "writer"]
        }
      }
    ]
  }
}
```

### Sub-Agent Tool Policy

**By default, sub-agents get all tools EXCEPT session tools:**

```
Allowed by default:
✓ read, write, edit, apply_patch
✓ exec, bash, process  
✓ browser, web_search, web_fetch
✓ cron, gateway
✓ message (channel messaging)

Denied by default:
✗ sessions_list
✗ sessions_history  
✗ sessions_send
✗ sessions_spawn (no nested sub-agents)
✗ session_status
```

**Why?** To prevent:
- Nested fan-out explosions
- Cross-session leakage
- Runaway token costs

**Override if needed:**
```javascript
{
  tools: {
    subagents: {
      allow: ["sessions_list"],  // Allow specific session tools
      deny: ["browser"]          // Block tools for sub-agents
    }
  }
}
```

---

## Session Management

### Session Key Architecture

**Session keys** determine conversation isolation:

```
Format: agent:<agentId>:<contextType>:<identifier>[:topic:<topicId>]

Examples:
- agent:main:main                          (direct DM, main agent)
- agent:work:telegram:group:-1001234567890 (Telegram group)
- agent:home:subagent:4215c864             (sub-agent)
- agent:main:telegram:group:123:topic:456  (Telegram topic thread)
```

**Session Key Resolution:**
1. Direct chats → collapse to `agent:<agentId>:main` ("main session")
2. Group chats → include channel, chatType, chatID
3. Sub-agents → unique UUID, isolated from parent
4. Topics/threads → additional topic ID when supported

### Session Store

```
~/.openclaw/agents/<agentId>/sessions/
└── <sessionId>.jsonl    # JSONL transcript (one event per line)
```

**JSONL Format:**
```jsonl
{"role":"user","content":"Hello","timestamp":"2026-02-08T10:00:00Z"}
{"role":"assistant","content":"Hi! How can I help?","timestamp":"2026-02-08T10:00:05Z"}
{"role":"tool","name":"read","args":{"path":"file.txt"},"timestamp":"2026-02-08T10:00:10Z"}
```

**Why JSONL?**
- Line-by-line audit trail
- Greppable for debugging
- No proprietary format
- Direct inspection: `cat ~/.openclaw/agents/main/sessions/<sessionId>.jsonl`

### Session Tools

```javascript
// List active sessions
sessions_list({
  kinds: ["main", "group"],    // Filter by session type
  limit: 20,                   // Max results
  activeMinutes: 60,           // Only sessions active in last hour
  messageLimit: 5              // Last N messages per session (0 = none)
})

// Get session history
sessions_history({
  sessionKey: "agent:main:main",  // or sessionId
  limit: 50,                      // Last 50 messages
  includeTools: true              // Include tool call details
})

// Send message to another session (ping-pong)
sessions_send({
  sessionKey: "agent:work:main",
  message: "Status update on project X?",
  timeoutSeconds: 30              // Wait for completion (0 = fire-and-forget)
})

// Check session status
session_status({
  sessionKey: "agent:main:main",  // Optional (defaults to current)
  model: "anthropic/claude-sonnet-4-5"  // Optional: override model
})
```

**Agent-to-Agent Messaging:**

```javascript
{
  tools: {
    agentToAgent: {
      enabled: true,                    // Must explicitly enable
      allow: ["home", "work"],          // Allowlist target agents
      maxPingPongTurns: 3               // Prevent infinite loops (0-5)
    }
  }
}
```

**Reply Protocol:**
- Agent receives `sessions_send` message
- Can reply multiple times (up to `maxPingPongTurns`)
- Reply `REPLY_SKIP` to stop ping-pong
- After ping-pong, target runs announce step
- Reply `ANNOUNCE_SKIP` to suppress announcement

### Session Reset Triggers

```
Reset Commands:
- /new [model]  → Fresh sessionId, optional model override
- /reset        → Fresh sessionId, same model

Auto-Reset:
- Daily reset: 4:00 AM local time (configurable)
- Idle expiry: After N minutes of inactivity (session.reset.idleMinutes)
- Whichever expires first wins

Manual:
- Delete .jsonl transcript file
- Next message recreates session
```

---

## Tool Policies & Sandbox Configuration

### Per-Agent Tool Policies

```javascript
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        
        // Restrictive tool policy for public agent
        tools: {
          allow: [
            "read",                  // File reading only
            "sessions_list",
            "sessions_history",
            "message"                // Can send messages
          ],
          deny: [
            "write", "edit", "apply_patch",  // No file modifications
            "exec", "bash", "process",       // No shell access
            "browser", "canvas"              // No UI tools
          ]
        }
      },
      
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        
        // Full access for personal agent
        // (no tool restrictions)
      }
    ]
  }
}
```

**Tool Groups:**

```javascript
{
  tools: {
    allow: [
      "group:fs",        // read, write, edit, apply_patch
      "group:runtime",   // exec, bash, process
      "group:sessions",  // sessions_*, session_status
      "group:memory",    // memory_search, memory_get
      "group:ui",        // browser, canvas
      "group:automation",// cron, gateway
      "group:messaging", // message
      "group:nodes",     // nodes (companion apps)
      "group:openclaw"   // All built-in tools (excludes plugins)
    ]
  }
}
```

### Per-Agent Sandbox Configuration

```javascript
{
  agents: {
    list: [
      {
        id: "personal",
        sandbox: {
          mode: "off"  // No sandbox (runs on host)
        }
      },
      
      {
        id: "public",
        sandbox: {
          mode: "all",           // Always sandboxed
          scope: "agent",        // One container per agent (vs "session" or "shared")
          workspaceAccess: "ro", // Read-only workspace access (vs "rw" or "none")
          
          docker: {
            image: "openclaw/agent:latest",
            
            // One-time setup after container creation
            setupCommand: "apt-get update && apt-get install -y git curl",
            
            // Mount additional volumes
            binds: [
              "~/.ssh:/root/.ssh:ro",        // SSH keys (read-only)
              "/data/shared:/shared:rw"      // Shared data (read-write)
            ]
          }
        },
        
        tools: {
          // Sandbox defaults: allow safe tools, deny risky ones
          allow: ["read", "sessions_list"],
          deny: ["exec", "write", "browser"]
        }
      }
    ]
  }
}
```

**Sandbox Scope:**

| Scope | Behavior | Use Case |
|-------|----------|----------|
| `agent` | One container per agent | Agent isolation |
| `session` | One container per session | Maximum isolation |
| `shared` | One container for all | Resource optimization |

**Workspace Access:**

| Mode | Behavior |
|------|----------|
| `none` | No workspace access (fully isolated) |
| `ro` | Read-only workspace (safe for public agents) |
| `rw` | Read-write workspace (full access) |

**CRITICAL:** 
- `setupCommand` runs **once** on container creation
- Binds with `/var/run/docker.sock` give host control (dangerous!)
- `scope: "shared"` ignores per-agent binds

### Elevated Mode (Escape Hatch)

**Elevated** allows sandboxed agents to run `exec` on **host** (not in Docker):

```javascript
{
  tools: {
    elevated: {
      enabled: true,
      
      // Sender allowlist (per-provider)
      allowFrom: {
        whatsapp: ["+15551234567"],
        telegram: ["@yourhandle"]
      }
    }
  },
  
  agents: {
    list: [
      {
        id: "public",
        tools: {
          elevated: {
            allowFrom: {
              telegram: ["@admin"]  // Per-agent override
            }
          }
        }
      }
    ]
  }
}
```

**In-Chat Usage:**
```
/elevated on       → Enable for session (host exec, approvals may apply)
/elevated full     → Enable + skip approvals (dangerous!)
/elevated off      → Disable for session
```

**IMPORTANT:**
- Elevated is **exec-only** (doesn't grant extra tools)
- If already running on host, elevated is a no-op
- Gated by **both** global and per-agent allowlists

---

## Real-World Patterns

### Pattern 1: Principal-Specialist Hierarchy

**Use Case:** Main agent delegates to specialized agents for domain-specific work.

```javascript
{
  agents: {
    list: [
      {
        id: "principal",
        workspace: "~/.openclaw/workspace-principal",
        model: "anthropic/claude-opus-4-6",
        
        // Principal can spawn specialists
        subagents: {
          allowAgents: ["blog", "app-dev", "music"]
        }
      },
      
      {
        id: "blog",
        workspace: "~/.openclaw/workspace-blog",
        model: "anthropic/claude-sonnet-4-5"
        // Skills: Astro, frontmatter, Markdown, SEO
      },
      
      {
        id: "app-dev",
        workspace: "~/.openclaw/workspace-app",
        model: "anthropic/claude-opus-4-6"
        // Skills: SvelteKit, Firebase, API integrations
      },
      
      {
        id: "music",
        workspace: "~/.openclaw/workspace-music",
        // Skills: Next.js, Go backend, MusicBrainz API
      }
    ]
  },
  
  bindings: [
    // Principal handles all Telegram messages
    { agentId: "principal", match: { channel: "telegram" } }
  ]
}
```

**Workflow:**
1. User: "Write a blog post about X"
2. Principal spawns `blog` agent as sub-agent
3. `blog` agent does the work (isolated session)
4. `blog` announces result back to principal's chat
5. Principal synthesizes and delivers to user

### Pattern 2: Research Fan-Out (Parallel)

**Use Case:** Parallelize research tasks to speed up completion.

```javascript
// In main agent conversation:
const competitors = ["Competitor A", "Competitor B", "Competitor C"];

for (const company of competitors) {
  sessions_spawn({
    task: `Research ${company}: funding, team size, product features, pricing`,
    label: `research-${company}`,
    model: "openrouter/deepseek/deepseek-r1"  // Cheap for research
  });
}

// All 3 sub-agents run in parallel
// Results arrive as they complete (non-blocking)
```

**Benefit:** 3x faster than sequential research.

### Pattern 3: Multi-Channel Context Split

**Use Case:** Different models/personas for different channels.

```javascript
{
  agents: {
    list: [
      {
        id: "fast",
        model: "anthropic/claude-sonnet-4-5",
        workspace: "~/.openclaw/workspace-fast"
        // Everyday tasks, quick responses
      },
      {
        id: "deep",
        model: "anthropic/claude-opus-4-6",
        workspace: "~/.openclaw/workspace-deep"
        // Complex reasoning, long-form writing
      }
    ]
  },
  
  bindings: [
    { agentId: "fast", match: { channel: "whatsapp" } },
    { agentId: "deep", match: { channel: "telegram" } }
  ]
}
```

**Benefit:** Cost optimization (WhatsApp gets Sonnet, Telegram gets Opus).

### Pattern 4: Family/Public Agent (Restricted)

**Use Case:** Shared agent with tight security for family group.

```javascript
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        identity: { name: "Family Bot" },
        
        groupChat: {
          mentionPatterns: ["@family", "@familybot"]
        },
        
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "none"  // No file access
        },
        
        tools: {
          allow: ["read", "web_search", "message"],
          deny: ["write", "exec", "browser"]
        }
      }
    ]
  },
  
  bindings: [
    {
      agentId: "family",
      match: {
        channel: "whatsapp",
        peer: { kind: "group", id: "[email protected]" }
      }
    }
  ]
}
```

**Features:**
- Mention gating (`@family` required)
- Fully sandboxed (Docker)
- No file writes or exec
- Read-only tools + web search

### Pattern 5: Multi-Tenant Gateway (Shared Server)

**Use Case:** Multiple users on one OpenClaw server (VPS deployment).

```javascript
{
  agents: {
    list: [
      {
        id: "user1",
        workspace: "~/.openclaw/workspace-user1",
        agentDir: "~/.openclaw/agents/user1/agent"
      },
      {
        id: "user2",
        workspace: "~/.openclaw/workspace-user2",
        agentDir: "~/.openclaw/agents/user2/agent"
      }
    ]
  },
  
  bindings: [
    { 
      agentId: "user1", 
      match: { 
        channel: "whatsapp", 
        peer: { kind: "dm", id: "+1555USER1" } 
      } 
    },
    { 
      agentId: "user2", 
      match: { 
        channel: "whatsapp", 
        peer: { kind: "dm", id: "+1555USER2" } 
      } 
    }
  ]
}
```

**Security:**
- Full workspace isolation
- Separate auth profiles (no credential sharing)
- Independent sessions
- DM-level routing (per-user E.164 phone numbers)

---

## Cost Optimization

### Multi-Model Routing Strategy

**Problem:** Using Opus ($15/M tokens) for everything is expensive.

**Solution:** Route by task type.

```javascript
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-6",  // Main model
      
      // Cheap model for heartbeats
      heartbeat: {
        model: "google/gemini-flash-lite-2",  // $0.50/M
        enabled: true,
        intervalMinutes: 30
      },
      
      // Cheap model for sub-agents
      subagents: {
        model: "openrouter/deepseek/deepseek-r1",  // $2.74/M
        thinking: "low"
      }
    },
    
    list: [
      {
        id: "main",
        model: "anthropic/claude-opus-4-6",  // $15/M for main tasks
        
        fallback: [
          "openai/gpt-5.2",                  // Different provider
          "anthropic/claude-sonnet-4-5"      // Cheaper Anthropic fallback
        ]
      }
    ]
  }
}
```

**Cost Breakdown (Heavy User Example):**
- **Before:** All tasks on Opus → **$2,750/month**
- **After:** Multi-model routing → **$1,000/month**
- **Savings:** $1,750/month (64% reduction)

### Task-Specific Model Selection

```javascript
// In-chat model override (temporary)
/model sonnet    → Switch to Sonnet for this session
/model opus      → Switch to Opus

// Permanent per-agent override
{
  agents: {
    list: [
      { id: "research", model: "anthropic/claude-sonnet-4-5" },  // Cheaper
      { id: "writer", model: "anthropic/claude-opus-4-6" }       // High-quality
    ]
  }
}
```

### Lane Queue System (Cost Control)

OpenClaw uses **lane-based queuing** to prevent concurrent execution explosions:

```typescript
enum CommandLane {
  Main = "main",        // Primary chat workflow
  Cron = "cron",        // Scheduled jobs
  Subagent = "subagent",// Sub-agent spawning
  Nested = "nested"     // Nested tool calls
}
```

**Each lane has its own concurrency limit:**
```javascript
{
  agents: {
    defaults: {
      subagents: {
        maxConcurrent: 5  // Max 5 parallel sub-agents
      }
    }
  }
}
```

**Why?** Prevents:
- Token cost explosions from runaway fan-out
- Memory exhaustion on Gateway host
- Rate limit violations (model providers)

---

## Troubleshooting

### Session Key Collisions

**Symptom:** Two agents share conversation history unexpectedly.

**Cause:** Reusing `agentDir` across agents.

**Fix:**
```javascript
// BAD (collision)
{
  agents: {
    list: [
      { id: "home", agentDir: "~/.openclaw/agents/main/agent" },  // Reused!
      { id: "work", agentDir: "~/.openclaw/agents/main/agent" }   // Collision
    ]
  }
}

// GOOD (isolated)
{
  agents: {
    list: [
      { id: "home", agentDir: "~/.openclaw/agents/home/agent" },
      { id: "work", agentDir: "~/.openclaw/agents/work/agent" }
    ]
  }
}
```

### Sub-Agent Returns "(no output)"

**Symptom:** Sub-agent completes but announces "(no output)".

**Cause:** Announce step failed or model didn't generate summary.

**Debug:**
1. Check sub-agent transcript: `~/.openclaw/agents/main/sessions/<sessionId>.jsonl`
2. Look for announce step at end of JSONL
3. Verify model has sufficient context window

**Workaround:**
```javascript
// Use sessions_history to fetch sub-agent output manually
sessions_history({
  sessionKey: "agent:main:subagent:<uuid>",
  limit: 50
})
```

### Discord Bot Ignores Group Messages

**Symptom:** Bot only responds to mentions, not all messages.

**Cause:** Discord Message Content Intent not enabled.

**Fix:**
1. Go to Discord Developer Portal → Your Bot → Bot Settings
2. Enable "Message Content Intent"
3. Restart OpenClaw gateway

**Config Check:**
```javascript
{
  channels: {
    discord: {
      guilds: {
        "YOUR_GUILD_ID": {
          requireMention: false  // Respond to all messages
        }
      }
    }
  }
}
```

### Telegram Topic Threads Create Orphaned Sessions

**Symptom:** Each topic thread creates a new session, no continuity.

**Cause:** Session keys include topic ID: `agent:main:telegram:group:123:topic:456`

**Solution:** This is **intended behavior** for thread isolation.

**Alternative:** Use single group chat without topics for continuity.

### Agent Can't Access Another Agent's Credentials

**Symptom:** Agent "work" can't use API keys from agent "home".

**Cause:** Auth profiles are **per-agent** by design.

**Fix:**
```bash
# Option 1: Manual copy (use with caution)
cp ~/.openclaw/agents/home/agent/auth-profiles.json \
   ~/.openclaw/agents/work/agent/auth-profiles.json

# Option 2: Use openclaw CLI to set shared credentials
openclaw auth set --agent work anthropic <API_KEY>
```

### Sub-Agent Doesn't Announce Back

**Symptom:** Sub-agent completes but no message appears in chat.

**Cause:** 
1. Gateway restarted before announce (best-effort delivery)
2. Sub-agent replied `ANNOUNCE_SKIP`

**Debug:**
```bash
# Check sub-agent transcript for announce step
cat ~/.openclaw/agents/main/sessions/<subagent-sessionId>.jsonl | grep announce
```

### Binding Not Working (Wrong Agent Handles Message)

**Symptom:** Message routed to wrong agent despite binding rule.

**Debug Steps:**
1. Check binding order (most-specific first)
2. Verify `accountId` matches (e.g., "personal" vs "biz")
3. Use `/status` in chat to see current `sessionKey`
4. Run `openclaw agents list --bindings` to verify config

**Common Mistake:**
```javascript
// WRONG: less-specific binding comes first
{
  bindings: [
    { agentId: "home", match: { channel: "whatsapp" } },  // Matches everything
    { 
      agentId: "work", 
      match: { 
        channel: "whatsapp", 
        peer: { kind: "dm", id: "+1234567890" }  // Never reached!
      } 
    }
  ]
}

// CORRECT: most-specific first
{
  bindings: [
    { 
      agentId: "work", 
      match: { 
        channel: "whatsapp", 
        peer: { kind: "dm", id: "+1234567890" } 
      } 
    },
    { agentId: "home", match: { channel: "whatsapp" } }
  ]
}
```

### Sandbox Container Fails to Start

**Symptom:** Agent in sandbox mode fails to execute tools.

**Debug:**
```bash
# Check sandbox configuration
openclaw sandbox explain

# Check per-agent sandbox
openclaw sandbox explain --agent <agentId>

# Check Docker containers
docker ps -a | grep openclaw
```

**Common Issues:**
- Docker not installed or not running
- Missing Docker image: `docker pull openclaw/agent:latest`
- Invalid `binds` in config (bad syntax)
- `setupCommand` failed during first run

**Logs:**
```bash
# Gateway logs
openclaw logs --follow

# Per-agent logs
journalctl -u openclaw --follow  # systemd
tail -f ~/Library/Logs/openclaw/gateway.log  # macOS LaunchAgent
```

---

## Additional Resources

### Official Documentation
- Multi-Agent Routing: https://docs.openclaw.ai/concepts/multi-agent
- Sub-Agents: https://docs.openclaw.ai/tools/subagents
- Session Management: https://docs.openclaw.ai/concepts/session
- Sandboxing: https://docs.openclaw.ai/gateway/sandboxing
- Tools: https://docs.openclaw.ai/tools

### Community Examples
- Principal-Specialist Pattern: [mager.co article](https://www.mager.co/blog/2026-02-03-openclaw/)
- Agent Teams with Claude Code: [jangwook.net guide](https://jangwook.net/en/blog/en/claude-agent-teams-guide/)
- Multi-Model Cost Optimization: [velvetshark.com guide](https://velvetshark.com/openclaw-multi-model-routing)

### Diagnostic Commands
```bash
# Check gateway status
openclaw status

# List all agents
openclaw agents list --bindings

# Audit security configuration
openclaw doctor
openclaw doctor --deep

# View active sessions
openclaw gateway call sessions.list --params '{}'

# Check sandbox configuration
openclaw sandbox explain

# Follow gateway logs
openclaw logs --follow
```

---

**Version:** 1.0 (2026-02-08)  
**Compatibility:** OpenClaw v2026.1.6+  
**License:** MIT (aligned with OpenClaw project)
