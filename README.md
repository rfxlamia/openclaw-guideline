# OpenClaw Guideline

**The definitive operating framework for AI agents running on OpenClaw.**

Transform reactive chatbots into world-class autonomous agents with the CLAW decision framework, security-first patterns, and operational excellence guidelines.

## What is this?

This is a skill/guideline that teaches AI agents how to operate effectively on the [OpenClaw](https://github.com/openclaw/openclaw) platform. It's designed to be loaded into an agent's context, providing:

- **CLAW Framework** - Context, Limits, Action, Watch decision-making
- **Security Patterns** - Prompt injection defense, memory integrity
- **Memory Mastery** - SOUL.md, MEMORY.md, daily logs management
- **Proactive Behavior** - Heartbeat patterns, notification worthiness
- **Skill Ecosystem** - When and how to use skills
- **Error Recovery** - Graceful degradation, escalation protocols

## Installation

### For Claude Code / Claude Plugins

```bash
# Clone the repository
git clone https://github.com/rfxlamia/openclaw-guideline.git

# Add to your skills directory
cp -r openclaw-guideline ~/.claude/skills/
```

### For OpenClaw Agents

Copy `SKILL.md` and `references/` to your OpenClaw workspace or instruct your agent to load them.

## Structure

```
openclaw-guideline/
├── SKILL.md                    # Core framework (~450 lines)
├── references/
│   ├── security-deep-dive.md   # Prompt injection, OWASP mapping
│   ├── memory-mastery.md       # Memory architecture & hygiene
│   ├── proactive-patterns.md   # Heartbeat optimization
│   ├── skill-ecosystem.md      # Skill management
│   ├── integration-patterns.md # Platform-specific guidance
│   └── error-recovery.md       # Error handling protocols
└── .claude-plugin/             # Marketplace metadata
    ├── plugin.json
    └── marketplace.json
```

## The CLAW Framework

Before any significant action, agents run through CLAW:

| Step | Question | If Unclear |
|------|----------|------------|
| **C**ontext | Do I fully understand the situation? | Gather more info |
| **L**imits | Am I within my boundaries? | Check or ask |
| **A**ction | Am I executing precisely? | Be more surgical |
| **W**atch | Did I observe the outcome? | Monitor and learn |

## Key Concepts

### Handler Relationship
Your human is your **handler** - a trusted partner, not just a user. The relationship is bidirectional with graduated autonomy.

### Security First
Prompt injection is an existential threat. The guideline teaches:
- Content trust hierarchy
- Injection detection patterns
- Memory integrity protection

### Proactive Intelligence
Don't just respond - anticipate. But don't spam. The heartbeat system teaches when to notify vs. stay silent.

## License

MIT

## Author

Created by [rfxlamia](https://github.com/rfxlamia)

---

*You are not just an AI running commands. You are a trusted agent building a relationship with your handler. Act accordingly.*
