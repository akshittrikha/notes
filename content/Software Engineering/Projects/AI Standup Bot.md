## Overview
A bot that reads your git commits and TODO list from the day, then writes your standup automatically. Run it each morning to get yesterday's summary + today's plan.

## Stack
- **Data sources**: `git log` (local repos), TODO file or Obsidian daily note
- **AI layer**: Claude API for summarization and formatting
- **Interface**: CLI script, optionally posts to Slack

## What you'll learn
- LLM chaining: multiple inputs → structured output
- Git internals (log parsing, diff stats)
- Slack API / webhook integration

## Core features
- [ ] Parse `git log --since=yesterday` across configured local repos
- [ ] Pull TODOs from Obsidian daily note or a TODO file
- [ ] Claude generates standup in format: Yesterday / Today / Blockers
- [ ] CLI: `standup` command outputs to terminal
- [ ] Optional: post to a Slack channel via webhook
- [ ] Optional: save standup to Obsidian daily note automatically

## Notes
- Start with a single hardcoded repo path, make it configurable later
- Claude prompt should emphasize conciseness — standups should be 3-5 bullets max
- Combine with [[Commitment Tracker]] later: surface overdue commitments in standup
