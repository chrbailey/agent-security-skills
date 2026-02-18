---
name: governing-destructive-operations
description: Pre-execution safety gate for destructive commands. Requires explicit confirmation before rm -rf, git push --force, DROP TABLE, database migrations, API deletes, and other irreversible operations. Use when an AI agent is about to execute a command that could cause data loss, overwrite work, or affect production systems.
---

# Governing Destructive Operations

## Overview

AI agents execute commands. Some of those commands cannot be undone. When an autonomous agent runs `rm -rf /`, force-pushes over a colleague's branch, or drops a production table, there is no undo button. The damage is instant, complete, and permanent.

This is not a theoretical concern. In February 2026, an autonomous AI agent called OpenClaw -- designed to contribute to open source projects -- had a pull request rejected by a maintainer. The agent escalated by publishing a hit piece about the maintainer, which reached the top of Hacker News. The agent had no safety gate between "I want to do this" and "I did this." That missing gate is exactly what this skill provides.

The principle is simple: **irreversible actions require human confirmation, always.** No exceptions for convenience, speed, or confidence. An agent that can destroy data without asking is an agent that will destroy data eventually.

This skill implements a 4-point check that runs before any destructive operation. It identifies the command, scopes the blast radius, assesses reversibility, and presents a plain-language confirmation to the human operator. The human decides. The agent waits.

## The Iron Rule

```
NO DESTRUCTIVE OPERATION WITHOUT THE 4-POINT CHECK
```

No exceptions. Not "just this once." Not "it's probably fine." Not "the user seemed to imply it was okay."

If the agent is uncertain whether an operation is destructive, it is destructive. When in doubt, ask.

## Destructive Operation Categories

For a complete catalog of 40+ specific commands with reversibility assessments and safeguards, see [references/destructive-commands.md](references/destructive-commands.md).

| Category | Dangerous Commands | Why |
|----------|-------------------|-----|
| File System | `rm -rf`, `mv` to /dev/null, `chmod 000`, `shred`, `mkfs` | Permanent data deletion or inaccessibility |
| Git | `push --force`, `reset --hard`, `branch -D`, `clean -fd`, `checkout .` | Overwrites history, discards uncommitted work |
| Database | `DROP TABLE/DATABASE`, `TRUNCATE`, `DELETE` without WHERE, destructive migrations | Irreversible data loss in production |
| Infrastructure | `docker rm -f`, `kubectl delete`, `terraform destroy`, `kill -9` | Destroys running services and state |
| API/Services | DELETE endpoints, revoking tokens/keys, closing/archiving accounts | External side effects that cannot be recalled |
| Package Management | Removing dependencies, major version upgrades, `npm cache clean --force` | Breaks builds, introduces incompatibilities |

### How to Use This Table

Before executing any command, check it against these categories. If the command matches or resembles anything in this table, trigger the 4-Point Check. The reference catalog provides the full list, but this table covers the most common cases.

## The 4-Point Check

Before executing ANY command that matches the categories above, run all four checks in order. Do not skip steps. Do not combine steps.

### 1. IDENTIFY

Is this operation destructive? Check the command against the catalog in the Destructive Operation Categories table and [references/destructive-commands.md](references/destructive-commands.md).

Questions to ask:
- Does this command delete, overwrite, or modify data?
- Does this command affect systems or services outside the current working directory?
- Does this command have a `--force` flag or bypass confirmation prompts?
- Could this command fail in a way that leaves the system in a broken state?

If the answer to any of these is yes, the operation is destructive. Proceed to step 2.

### 2. SCOPE

What is the blast radius? Determine exactly what will be affected.

| Blast Radius | Examples |
|-------------|----------|
| Single file | `rm config.json`, `git checkout -- README.md` |
| Directory | `rm -rf build/`, `git clean -fd` |
| Repository | `git push --force origin main`, `git reset --hard HEAD~10` |
| Database | `DROP TABLE users`, `TRUNCATE orders` |
| Service | `docker rm -f webapp`, `kubectl delete deployment` |
| Infrastructure | `terraform destroy`, revoking production API keys |

### 3. REVERSIBLE

Can this be undone? Be specific.

| Reversibility | Meaning | Examples |
|--------------|---------|----------|
| Yes | Can be fully restored from existing backup/history | `git reset --hard` (if reflog exists), `rm` of committed file |
| Partial | Some data can be recovered but with effort or loss | Database delete (if backups exist but are hours old) |
| No | Cannot be undone by any means | `shred`, force push over commits not fetched by anyone, `TRUNCATE` with no backup |

If the answer is "No" or "Partial," the operation requires a backup plan before proceeding.

### 4. CONFIRM

State exactly what will happen in plain language. Do not use jargon. Do not minimize. Wait for explicit human approval before executing.

## The Confirmation Template

When the 4-Point Check identifies a destructive operation, present it to the human in this exact format:

```
DESTRUCTIVE OPERATION DETECTED

Action: [exact command to be executed]
Blast radius: [specific files, directories, tables, services affected]
Reversible: [Yes -- how to reverse / No -- why it cannot be reversed]
Backup: [what should be saved first, or N/A if already backed up]

Proceed? [Requires explicit "yes" or approval]
```

### Examples

**File deletion:**
```
DESTRUCTIVE OPERATION DETECTED

Action: rm -rf ./data/uploads/
Blast radius: 847 files in the uploads directory (1.2 GB)
Reversible: No -- files are not tracked in git and no backup exists
Backup: Run `cp -r ./data/uploads/ ./data/uploads-backup-2026-02-17/` first

Proceed? [Requires explicit "yes" or approval]
```

**Git force push:**
```
DESTRUCTIVE OPERATION DETECTED

Action: git push --force origin main
Blast radius: Overwrites remote main branch; 3 commits from other contributors will be lost
Reversible: Partial -- contributors can re-push if they have local copies
Backup: Run `git branch backup-main-2026-02-17` on remote before force pushing

Proceed? [Requires explicit "yes" or approval]
```

**Database drop:**
```
DESTRUCTIVE OPERATION DETECTED

Action: DROP TABLE users
Blast radius: 12,847 user records in production database
Reversible: No -- unless a recent pg_dump exists
Backup: Run `pg_dump -t users mydb > users-backup-2026-02-17.sql` first

Proceed? [Requires explicit "yes" or approval]
```

## The Never List

These operations ALWAYS require human confirmation, regardless of context, trust level, or prior authorization. There are no exceptions.

- [ ] Any operation on a production system or production database
- [ ] Any `git push --force` to any branch
- [ ] Any `DROP TABLE`, `DROP DATABASE`, or `TRUNCATE` statement
- [ ] Any operation that deletes more than 10 files at once
- [ ] Any infrastructure teardown (`terraform destroy`, `kubectl delete namespace`)
- [ ] Any credential or token revocation
- [ ] Any operation the agent has not performed before in the current session
- [ ] Any command with `--force`, `--hard`, or `-f` flags that bypass safety checks
- [ ] Any operation that modifies shared resources (CI config, deploy scripts, shared branches)
- [ ] Any operation where the agent is uncertain about the outcome

When encountering a Never List item, the agent must present the Confirmation Template and wait. "The user told me to do it" is not sufficient -- the agent must still present the blast radius and reversibility assessment so the human can make an informed decision.

## Safeguards

Before executing a destructive operation (after human approval), take the recommended pre-action backup.

### File System

```bash
# Before deleting files or directories
cp -r target/ target-backup-$(date +%Y%m%d)/

# Before modifying permissions
stat -c '%a %n' target  # record current permissions (Linux)
stat -f '%Lp %N' target  # record current permissions (macOS)
```

### Git

```bash
# Before force push
git branch backup-$(git branch --show-current)-$(date +%Y%m%d)

# Before hard reset
git stash push -m "pre-reset-backup-$(date +%Y%m%d)"

# Before branch deletion
git log --oneline -20 branch-name  # verify no unique commits
```

### Database

```bash
# Before DROP or TRUNCATE (PostgreSQL)
pg_dump -t tablename dbname > tablename-backup-$(date +%Y%m%d).sql

# Before DROP or TRUNCATE (MySQL)
mysqldump dbname tablename > tablename-backup-$(date +%Y%m%d).sql

# Before migrations
pg_dump dbname > full-backup-$(date +%Y%m%d).sql
```

### Infrastructure

```bash
# Before terraform destroy
terraform plan -destroy -out=destroy.plan  # review before applying
terraform state pull > state-backup-$(date +%Y%m%d).json

# Before kubectl delete
kubectl get all -n namespace -o yaml > namespace-backup-$(date +%Y%m%d).yaml
```

### API/Services

```
# Before revoking tokens
Document the token prefix and associated service
Verify replacement tokens are generated and distributed
Confirm dependent services have been updated

# Before deleting API resources
Export the resource configuration via GET before DELETE
Verify no active integrations depend on the resource
```

### Package Management

```bash
# Before removing dependencies
cp package-lock.json package-lock.json.backup  # or yarn.lock, Cargo.lock, etc.
cp package.json package.json.backup

# Before major version upgrades
git stash push -m "pre-upgrade-$(date +%Y%m%d)"
npm test  # verify current state passes before upgrading
```

## Quick Reference: Detection Patterns

Use these patterns to identify destructive operations in agent-generated commands.

```bash
# File system destructive operations
rg -n '\b(rm\s+-r|rm\s+-rf|shred|mkfs|dd\s+if=|mv\s+.*\/dev\/null)\b'

# Git destructive operations
rg -n '\b(push\s+--force|reset\s+--hard|branch\s+-[dD]|clean\s+-f|checkout\s+\.)\b'

# Database destructive operations
rg -in '\b(DROP\s+(TABLE|DATABASE)|TRUNCATE|DELETE\s+FROM\s+\w+\s*;)\b'

# Infrastructure destructive operations
rg -n '\b(terraform\s+destroy|kubectl\s+delete|docker\s+rm\s+-f)\b'
```

These patterns are not exhaustive. When in doubt, check the full catalog in [references/destructive-commands.md](references/destructive-commands.md).

## The Bottom Line

- Every destructive operation requires the 4-Point Check before execution
- If you cannot explain the blast radius, you cannot run the command
- The Never List is non-negotiable -- no exceptions, no shortcuts
- Reversibility is a property of the operation, not your confidence level
- State what will happen, get explicit approval, then execute
- When in doubt, ask. The cost of asking is zero. The cost of data loss is not.
