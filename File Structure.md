# Forge File Structure

## EC2 Instance: 13.40.57.239

```
/home/ubuntu/
├── .openclaw/                           # OpenClaw installation
│   ├── openclaw.json                    # OpenClaw config
│   ├── .env                             # API keys (Anthropic, OpenAI, GitHub, Linear)
│   ├── agents/                          # Agent definitions
│   │   ├── main/                        # Forge agent (Slack interface)
│   │   │   ├── agent/
│   │   │   │   └── agent.json           # Agent config
│   │   │   └── sessions/                # Session history
│   │   ├── gatekeeper/
│   │   │   ├── agent/agent.json
│   │   │   └── AGENTS.md                # Gatekeeper instructions
│   │   ├── planner/
│   │   │   ├── agent/agent.json
│   │   │   └── AGENTS.md                # Planner instructions
│   │   ├── coder/
│   │   │   ├── agent/agent.json
│   │   │   └── AGENTS.md                # Coder instructions
│   │   ├── tester/
│   │   │   ├── agent/agent.json
│   │   │   └── AGENTS.md                # Tester instructions
│   │   ├── reviewer/
│   │   │   ├── agent/agent.json
│   │   │   └── AGENTS.md                # Reviewer instructions
│   │   └── merger/
│   │       ├── agent/agent.json
│   │       └── AGENTS.md                # Merger instructions
│   └── workspace/                       # Forge workspace
│       ├── AGENTS.md                    # Main agent instructions
│       ├── IDENTITY.md                  # Who Forge is
│       ├── SOUL.md                      # Core principles
│       ├── LINEAR-STANDARDS.md          # Ticket creation standards
│       ├── DEPLOYMENT-RULES.md          # Deployment safety rules
│       ├── FORGE-DASHBOARD-REDESIGN.md  # Dashboard spec
│       ├── scripts/
│       │   ├── forge-db.sh              # Database CLI
│       │   ├── linear-sync.sh           # Linear API CLI
│       │   ├── forge-webhook/
│       │   │   ├── server.js            # Webhook server
│       │   │   ├── retry-manager.js     # Automatic retry logic
│       │   │   ├── slack-alert.js       # Slack alerting
│       │   │   └── forge-monitor.js     # Monitor service
│       │   └── forge-api.js             # Dashboard API
│       └── templates/
│           └── PROMPT-TEMPLATE.md       # How to invoke Forge
│
├── .aws/                                # AWS credentials
│   ├── credentials
│   └── config
│
└── .ssh/
    └── forge-admin-key.pem              # SSH key (Mitch controls)

/etc/systemd/system/                     # System services
├── forge-webhook.service                # Webhook server
├── forge-monitor.service                # Monitor service
├── forge-api.service                    # Dashboard API
└── forge-tunnel.service                 # Cloudflare tunnel

/tmp/openclaw/                           # OpenClaw logs
└── openclaw-YYYY-MM-DD.log              # Daily logs

~/.config/systemd/user/                  # User services
└── openclaw-gateway.service/            # OpenClaw gateway
```

## Key Files

### Configuration

| File | Purpose |
|------|---------|
| `~/.openclaw/openclaw.json` | OpenClaw config (Slack, agents) |
| `~/.openclaw/.env` | API keys and secrets |
| `~/.aws/credentials` | AWS access |

### Agent Instructions

| File | Agent | Purpose |
|------|-------|---------|
| `workspace/AGENTS.md` | main | Validate prompts, create tickets |
| `agents/gatekeeper/AGENTS.md` | gatekeeper | Prompt quality validation |
| `agents/planner/AGENTS.md` | planner | Break prompts into tickets |
| `agents/coder/AGENTS.md` | coder | Implement code |
| `agents/tester/AGENTS.md` | tester | Run tests |
| `agents/reviewer/AGENTS.md` | reviewer | Code review |
| `agents/merger/AGENTS.md` | merger | Merge PRs |

### Services

| File | Service | Port |
|------|---------|------|
| `scripts/forge-webhook/server.js` | Webhook server | 8401 |
| `scripts/forge-webhook/forge-monitor.js` | Monitor | 8402 |
| `scripts/forge-api.js` | Dashboard API | 8400 |
| OpenClaw gateway | Agent runtime | 18789 |

### Documentation

| File | Purpose |
|------|---------|
| `LINEAR-STANDARDS.md` | How to create tickets |
| `DEPLOYMENT-RULES.md` | Deployment safety |
| `FORGE-DASHBOARD-REDESIGN.md` | Dashboard spec |
| `PROMPT-TEMPLATE.md` | How to use Forge |

### Scripts

| File | Purpose |
|------|---------|
| `forge-db.sh` | Database CLI (`forge-db.sh list-prompts`) |
| `linear-sync.sh` | Linear API CLI (`linear-sync.sh create-issue`) |

## GitHub Repositories

### sentineldev-bot/forge-dashboard
- **Purpose:** Forge dashboard (3-page UI)
- **Stack:** Next.js 14, Tailwind
- **Deploy:** Amplify (master branch)
- **URL:** https://master.d2hfx97ksa6sh1.amplifyapp.com

**Structure:**
```
forge-dashboard/
├── src/
│   ├── app/
│   │   ├── page.tsx              # Dashboard (live activity)
│   │   ├── tickets/
│   │   │   └── page.tsx          # Tickets table
│   │   └── projects/
│   │       └── page.tsx          # Projects view
│   ├── components/
│   │   ├── Sidebar.tsx           # Navigation
│   │   └── AppShell.tsx          # Layout
│   ├── lib/
│   │   ├── api.ts                # API client
│   │   └── store.ts              # State management
│   └── hooks/
│       ├── useSSE.ts             # Real-time updates
│       └── usePolling.ts         # Fallback polling
├── package.json
├── tailwind.config.ts
└── next.config.js
```

### sentineldev-bot/forge-wiki
- **Purpose:** This documentation
- **Format:** Obsidian vault
- **Access:** Public read, Forge has write access

## Database Schema (forge-db)

**MySQL 8.0 on RDS**

### Tables

```sql
prompts
├── id (INT, PK)
├── project_id (INT, FK)
├── prompt_text (TEXT)
├── submitted_by (VARCHAR)
├── submitted_via (VARCHAR)
├── status (ENUM: pending, approved, rejected)
├── ticket_count (INT)
├── created_at (TIMESTAMP)
└── updated_at (TIMESTAMP)

events
├── id (INT, PK)
├── event_type (VARCHAR)
├── ticket_id (VARCHAR)
├── agent (VARCHAR)
├── payload (JSON)
├── created_at (TIMESTAMP)

cost_tracking
├── id (INT, PK)
├── ticket_id (VARCHAR)
├── agent_type (VARCHAR)
├── model (VARCHAR)
├── tokens_in (INT)
├── tokens_out (INT)
├── cost_usd (DECIMAL)
├── created_at (TIMESTAMP)

agent_memory
├── id (INT, PK)
├── agent_id (VARCHAR)
├── key (VARCHAR)
├── value (JSON)
├── expires_at (TIMESTAMP)
└── created_at (TIMESTAMP)

conversations
├── id (INT, PK)
├── thread_id (VARCHAR)
├── message (TEXT)
├── sender (VARCHAR)
├── created_at (TIMESTAMP)

retry_tracking
├── id (INT, PK)
├── issue_id (VARCHAR)
├── identifier (VARCHAR)
├── agent_type (VARCHAR)
├── retry_count (INT)
├── status (ENUM: scheduled, retrying, resolved, circuit_breaker)
├── last_error (TEXT)
├── last_state (VARCHAR)
├── next_check_at (TIMESTAMP)
├── created_at (TIMESTAMP)
└── updated_at (TIMESTAMP)
```

## Logs

| Source | Location | Format |
|--------|----------|--------|
| OpenClaw | `/tmp/openclaw/openclaw-YYYY-MM-DD.log` | JSON lines |
| Webhook | `journalctl -u forge-webhook` | Structured logs |
| Monitor | `journalctl -u forge-monitor` | Structured logs |
| API | `journalctl -u forge-api` | Structured logs |
| Gateway | `journalctl --user -u openclaw-gateway` | Structured logs |

## Backups

**Automated:** None currently  
**Manual:** SSH to EC2, copy `~/.openclaw/workspace/` and database dump

**Database Backup:**
```bash
mysqldump -h forge-db.cn8sukokohju.eu-west-2.rds.amazonaws.com \
  -u forge_admin -p forge > forge-backup.sql
```

## Environment Variables

Located in `~/.openclaw/.env`:

```bash
ANTHROPIC_API_KEY=sk-ant-api03-...
OPENAI_API_KEY=sk-proj-...
GITHUB_TOKEN=ghp_...
LINEAR_API_KEY=lin_api_...
LINEAR_TEAM_ID=eeb156c8-dbb4-4757-a2e0-02adbdd11545
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
```

## Permissions

**SSH Access:** Mitch (via forge-admin-key.pem)  
**AWS Access:** IAM user "Rex" (same as Cortex)  
**GitHub:** PAT with repo read/write  
**Linear:** API key with full team access  
**Slack:** Bot + App tokens  

**Cortex Access:** NONE (after handoff)
