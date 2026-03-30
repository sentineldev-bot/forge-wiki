# Forge Overview

## What is Forge?

Forge is an **autonomous development pipeline** built on OpenClaw. You send it a natural language prompt (like "build a timer app"), and it:

1. Validates the prompt
2. Creates tickets in Linear
3. Spawns AI agents to code, test, review, and merge
4. Reports back when done

**No human in the loop** (unless something breaks).

## The Vision

Software development should be as simple as describing what you want. Forge turns prompts into shipped code, automatically.

## Core Principles

1. **Linear as Source of Truth** - All tickets live in Linear, not internal databases
2. **1 Ticket = 1 Agent Sequence** - Webhook-driven, automatic agent spawning
3. **Fail Loud** - Silent failures become loud alerts (Slack DMs)
4. **Autonomous** - Runs without human intervention
5. **Observable** - Dashboard shows what's happening in real-time

## What Forge Does

```
You → Forge (Slack)
  ↓
Validate prompt (gatekeeper logic)
  ↓
Create Linear tickets (planner logic)
  ↓
Webhook detects ticket → spawns coder
  ↓
Coder implements → moves ticket to "In Review"
  ↓
Webhook spawns reviewer
  ↓
Reviewer approves → moves to "Testing"
  ↓
Webhook spawns tester
  ↓
Tester passes → moves to "Ready to Merge"
  ↓
Webhook spawns merger
  ↓
Merger merges PR → moves to "Done"
  ↓
You get notified
```

## What Forge Doesn't Do

- Forge is NOT a chatbot (it's a pipeline)
- Forge does NOT orchestrate multi-agent workflows (webhook does that)
- Forge does NOT store tickets (Linear does)
- Forge does NOT have opinions about architecture (it follows your prompts)

## Key Components

1. **Forge Agent (main)** - Slack interface, validates prompts, creates tickets
2. **Webhook Server** - Receives Linear events, spawns agents
3. **Pipeline Agents** - coder, reviewer, tester, merger (spawned per ticket)
4. **Monitor Service** - Detects stuck tickets, auto-retries, sends alerts
5. **Dashboard** - Live visibility (agents, tickets, projects)
6. **Linear** - Source of truth for all tickets

## Technology Stack

- **Runtime:** OpenClaw 2026.3.28
- **Language:** Node.js 22, Bash
- **Infrastructure:** AWS EC2 (t3.xlarge), RDS MySQL
- **AI Models:** Claude Opus 4, Claude Sonnet 4.5
- **Ticketing:** Linear API
- **Dashboard:** Next.js 14, Amplify
- **Messaging:** Slack (socket mode)

## History

**March 24, 2026** - Initial build (as standalone Node.js service with SQS)  
**March 30, 2026** - Complete rebuild as OpenClaw-native agent  
**March 30, 2026** - Dashboard simplified to 3 pages  
**March 30, 2026** - Automatic retry + monitoring added  

See [[Changelog]] for full history.

## Quick Start

1. Send Forge a prompt in Slack: "Build a todo list app"
2. Forge validates and creates a Linear ticket
3. Agents automatically work through the pipeline
4. Watch progress on the dashboard: https://master.d2hfx97ksa6sh1.amplifyapp.com
5. Get notified when done

That's it.

## Learn More

- [[Architecture]] - How it's built
- [[Usage]] - Detailed usage guide
- [[Linear Integration]] - Ticket workflow
- [[Troubleshooting]] - When things break
