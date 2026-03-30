# Usage Guide

## How to Use Forge

Forge takes natural language prompts and turns them into shipped code. Here's how to use it.

## Sending Prompts

### Via Slack (Primary Method)

1. Open Slack
2. Find the Forge bot DM
3. Send your prompt:

```
Build a todo list app with:
- Add/remove items
- Mark as complete
- Responsive design
Deploy to Amplify
```

4. Forge validates and creates tickets
5. Watch progress on dashboard
6. Get notified when done

### Prompt Quality

**Good prompts:**
- Clear goal ("Build a timer app")
- Key features listed
- Acceptance criteria obvious

**Bad prompts:**
- Too vague ("Make the site better")
- No details ("Add a feature")
- Conflicting requirements

**Override:**
If Forge asks questions you don't want to answer, say "override" and it will proceed with defaults.

## Ticket Creation

Forge creates tickets in Linear automatically.

**Every ticket has:**
- Clear title
- Description with acceptance criteria
- Labels: `type:*` and `priority:*`
- State: Usually "Todo"

**Example ticket:**
```
Title: Add dark mode toggle
Description: Users want dark mode.
Acceptance criteria:
- [ ] Toggle in settings
- [ ] Persists across sessions
- [ ] Applies to all pages
Labels: type:feature, priority:normal
State: Todo
```

## Pipeline Flow

Once a ticket is created, the pipeline runs automatically:

### 1. Coder Phase
- Ticket moves from "Todo" to "In Progress"
- Coder agent spawns
- Implements the feature
- Creates a PR
- Moves ticket to "In Review"

### 2. Reviewer Phase
- Reviewer agent spawns
- Reviews code for quality, security, correctness
- If pass: moves to "Testing"
- If fail: moves back to "Todo" with feedback

### 3. Tester Phase
- Tester agent spawns
- Runs tests
- Verifies acceptance criteria
- If pass: moves to "Ready to Merge"
- If fail: moves back to "Todo"

### 4. Merger Phase
- Merger agent spawns
- Merges PR (squash merge)
- Moves ticket to "Done"
- You get notified

**Total time:** 5-30 minutes depending on complexity

## Monitoring Progress

### Dashboard

Visit: https://master.d2hfx97ksa6sh1.amplifyapp.com

**Dashboard page:**
- Live activity feed (what's happening right now)
- Running agents (which ticket, how long)
- Recent completions
- Errors

**Tickets page:**
- All Linear tickets
- Inline agent status (C/R/T/M badges)
- Click to expand details
- Filter by state/project

**Projects page:**
- Group tickets by project
- Health indicators (in progress, done, blocked)
- Progress bars

### Linear

Visit Linear directly: https://linear.app/sentinelopenclaw/team/SEN

See tickets in their native environment with full Linear features.

### Slack Alerts

Forge DMs you when:
- Tickets are stuck (>15 min with no progress)
- Tickets blocked after 3 failed retries
- Agent type fails across multiple tickets
- Webhook server down

## Common Scenarios

### Building a New Feature

```
Build a user profile page with:
- Display name, email, avatar
- Edit form
- Save to database
- Responsive
Repo: sentineldev-bot/operations-dashboard
```

Forge creates 1 ticket → pipeline runs → PR merged → done.

### Fixing a Bug

```
Fix: Payment button doesn't work on mobile Safari
Repo: sentineldev-bot/payment-flow
Priority: high
```

Forge creates 1 ticket with `type:bug, priority:high` → pipeline runs.

### Building an App from Scratch

```
Build a pomodoro timer app:
- 25 min work / 5 min break timer
- Sound notification
- Session history
- Dark theme
- Deploy to Amplify as new repo
```

Forge creates 5-10 tickets (features broken down) → all process through pipeline.

## Advanced Usage

### Manual Retry

If a ticket gets stuck and monitoring doesn't auto-retry:

1. SSH to Forge: `ssh -i forge-admin-key.pem ubuntu@13.40.57.239`
2. Trigger retry: `curl -X POST http://localhost:8402/retry/SEN-XXX`

### Check Health

```bash
curl http://13.40.57.239:8402/health/stuck-tickets
curl http://13.40.57.239:8402/health/failed-agents
```

### View Logs

**OpenClaw:**
```bash
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

**Webhook:**
```bash
ssh -i forge-admin-key.pem ubuntu@13.40.57.239 'sudo journalctl -u forge-webhook -f'
```

**Monitor:**
```bash
ssh -i forge-admin-key.pem ubuntu@13.40.57.239 'sudo journalctl -u forge-monitor -f'
```

### Database Queries

**SSH to Forge:**
```bash
ssh -i forge-admin-key.pem ubuntu@13.40.57.239
```

**Query prompts:**
```bash
bash ~/.openclaw/workspace/scripts/forge-db.sh list-prompts
```

**Query events:**
```bash
bash ~/.openclaw/workspace/scripts/forge-db.sh list-events --limit 50
```

**Query costs:**
```bash
bash ~/.openclaw/workspace/scripts/forge-db.sh get-costs --period today
```

### Linear Operations

**Create ticket manually:**
```bash
ssh -i forge-admin-key.pem ubuntu@13.40.57.239
bash ~/.openclaw/workspace/scripts/linear-sync.sh create-issue \
  --title "Your title" \
  --description "Description" \
  --state todo \
  --labels "type:feature,priority:normal"
```

**Update ticket state:**
```bash
bash ~/.openclaw/workspace/scripts/linear-sync.sh update-state <UUID> in_progress
```

## Troubleshooting

### Ticket Stuck

**Symptoms:** Ticket hasn't moved in >15 min  
**Solution:** Wait for monitor to detect and auto-retry, or manually trigger retry

### Agent Failed

**Symptoms:** Ticket moved back to "Todo" with error comment  
**Solution:** Check error message in Linear, fix issue, move ticket back to "Todo"

### Webhook Not Firing

**Symptoms:** Tickets created but no agents spawning  
**Solution:** Check `journalctl -u forge-webhook` for errors, verify Cloudflare tunnel is up

### Dashboard Not Loading

**Symptoms:** Dashboard shows error or stale data  
**Solution:** Check `journalctl -u forge-api`, verify RDS and Linear API accessible

### OpenClaw Gateway Down

**Symptoms:** Can't message Forge in Slack  
**Solution:** `systemctl --user restart openclaw-gateway` on Forge EC2

See [[Troubleshooting]] for full guide.

## Best Practices

### Prompt Writing

1. **Be specific** - "Add a timer" not "improve the site"
2. **List acceptance criteria** - What does "done" look like?
3. **Mention the repo** - Which codebase?
4. **Set priority** - critical/high/normal/low

### Ticket Management

1. **Let the pipeline run** - Don't manually intervene unless stuck
2. **Check dashboard first** - See what's happening before asking
3. **Read error messages** - Agents leave detailed comments on failures

### Monitoring

1. **Glance at dashboard daily** - Spot stuck tickets early
2. **Check Slack alerts** - Forge DMs you when things break
3. **Linear is source of truth** - Always check Linear for ticket status

## Limits

**Concurrent tickets:** ~10-20 (depends on complexity)  
**Ticket size:** Single-feature changes work best  
**Retry limit:** 3 automatic retries before circuit breaker  
**Timeout:** Agents have ~30 min per stage  

## Getting Help

1. Check [[Troubleshooting]]
2. Check dashboard for live status
3. Check Linear comments for agent error messages
4. SSH to Forge and check logs
5. Ask Mitch

## Examples

See [[Examples]] for real-world prompts and their results.
