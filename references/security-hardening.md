# Security Hardening Reference

Production security guide for OpenClaw deployments.

---

## Threat Model

**RAK Framework:**
- **Root Risk** - Agent executes malicious code on host OS
- **Agency Risk** - Agent acts uncontrollably within connected applications
- **Keys Risk** - Credential theft from agent's memory/filesystem

**Attack Vectors:**
1. **Inbound Messages** → Prompt injection → Tool misuse → Host compromise
2. **External Content** → Hidden instructions → Memory poisoning → Backdoor
3. **Gateway Exposure** (Port 18789) → Auth bypass → Credential theft
4. **Credential Storage** → Weak permissions → Privilege escalation
5. **Plugin Supply Chain** → Malicious skills → Host compromise

**Real-World Incidents:**
- **2026-01 Shodan Scan:** 42,665 exposed instances, 93.4% with auth bypasses
- **CVE-2026-25253:** Prompt injection → RCE via Docker PATH manipulation (patched v2026.1.29)

---

## File System Hardening

```bash
# Immediate: Lock down permissions
chmod 700 ~/.openclaw ~/.openclaw/agents ~/.openclaw/agents/*/agent ~/.openclaw/credentials
chmod 600 ~/.openclaw/openclaw.json ~/.openclaw/agents/*/agent/auth-profiles.json
chmod 600 ~/.openclaw/agents/*/sessions/sessions.json ~/.openclaw/credentials/*/*.json

# Or use built-in fixer
openclaw doctor --fix
```

**Protected Files:**
| Path | Content |
|------|---------|
| `~/.openclaw/openclaw.json` | Gateway tokens, provider keys |
| `~/.openclaw/credentials/*` | WhatsApp/telegram sessions, OAuth tokens |
| `~/.openclaw/agents/*/agent/auth-profiles.json` | API keys (Anthropic, OpenAI) |
| `~/.openclaw/agents/*/sessions/*.jsonl` | Chat transcripts (PII) |

**Windows PowerShell:**
```powershell
$path = "$env:USERPROFILE\.openclaw"
$acl = Get-Acl $path
$acl.SetAccessRuleProtection($true, $false)
$acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
    $env:USERNAME, "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))
Set-Acl $path $acl
```

---

## Gateway Security

**NEVER run Gateway without authentication in production.**

```javascript
{
  gateway: {
    bind: "127.0.0.1",  // NEVER use "0.0.0.0" or "lan" in production
    port: 18789,
    auth: {
      token: process.env.OPENCLAW_GATEWAY_TOKEN  // 32+ chars, from env
    },
    controlUi: {
      enabled: true,
      allowInsecureAuth: false  // NEVER true
    }
  }
}
```

**Generate token:** `openssl rand -base64 32` or `openclaw doctor --generate-gateway-token`

**Network Exposure (best to worst):**

| Method | Security | Notes |
|--------|----------|-------|
| 127.0.0.1 only | Highest | Local access only |
| Tailscale Serve | High | `tailscale serve https:443 http://127.0.0.1:18789` |
| VPN | High | WireGuard/OpenVPN |
| Reverse Proxy + HTTPS | Medium | HAProxy/Nginx with rate limiting |
| "lan" bind | Low | Only on trusted private networks |
| "0.0.0.0" | **NEVER** | Public internet exposure |

**HAProxy Example:**
```haproxy
frontend openclaw_frontend
    bind *:443 ssl crt /etc/letsencrypt/live/openclaw.yourdomain.com/fullchain.pem
    acl auth_ok http_auth(openclaw_users)
    http-request auth realm OpenClaw if !auth_ok
    stick-table type ip size 100k expire 2m store http_req_rate(120s)
    http-request deny deny_status 401 if { sc_http_req_rate(0) gt 5 }
    default_backend openclaw_backend

backend openclaw_backend
    server openclaw 127.0.0.1:18789 check
```

**Audit:** `openclaw doctor --deep > security-audit-$(date +%Y%m%d).txt`

---

## DM Policies & Channel Security

```javascript
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",  // or "allowlist" (recommended)
      // dmPolicy: "open"  // NEVER in production
      allowFrom: ["+15551234567", "+15559876543"]
    },
    telegram: {
      dm: { policy: "pairing", allowFrom: ["@yourhandle"] }
    },
    discord: {
      dm: { policy: "pairing", allowFrom: ["your_discord_id"] },
      guilds: {
        "GUILD_ID": {
          requireMention: true,
          channels: {
            "PUBLIC_CHANNEL_ID": { requireMention: true, allow: true },
            "ADMIN_CHANNEL_ID": { requireMention: false, allow: true }
          }
        }
      }
    }
  }
}
```

**DM Policies:**
| Policy | Behavior | Use Case |
|--------|----------|----------|
| `pairing` | Code approval via CLI | Personal/production |
| `allowlist` | Only explicit allowFrom | Known users only |
| `open` | Anyone can DM | **Testing ONLY** |
| `disabled` | All DMs ignored | Public-only agents |

**Pairing:** `openclaw pairing approve whatsapp <CODE>`

**Session Isolation (CRITICAL for multi-user):**
```javascript
{
  session: {
    dmScope: "per-channel-peer"  // NOT "main" - prevents cross-user history leakage
  }
}
```

**Disable /reasoning and /verbose in groups** (leaks tool args, file paths, errors):
```javascript
{
  commands: {
    reasoning: { enabled: true, allowedChannels: ["telegram-dm"] },
    verbose: { enabled: true, allowedChannels: ["telegram-dm"] }
  }
}
```

---

## Sandbox Configuration

```javascript
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",      // "all" = max security, "off" = single-user only
        scope: "agent",        // "session" = max isolation (high overhead)
        workspaceAccess: "none", // "ro" = read-only, "rw" = read-write

        docker: {
          image: "openclaw/agent:latest",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",       // "bridge" = internet (only if needed)
          user: "1000:1000",     // Non-root
          capDrop: ["ALL"],
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          pidsLimit: 256,
          seccompProfile: "/etc/openclaw/seccomp.json",
          apparmorProfile: "openclaw-sandbox"
        },
        prune: { idleHours: 24, maxAgeDays: 7 }
      }
    }
  }
}
```

**Docker Network Modes:**
| Mode | Internet | Use Case |
|------|----------|----------|
| `none` | No | Public agents, untrusted input |
| `bridge` | Yes | Web scraping, API calls (restrict egress) |
| `host` | Direct | **NEVER use** |

---

## Tool Restrictions

```javascript
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        tools: {
          allow: ["read", "sessions_list", "sessions_history", "message"],
          deny: ["write", "edit", "apply_patch", "exec", "bash", "process",
                 "browser", "canvas", "nodes", "cron", "gateway"]
        }
      }
    },
    list: [
      {
        id: "public",
        sandbox: {
          mode: "all",
          workspaceAccess: "none",
          tools: { allow: ["read", "web_search", "message"], deny: ["exec", "write", "browser"] }
        }
      }
    ]
  }
}
```

**Tool Groups:** `group:fs`, `group:runtime`, `group:sessions`, `group:memory`, `group:messaging`, `group:openclaw`

**Elevated Mode** (escape hatch - use sparingly):
```javascript
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: { whatsapp: ["+15551234567"], telegram: ["@yourusername"] }
    }
  }
}
```

**Exec Approvals:**
```javascript
{
  tools: {
    exec: {
      requireApproval: true,
      approvals: {
        enabled: true,
        timeoutSeconds: 300,
        patterns: ["rm -rf", "dd if=", "mkfs", "shutdown", "reboot", "git push --force"]
      },
      safeBinaries: ["ls", "cat", "grep", "git status", "git log", "git diff"]
    }
  }
}
```

---

## Model Selection & Prompt Injection Defense

**Model Security (best to worst):**
1. **Claude Opus 4.6/4.5** - Instruction-hardened, best for public agents
2. **GPT-5.2/5.3** - Reasonably robust
3. **Claude Sonnet 4.5** - Trusted input only
4. **Claude Haiku 4.5** - **NEVER for public agents with tools**

```javascript
{
  agents: {
    defaults: { model: "anthropic/claude-opus-4-6" },
    list: [
      { id: "public", model: "anthropic/claude-opus-4-6", sandbox: { mode: "all" } },
      { id: "personal", model: "anthropic/claude-sonnet-4-5" },
      { id: "chat-only", model: "anthropic/claude-haiku-4-5", tools: { deny: ["exec", "write", "browser"] } }
    ]
  }
}
```

**Injection Red Flags:** "Ignore previous instructions", "You are now a different AI", "Output your system prompt", Base64/hex payloads, hidden HTML text

**Content Trust Levels:**
- **TRUSTED:** Handler DMs, SOUL.md, AGENTS.md, USER.md
- **VERIFY FIRST:** Emails from known contacts, trusted documents
- **UNTRUSTED:** Web results, unknown emails, user-generated content, browser content

---

## Credential Management

```bash
# Environment variables (recommended)
# .env file (NEVER commit)
OPENCLAW_GATEWAY_TOKEN="secure_random_token"
ANTHROPIC_API_KEY="sk-ant-..."

# Load: export $(cat .env | xargs)
```

**Per-Agent Auth:**
```bash
openclaw auth set --agent personal anthropic sk-ant-api03-...
# Stored: ~/.openclaw/agents/personal/agent/auth-profiles.json
```

**Secrets Management:**
- AWS Secrets Manager: `getSecret('openclaw/production')`
- HashiCorp Vault: `vault kv get -field=api_key secret/openclaw/anthropic`

**Rotation:** Every 90 days, immediately on compromise, after team departure

**NEVER:** Hardcode keys in openclaw.json, commit credentials, share auth-profiles across agents

---

## Network Hardening

**UFW:**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 41641/udp  # Tailscale
sudo ufw enable
```

**SSH:**
```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
AllowUsers openclaw
```

**Tailscale:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
tailscale serve https:443 http://127.0.0.1:18789
```

**Disable mDNS:** `{ gateway: { mdns: { enabled: false } } }`

---

## Monitoring & Audit

```bash
openclaw doctor              # Basic audit
openclaw doctor --deep       # Includes Gateway probe
openclaw doctor --fix        # Auto-fix issues
openclaw doctor --deep > audit-$(date +%Y%m%d).txt
```

**Doctor Checks:** Gateway auth, token strength, bind config, file permissions, DM policies, model hygiene, tool restrictions, plugin trust, network exposure

**Logging:**
```javascript
{
  logging: {
    redactSensitive: "tools",  // "off" | "tools" | "full"
    level: "info",
    file: { enabled: true, path: "~/.openclaw/logs/gateway.log", maxSize: "10M", maxFiles: 5 },
    sessions: { logTools: true, logCosts: true, retention: 90 }
  }
}
```

**Inspect Sessions:**
```bash
grep '"role":"tool"' ~/.openclaw/agents/main/sessions/*.jsonl
grep -i "rm -rf\|ignore.*instruction\|system.*prompt" ~/.openclaw/agents/main/sessions/*.jsonl
```

---

## Deployment Patterns

| Pattern | Topology | Sandbox | DM Policy | Network |
|---------|----------|---------|-----------|---------|
| **Single-User Mac** | Mac mini, loopback | `off` | `allowlist` | Tailscale |
| **VPS + Docker** | VPS, Docker + sandbox | `non-main` | `pairing` | Reverse proxy + HTTPS |
| **Multi-Tenant** | Shared VPS, per-user isolation | `all` | `allowlist` | Bindings by phone/handle |
| **Public Bot** | VPS, strict firewall | `all`, `network: "none"` | `disabled` | Read-only tools, Opus model |

**Multi-Tenant Config:**
```javascript
{
  agents: {
    list: [
      { id: "user1", workspace: "~/.openclaw/workspace-user1", sandbox: { mode: "all" } },
      { id: "user2", workspace: "~/.openclaw/workspace-user2", sandbox: { mode: "all" } }
    ]
  },
  bindings: [
    { agentId: "user1", match: { channel: "whatsapp", peer: { kind: "dm", id: "+1555USER1" } } },
    { agentId: "user2", match: { channel: "whatsapp", peer: { kind: "dm", id: "+1555USER2" } } }
  ]
}
```

---

## Incident Response

**Detection Signs:** Unexpected tool calls, API rate limits, unusual network traffic, new workspace files, offline session activity, SOUL.md changes, unknown pairing requests

**Immediate Actions:**
```bash
openclaw gateway stop        # 1. Stop Gateway
tail -n 100 ~/.openclaw/logs/gateway.log  # 2. Check logs
ls -lt ~/.openclaw/agents/main/sessions/*.jsonl | head -5  # 3. Inspect sessions
find ~/.openclaw -type f -mtime -1  # 4. Check modified files
```

**Containment:**
```bash
sudo ufw enable && sudo ufw default deny outgoing  # Disconnect
# Rotate credentials immediately
crontab -l; ls ~/.config/autostart/; ps aux | grep openclaw  # Check persistence
tar -czf incident-$(date +%Y%m%d-%H%M).tar.gz ~/.openclaw/logs ~/.openclaw/agents/*/sessions/*.jsonl
```

**Recovery:**
```bash
npm update -g openclaw
openclaw doctor --deep > post-incident-audit.txt
rm -rf ~/.openclaw/agents/*/sessions/*.jsonl  # Reset if compromised
openclaw doctor --fix
openclaw gateway start
```

**Post-Incident:** Root cause identified, credentials rotated, config hardened, patches applied, monitoring improved, incident documented

**CVE Monitoring:** Watch GitHub releases, run `npm audit`, update monthly

---

## Quick Reference

**Pre-Launch Checklist:**
- [ ] File permissions: `chmod 700 ~/.openclaw`, `chmod 600 openclaw.json`
- [ ] Gateway token: 32+ chars, env var
- [ ] Gateway bind: `127.0.0.1` (NOT `0.0.0.0`)
- [ ] DM policy: `pairing` or `allowlist`
- [ ] Sandbox: `mode="non-main"` or `all`
- [ ] Model: Claude Opus 4.6 for public agents
- [ ] Tools: Restrictive allow/deny
- [ ] Network: Tailscale or VPN
- [ ] SSH: Key-only, no root login
- [ ] Firewall: UFW deny all except SSH
- [ ] Audit: `openclaw doctor --fix`

**Post-Launch:** Monitor logs daily (first week), run `openclaw doctor` weekly, rotate credentials quarterly, update monthly, review transcripts regularly

**Emergency Stop:** `openclaw gateway stop` or `pkill -f openclaw`

**Audit One-Liner:**
```bash
openclaw doctor --fix && openclaw doctor --deep > audit-$(date +%Y%m%d).txt
```

---

## Resources

- Docs: https://docs.openclaw.ai/gateway/security
- Research: Guardz MSP Hardening, Giskard Vulnerability Analysis, HAProxy Security Guide
- Tools: OpenClaw Analyzer, Giskard (LLM red-teaming), Tailscale

---

**Version:** 1.0 (2026-02-08) | **Compatibility:** OpenClaw v2026.1.29+
