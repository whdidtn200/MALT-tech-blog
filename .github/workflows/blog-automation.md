---
name: Blog Automation Agent
on:
  pull_request:
    paths:
      - '_posts/**'
  push:
    branches:
      - main
---

# GitHub Agentic Workflow: Blog Post Auditor & FIFO Indexer

## Goal
Audit new blog posts for compliance (Korean-only, no PII) and update the main index file using the FIFO 13-link rule.

## Workflow
1. When a PR is opened or updated in `MALT-tech-blog`:
2. **Audit Post Content**:
   - Check modified files in `_posts/*.md`.
   - Ensure text is primarily Korean (technical terms in English are okay).
   - Ensure '조양수' (user's real name) is not present.
   - If violations found, comment on the PR asking for specific fixes.
3. **Update Index (on Push to main)**:
   - Run `python3 update_index.py`.
   - This script automatically maintains the 13-link limit on `whdidtn200.github.io/index.html` using FIFO logic (latest 13).
4. **Publish**:
   - Execute `./publish.sh` to push changes to the deployment repository.

## Instructions for Agent
- Use the `gh` CLI to interact with PRs.
- Use `python3 update_index.py` for index maintenance.
- Be polite but firm about the 'no real name' policy.
