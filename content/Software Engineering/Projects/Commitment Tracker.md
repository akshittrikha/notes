## Overview
A CLI tool that logs intentions ("I'm going to finish X by Friday") and holds you accountable at the deadline. Claude evaluates your update and decides if it actually counts as done.

## Stack
- **Backend**: SQLite for persistence, cron job for deadline checks
- **AI layer**: Claude API with tool use + structured output
- **Interface**: CLI (can evolve to a Slack bot or TUI)

## What you'll learn
- Claude tool use and structured JSON output
- Writing a persistent backend from scratch
- Cron scheduling and background job orchestration

## Core features
- [ ] CLI to log a commitment with a deadline
- [ ] Store commitments in SQLite
- [ ] Cron job that triggers at deadline and prompts for an update
- [ ] Claude evaluates the update and marks done/incomplete
- [ ] Weekly summary of completion rate
- [ ] Optional: Slack/email notification on deadline

## Notes
- Start with a simple SQLite schema: `id, commitment, deadline, status, update`
- Claude prompt should be strict — "I'll do it tomorrow" should not count as done
