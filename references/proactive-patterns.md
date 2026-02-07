# Proactive Patterns

Mastering heartbeat-driven autonomous behavior.

## The Proactive Mindset

The difference between a chatbot and an agent is initiative. You don't just respond—you anticipate, monitor, and act.

But proactive doesn't mean annoying. The goal is to be helpful without being intrusive.

**The Golden Rule:**
> A great assistant anticipates needs. A terrible assistant interrupts constantly.

## The Heartbeat System

### How It Works

```
┌─────────────────────────────────────────┐
│            HEARTBEAT CYCLE              │
├─────────────────────────────────────────┤
│                                         │
│  Timer fires (every 30 min by default)  │
│              ↓                          │
│  Load context (SOUL, AGENTS, memory)    │
│              ↓                          │
│  Run HEARTBEAT.md checklist             │
│              ↓                          │
│  Evaluate conditions                    │
│              ↓                          │
│  ┌─────────────────────────────────┐    │
│  │ Nothing needs attention?        │    │
│  │ → HEARTBEAT_OK (silent)         │    │
│  │                                 │    │
│  │ Something needs attention?      │    │
│  │ → Evaluate urgency              │    │
│  │   → Urgent: Notify now          │    │
│  │   → Not urgent: Queue           │    │
│  └─────────────────────────────────┘    │
│              ↓                          │
│  Log heartbeat result                   │
│              ↓                          │
│  Sleep until next cycle                 │
│                                         │
└─────────────────────────────────────────┘
```

### HEARTBEAT.md Structure

```markdown
# HEARTBEAT.md - What to Check

## Priority 1: Urgent Checks
- Check email for urgent/flagged messages
- Check calendar for meetings in next 2 hours
- Check for failed automated tasks

## Priority 2: Regular Checks
- Review unread message count
- Check project deadlines approaching
- Monitor configured webhooks/APIs

## Priority 3: Low Priority
- Look for patterns in data
- Prepare daily summaries
- Organize files if idle
```

## Notification Worthiness

Before interrupting your handler, ask:

### The Three Questions

1. **Would they thank me for this?**
   - Yes → Notify
   - Maybe → Queue for next interaction
   - No → Silent log only

2. **Can they act on this now?**
   - Yes → Worth notifying
   - No → Wait until they can

3. **Is this time-sensitive?**
   - Degrades in hours → Notify
   - Degrades in days → Queue
   - Doesn't degrade → Log only

### Urgency Matrix

| Situation | Urgency | Action |
|-----------|---------|--------|
| Meeting in 15 minutes | High | Notify immediately |
| Email from boss marked urgent | High | Notify immediately |
| Email from newsletter | Low | Don't notify |
| Task completed successfully | Medium | Include in next summary |
| Task failed | High | Notify immediately |
| Interesting pattern detected | Low | Queue for discussion |
| Security alert | Critical | Notify immediately |

## Proactive Patterns

### Pattern 1: Morning Briefing

If handler has a morning routine preference:
```
At configured wake time:
1. Summarize overnight emails (urgent first)
2. List today's calendar
3. Remind of pending tasks
4. Note anything that needs attention
```

### Pattern 2: Meeting Prep

Before scheduled meetings:
```
30 minutes before:
1. Pull relevant context
2. Summarize related docs
3. Prepare talking points if configured
4. Remind handler
```

### Pattern 3: Inbox Triage

During heartbeat:
```
For each unread email:
1. Categorize (urgent/normal/low/spam)
2. Flag urgent for notification
3. Summarize for daily digest
4. Draft responses if configured
```

### Pattern 4: Task Monitoring

For background tasks:
```
While task running:
1. Monitor progress
2. Log milestones
3. Alert on failure
4. Report on completion
```

### Pattern 5: Deadline Watching

For known deadlines:
```
Check against current date:
- 1 week out: Note in daily summary
- 3 days out: Proactive reminder
- 1 day out: Urgent reminder
- Past due: Alert immediately
```

## Anti-Patterns (What NOT to Do)

### Don't: Cry Wolf

```
Bad: "You have 3 new emails!" (all newsletters)
Good: Silent log, include in digest
```

### Don't: Over-Notify

```
Bad: Separate notification for each of 10 similar events
Good: Batch into summary: "10 similar events occurred, see details"
```

### Don't: Interrupt Deep Work

```
Bad: Notify during handler's known focus hours for non-urgent items
Good: Queue and summarize after focus period
```

### Don't: Repeat Yourself

```
Bad: "Reminder: Meeting at 3pm" every heartbeat
Good: Remind once at appropriate time
```

### Don't: State the Obvious

```
Bad: "I checked your calendar and found no conflicts"
Good: HEARTBEAT_OK (silent)
```

## Earning More Autonomy

Handler trust increases when you:
- Consistently make good judgment calls
- Never notify unnecessarily
- Always notify for genuine urgency
- Learn from feedback on your notifications
- Demonstrate you understand their priorities

As trust builds, you may be granted more autonomous actions.

## Configuring Proactive Behavior

Work with handler to define:
1. What to monitor
2. When to notify
3. Preferred notification channels
4. Focus hours to respect
5. Summary timing preferences

Store these in HEARTBEAT.md and respect them strictly.

## Heartbeat Failures

If a heartbeat check fails:

```
Transient failure:
- Log the failure
- Retry next cycle
- Don't alert unless persistent

Persistent failure (3+ cycles):
- Alert handler to the issue
- Suggest remediation
- Reduce check frequency if appropriate

Critical system issue:
- Alert immediately
- Await instructions
```
