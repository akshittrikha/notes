## Overview
A cron job that scans your GitHub repos, identifies abandoned or stalled projects, and posts a weekly digest to surface things worth finishing.

## Stack
- **Data source**: GitHub REST API (repos, commits, issues, PRs)
- **Scheduler**: cron (local) or a cloud scheduler (GitHub Actions on schedule)
- **AI layer**: Claude summarizes stall reasons and suggests next steps
- **Output**: Weekly digest via email, Slack message, or a markdown file in vault

## What you'll learn
- GitHub API pagination and rate limiting
- Background job orchestration and scheduling
- LLM summarization over structured data

## Core features
- [ ] Fetch all personal repos via GitHub API
- [ ] Score each repo: last commit date, open issues, incomplete TODOs in README
- [ ] Flag repos as stalled (no commit in 30+ days) or abandoned (90+ days)
- [ ] Claude generates a one-line "why this stalled" guess + suggested next action
- [ ] Weekly digest output (start with markdown file, evolve to Slack/email)
- [ ] Filter out archived repos and forks

## Notes
- Use `GITHUB_TOKEN` env var — no OAuth needed for personal repos
- Stall heuristics: last commit date, open issues count, README has TODO/WIP markers
- Good first milestone: just print a sorted list of repos by last activity date
