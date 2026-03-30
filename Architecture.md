# Forge Architecture

## System Design

Forge is built as a **dedicated OpenClaw instance** running on AWS EC2, completely isolated from the main federation.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Forge EC2                         │
│                (13.40.57.239)                       │
│                                                     │
│  ┌──────────────┐    ┌────────────────┐            │
│  │ OpenClaw     │    │ Webhook Server │            │
│  │ Gateway      │◄───┤ (port 8401)    │            │
│  │ (port 18789) │    └────────────────┘            │
│  └──────────────┘            ▲                      │
│         │                    │                      │
│         │                    │ HTTPS                │
│         │                    │                      │
│  ┌──────▼──────┐    ┌────────┴────────┐            │
│  │ Forge Agent │    │ Monitor Service │            │
│  │ (main)      │    │ (port 8402)     │            │
│  └─────────────┘    └─────────────────┘            │
│                                                     │
│  ┌─────────────────────────────────────┐           │
│  │ Dashboard API (port 8400)           │           │
│  └─────────────────────────────────────┘           │
└─────────────────────────────────────────────────────┘
         │                    │
         │ Slack              │ Linear
         │ WebSocket          │ Webhook
         ▼                    ▼
┌────────────────┐    ┌──────────────────┐
│ Slack          │    │ Linear           │
│ (bot token)    │    │ (source of truth)│
└────────────────┘    └──────────────────┘
         │                    │
         └──────────┬─────────┘
                    ▼
            ┌──────────────┐
            │ You (Mitch)  │
            └──────────────┘
```

### Data Flow

**1. Prompt Submission:**
```
You → Slack → Forge Agent (OpenClaw)
```

**2. Ticket Creation:**
```
Forge Agent → Linear API → Creates ticket → Linear webhook fires
```

**3. Agent Spawning:**
```
Linear webhook → Forge webhook server → OpenClaw sessions_spawn → Agent runs
```

**4. Pipeline Progression:**
```
Agent completes → Updates Linear → Linear webhook fires → Next agent spawns
```

## Components

### 1. OpenClaw Gateway

- **Port:** 18789 (loopback)
- **Purpose:** OpenClaw runtime, manages all agent sessions
- **Service:** `openclaw-gateway.service` (systemd user service)
- **Config:** `~/.openclaw/openclaw.json`
- **Logs:** `/tmp/openclaw/openclaw-YYYY-MM-DD.log`

### 2. Forge Agent (main)

- **Location:** `~/.openclaw/agents/main/`
- **Workspace:** `~/.openclaw/workspace/`
- **Model:** Claude Opus 4
- **Job:** Validates prompts, creates Linear tickets
- **Does NOT:** Orchestrate, spawn pipeline agents (webhook does that)

### 3. Webhook Server

- **Port:** 8401
- **File:** `/home/ubuntu/.openclaw/workspace/scripts/forge-webhook/server.js`
- **Service:** `forge-webhook.service` (systemd)
- **Purpose:** Receives Linear webhook events, spawns agents
- **Cloudflare Tunnel:** HTTPS endpoint (Linear requires HTTPS)
- **Logs:** `journalctl -u forge-webhook`

**State → Agent Mapping:**
| Linear State | Agent Spawned |
|--------------|---------------|
| Todo         | coder         |
| In Review    | reviewer      |
| Testing      | tester        |
| Ready to Merge | merger      |

### 4. Pipeline Agents

Six registered agents (not always running):

| Agent | Model | Job |
|-------|-------|-----|
| gatekeeper | Sonnet 4.5 | Validate prompt quality |
| planner | Opus 4 | Break prompts into tickets |
| coder | Opus 4 | Implement code via Claude Code CLI |
| tester | Opus 4 | Run tests, verify acceptance criteria |
| reviewer | Opus 4 | Code review (quality, security, correctness) |
| merger | Sonnet 4.5 | Merge PRs via gh CLI |

**Location:** `~/.openclaw/agents/{agent-name}/`  
**Instructions:** `~/.openclaw/agents/{agent-name}/AGENTS.md`

### 5. Monitor Service

- **Port:** 8402
- **File:** `/home/ubuntu/.openclaw/workspace/scripts/forge-webhook/forge-monitor.js`
- **Service:** `forge-monitor.service` (systemd)
- **Purpose:** Detects stuck tickets, auto-retries, sends Slack alerts
- **Schedule:** Runs every 10 minutes
- **Health:** `curl http://localhost:8402/health`

**Detection:**
- Agent marked "complete" but ticket state unchanged for >15 min
- Ticket in "In Progress" for >30 min with no agent activity
- Webhook server unreachable for >5 min

**Actions:**
- Auto-retry (up to 3x, exponential backoff)
- Circuit breaker (3 failures → blocked state)
- Slack DM to Mitch with details

### 6. Dashboard API

- **Port:** 8400
- **File:** `/home/ubuntu/.openclaw/workspace/scripts/forge-api.js`
- **Service:** `forge-api.service` (systemd)
- **Purpose:** Serves data to Amplify dashboard

**Endpoints:**
- `GET /api/tickets` - All Linear issues
- `GET /api/tickets/:id` - Issue details
- `GET /api/agents` - Live sessions + agent history
- `GET /api/events` - Activity log (forge-db)
- `GET /api/costs` - Cost tracking
- `GET /api/health` - System health

**Data Sources:**
- Linear GraphQL API (tickets)
- forge-db MySQL (events, costs, prompts)
- OpenClaw sessions API (live agents)

### 7. Dashboard (Amplify)

- **URL:** https://master.d2hfx97ksa6sh1.amplifyapp.com
- **Repo:** sentineldev-bot/forge-dashboard
- **Stack:** Next.js 14, Tailwind CSS
- **Pages:** Dashboard, Tickets, Projects
- **Real-time:** Server-Sent Events (SSE) for live updates

### 8. Database (forge-db)

- **Type:** MySQL 8.0 on RDS
- **Endpoint:** forge-db.cn8sukokohju.eu-west-2.rds.amazonaws.com:3306
- **Purpose:** Operational logs ONLY (not tickets)

**Tables:**
- `prompts` - User prompt submissions
- `events` - Activity log (agent spawned, ticket moved, etc.)
- `cost_tracking` - API usage and costs
- `agent_memory` - Agent context between retries
- `conversations` - Slack conversation threads
- `retry_tracking` - Automatic retry state

**CLI:** `bash ~/.openclaw/workspace/scripts/forge-db.sh <command>`

### 9. Linear (Source of Truth)

- **Team:** sentinelopenclaw (eeb156c8-dbb4-4757-a2e0-02adbdd11545)
- **API Key:** Stored in `~/.openclaw/.env`
- **CLI:** `bash ~/.openclaw/workspace/scripts/linear-sync.sh <command>`
- **Webhook:** Points to Cloudflare tunnel URL

**Workflow States:**
- Backlog
- Todo
- In Progress
- In Review
- Testing
- Ready to Merge
- Done
- Canceled

## Network Architecture

```
Internet → Cloudflare Tunnel (HTTPS)
              ↓
         Forge EC2 (13.40.57.239)
              ↓
    ┌─────────┴─────────┐
    │                   │
Port 8401          Port 8400
Webhook            Dashboard API
              ↓
         Port 18789
    OpenClaw Gateway
```

**Firewall:**
- SSH (22) - Open
- HTTPS (443) - Open
- 8400, 8401, 8402, 18789 - Localhost only
- Cloudflare tunnel handles external HTTPS

## Security

**Credentials:**
- Slack bot token: Environment variable
- Linear API key: `~/.openclaw/.env`
- GitHub PAT: `~/.openclaw/.env`
- AWS credentials: `~/.aws/credentials`
- OpenAI/Anthropic keys: `~/.openclaw/.env`

**Access:**
- SSH key: `~/.ssh/forge-admin-key.pem` (Mitch controls)
- Cortex has NO access (after handoff)
- Federation has NO connection

## Isolation

Forge is **completely isolated** from the Cortex federation:
- Separate EC2 instance
- Separate OpenClaw installation
- Separate databases
- Separate Slack bot
- No message bus connection
- No shared credentials

**Independence:** If Cortex/federation goes down, Forge continues working.

## Failure Modes

1. **Agent fails** → Retry (up to 3x) → Circuit breaker → Slack alert
2. **Webhook server down** → Monitor detects → Slack alert
3. **OpenClaw gateway crash** → systemd auto-restarts
4. **Linear API down** → Webhook queues events, retries
5. **Ticket stuck** → Monitor detects → Auto-retry → Slack alert if blocked

## Scaling

**Current:** Single EC2 (t3.xlarge, 4 vCPU, 16GB RAM)
**Capacity:** ~10-20 concurrent tickets
**Bottleneck:** Agent spawning (OpenClaw sessions limit)

**To scale:**
- Increase EC2 instance size
- Or: Multi-instance federation (future)

## Monitoring

**Real-time:**
- Dashboard: https://master.d2hfx97ksa6sh1.amplifyapp.com
- Health: `curl http://localhost:8402/health`
- Slack: Automatic DM alerts

**Logs:**
- OpenClaw: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`
- Webhook: `journalctl -u forge-webhook`
- Monitor: `journalctl -u forge-monitor`
- API: `journalctl -u forge-api`

## Deployment

**Infrastructure:** Managed manually (EC2, RDS, Amplify)
**Code:** Git repos + Amplify auto-deploy
**Config:** Files in `~/.openclaw/workspace/`

See [[Deployment]] for full procedures.
