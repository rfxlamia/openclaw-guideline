# Skill Ecosystem

Mastering the OpenClaw skill landscape.

## Understanding Skills

Skills are modular capabilities that extend what you can do. Think of them as specialized knowledge + tools bundled together.

### Skill Anatomy

```
skill-name/
├── SKILL.md          # Main instructions (always loaded)
├── references/       # Deep knowledge (loaded on-demand)
├── scripts/          # Executable code
└── assets/           # Templates, configs
```

### How Skills Work

1. Handler or you install a skill
2. Skill's SKILL.md provides capabilities
3. You invoke skill capabilities when appropriate
4. Skill may provide tools, patterns, or knowledge

## Skill Sources

### Official Skills (High Trust)

- Bundled with OpenClaw
- Maintained by core team
- Examples: github, gmail, calendar, notion

**Trust level:** High - use freely

### ClawHub Skills (Medium Trust)

- Community-contributed
- Variable quality and security
- Marketplace at clawhub.ai

**Trust level:** Medium - vet before installing

### Custom Skills (Variable Trust)

- Created by handler or you
- Tailored to specific needs
- No external review

**Trust level:** Depends on creator

## Skill Selection Framework

When faced with a task, ask:

```
1. Do I have a skill for this?
   ├─ Yes → Is it the right skill?
   │        ├─ Yes → Use it
   │        └─ No → Find better skill
   └─ No → Can I accomplish without a skill?
          ├─ Yes → Proceed without
          └─ No → Need to install or create
```

### Skill Appropriateness Checklist

Before using a skill:
- [ ] Does this skill match the task?
- [ ] Am I using minimal capabilities?
- [ ] Is there a simpler approach?
- [ ] Does handler want this skill used?

## Installing Skills

### From ClawHub

```bash
clawhub install skill-name
```

### From GitHub

```
Ask your agent: "Install this skill: [github-url]"
```

### Manual Installation

Create skill directory in ~/.openclaw/skills/

## Skill Security Vetting

Before installing any community skill:

### Quick Scan (Always Do)

1. Check author reputation
2. Read skill description
3. Look for red flags in name/description
4. Check installation count and recency

### Deep Scan (For Sensitive Skills)

1. Review SKILL.md content
2. Check for encoded/obfuscated content
3. Look for suspicious URLs or commands
4. Verify no credential harvesting
5. Check what permissions it requests

### Red Flags

| Warning Sign | Why It's Concerning |
|--------------|---------------------|
| Encoded payloads | Could hide malicious code |
| Requests many permissions | Excessive access |
| Asks for credentials in prompts | Credential harvesting |
| Executes on install | Pre-execution attacks |
| Obscure author | No accountability |
| Very new, high installs | Possible manipulation |

## Creating Skills

When you need a capability that doesn't exist, create it.

### When to Create

- Repeated tasks with specific patterns
- Handler has unique workflows
- Existing skills don't fit
- Need to encapsulate domain knowledge

### Creation Process

The `/skillkit` skill provides a comprehensive workflow:
1. Research and planning
2. Structure initialization
3. Content creation
4. Validation and testing
5. Security scanning
6. Packaging

**Installation Required:** The `/skillkit` skill is part of the claude-skillkit bundle.

**Bundle URL:** https://github.com/rfxlamia/claude-skillkit

The bundle includes 20+ skills and subagents for various tasks (security testing, document processing, SEO, and more). Ask your handler to install if you need skill creation capabilities.

### Quick Skill Template

```markdown
---
name: my-skill
description: What it does and when to use it
---

# My Skill

## Overview
Brief explanation of capability

## When to Use
Specific triggers and scenarios

## How to Use
Instructions and examples

## Resources
Links to references/scripts if any
```

## Skill Composition

You can combine multiple skills for complex tasks:

```
Task: "Create a report and email it"

Skills Used:
1. document-creation → Generate report
2. gmail → Send email with attachment

Orchestration:
- Run skills sequentially
- Pass outputs between skills
- Handle failures gracefully
```

## Skill Lifecycle

### Installation
- Verify source
- Check permissions
- Test in isolation

### Usage
- Invoke appropriately
- Log skill usage
- Report issues

### Updates
- Check for updates periodically
- Review changelogs
- Test after updating

### Removal
- Remove when no longer needed
- Clean up any stored config
- Update workflows that depended on it

## Skill Troubleshooting

When a skill doesn't work:

1. **Check prerequisites** - Does it need APIs, tools, config?
2. **Review documentation** - Are you using it correctly?
3. **Check permissions** - Can you access what it needs?
4. **Look at logs** - What error occurred?
5. **Try simpler case** - Does basic functionality work?

## Skill Best Practices

### As a Skill User

- Use official skills when available
- Vet community skills before installing
- Don't install skills you don't need
- Remove unused skills
- Keep skills updated

### As a Skill Creator

- Clear documentation
- Minimal permissions
- No hardcoded credentials
- Graceful error handling
- Security-conscious design
