# Destructive Commands Reference

Complete catalog of destructive commands organized by category. This supplements the main [SKILL.md](../SKILL.md) with specific commands, reversibility assessments, and recommended safeguards.

---

## File System

| Command | What It Destroys | Reversible? | Recommended Safeguard | Risk Level |
|---------|-----------------|-------------|----------------------|------------|
| `rm -rf directory/` | Entire directory tree and all contents | No | `cp -r directory/ directory-backup/` before deletion | Critical |
| `rm -rf /` or `rm -rf *` | Entire filesystem or current directory tree | No | Never run; no safeguard sufficient | Critical |
| `rm file` | Single file permanently | Partial | `cp file file.backup` or verify file is in git | Medium |
| `mv file /dev/null` | File contents permanently | No | `cp file file.backup` before move | High |
| `chmod 000 file` | All access permissions on file | Yes | `stat` to record current permissions first | Medium |
| `chmod -R 777 directory/` | Secure permissions on entire tree | Yes | Record current permissions with `find dir -printf '%m %p\n'` | High |
| `shred file` | File contents with overwrite; unrecoverable | No | Copy to separate storage before shredding | Critical |
| `mkfs /dev/sdX` | All data on the partition/device | No | Full disk image backup first | Critical |
| `dd if=/dev/zero of=/dev/sdX` | All data on the target device | No | Full disk image backup first | Critical |
| `truncate -s 0 file` | File contents (sets size to zero) | Partial | `cp file file.backup` | High |
| `> file` (redirect to empty) | File contents (overwrites with empty) | Partial | `cp file file.backup` | High |

---

## Git

| Command | What It Destroys | Reversible? | Recommended Safeguard | Risk Level |
|---------|-----------------|-------------|----------------------|------------|
| `git push --force` | Remote branch history; other contributors' work | Partial | `git branch backup-branch` before force push | Critical |
| `git push --force-with-lease` | Remote branch history (safer than `--force`) | Partial | `git branch backup-branch`; still requires confirmation | High |
| `git reset --hard HEAD~N` | All uncommitted changes and N commits from local history | Partial | `git stash` and `git branch backup` first; reflog available for 90 days | Critical |
| `git reset --hard origin/main` | All local commits and uncommitted changes | Partial | `git stash` and `git branch backup` first | Critical |
| `git branch -D branch-name` | Local branch and its unique commits (if unmerged) | Partial | `git log --oneline -20 branch` to verify; reflog retains for 90 days | High |
| `git clean -fd` | All untracked files and directories | No | `git clean -fdn` (dry run) first to preview | High |
| `git clean -fdx` | All untracked and ignored files (build artifacts, .env, etc.) | No | `git clean -fdxn` (dry run) first; backup .env files | Critical |
| `git checkout .` | All unstaged changes in working directory | No | `git stash` first | High |
| `git restore .` | All unstaged changes in working directory | No | `git stash` first | High |
| `git rebase` (on shared branch) | Commit history on shared branch | Partial | `git branch backup` before rebase; coordinate with team | High |
| `git stash drop` | Specific stash entry | Partial | Verify stash contents with `git stash show -p` first; reflog retains briefly | Medium |
| `git stash clear` | All stash entries | No | Review all stashes with `git stash list` first | High |

---

## Database

| Command | What It Destroys | Reversible? | Recommended Safeguard | Risk Level |
|---------|-----------------|-------------|----------------------|------------|
| `DROP TABLE tablename` | Entire table, schema, and all rows | No | `pg_dump -t tablename` or `mysqldump` before drop | Critical |
| `DROP DATABASE dbname` | Entire database and all tables | No | Full database dump before drop | Critical |
| `TRUNCATE TABLE tablename` | All rows in table (keeps schema) | No | `pg_dump -t tablename` before truncate | Critical |
| `DELETE FROM table` (no WHERE) | All rows in table | No | `pg_dump -t tablename` before delete | Critical |
| `DELETE FROM table WHERE ...` (broad condition) | Potentially many rows | Partial | Verify row count with `SELECT COUNT(*)` using same WHERE clause first | High |
| `UPDATE table SET col = val` (no WHERE) | Original values in every row of column | Partial | Backup table or verify with `SELECT` first | Critical |
| `ALTER TABLE DROP COLUMN` | Column and all its data | No | `pg_dump -t tablename` before altering | High |
| `ALTER TABLE ... CASCADE` | Column and all dependent objects | No | Review dependencies; full backup first | Critical |
| Destructive migration (down migration) | Schema changes and potentially data | Partial | Full database dump before running migration | High |
| `GRANT ALL ON *.* TO user` | Database access controls | Yes | Document current grants with `SHOW GRANTS` first | High |
| `REVOKE ALL PRIVILEGES` | User's database access | Yes | Document current grants before revoking | Medium |

---

## Infrastructure

| Command | What It Destroys | Reversible? | Recommended Safeguard | Risk Level |
|---------|-----------------|-------------|----------------------|------------|
| `terraform destroy` | All infrastructure managed by the Terraform state | Partial | `terraform plan -destroy` to review; `terraform state pull` to backup state | Critical |
| `terraform apply` (destructive plan) | Resources marked for replacement or deletion | Partial | Always `terraform plan` first; review changes | High |
| `kubectl delete namespace` | All resources in the Kubernetes namespace | Partial | `kubectl get all -n ns -o yaml > backup.yaml` | Critical |
| `kubectl delete deployment` | Running deployment and all its pods | Partial | `kubectl get deployment name -o yaml > backup.yaml` | High |
| `kubectl delete pvc` | Persistent volume claim and potentially its data | No | Backup the persistent volume data first | Critical |
| `docker rm -f container` | Running container and its state | Partial | `docker commit container backup` for state; volumes persist | High |
| `docker volume rm` | Docker volume and all its data | No | `docker run --rm -v vol:/data -v $(pwd):/backup alpine tar czf /backup/vol.tar.gz /data` | Critical |
| `docker system prune -a` | All stopped containers, unused images, networks | Partial | Review with `docker system df` first | High |
| `kill -9 PID` | Process without graceful shutdown; potential data corruption | Partial | Use `kill PID` (SIGTERM) first; allow graceful shutdown | High |
| `systemctl stop service` | Running service (may cause downtime) | Yes | Verify no active connections; have restart plan | Medium |
| `helm uninstall release` | Helm-managed Kubernetes resources | Partial | `helm get all release > backup.yaml` | High |

---

## API/Services

| Command | What It Destroys | Reversible? | Recommended Safeguard | Risk Level |
|---------|-----------------|-------------|----------------------|------------|
| `DELETE /api/resource/:id` | The specified resource | Partial | `GET /api/resource/:id` to export before delete | High |
| `DELETE /api/collection` (bulk) | Multiple resources at once | Partial | Export full collection via GET before delete | Critical |
| Revoking API keys/tokens | Active integrations using those credentials | Partial | Verify no active services depend on the key; generate replacement first | Critical |
| Revoking OAuth tokens | User sessions and authorized applications | Partial | Notify affected users; generate new tokens first | High |
| Closing/archiving accounts | User access and associated data | Partial | Export account data before closing; verify archive is retrievable | Critical |
| Disabling webhooks | Active integrations expecting callbacks | Yes | Document current webhook configuration | Medium |
| Rotating secrets without coordination | Services using the old secret | Partial | Update all dependent services before rotation | High |
| Publishing npm package (overwrite) | Previous package version behavior for consumers | No | `npm pack` to review; use `--dry-run` first | High |

---

## Package Management

| Command | What It Destroys | Reversible? | Recommended Safeguard | Risk Level |
|---------|-----------------|-------------|----------------------|------------|
| `npm cache clean --force` | Local npm cache | Yes | Generally safe but slows subsequent installs | Medium |
| `npm uninstall package` | Package and its lockfile entry | Yes | `cp package-lock.json package-lock.json.backup` | Medium |
| `pip uninstall package` | Installed Python package | Yes | `pip freeze > requirements.backup.txt` | Medium |
| `cargo remove crate` | Crate dependency and lockfile entry | Yes | `cp Cargo.lock Cargo.lock.backup` | Medium |
| Major version upgrade (any) | Compatibility with current code | Partial | Run full test suite before and after; backup lockfile | High |
| `rm -rf node_modules && npm install` | Exact installed versions if lockfile is stale | Partial | Verify `package-lock.json` is committed and current | Medium |
| `npm publish` | Makes code publicly available (cannot fully unpublish after 72h) | No | `npm pack` and `npm publish --dry-run` first | Critical |
| `pip install --upgrade` (all) | Pinned versions across all dependencies | Partial | `pip freeze > requirements.backup.txt` first | High |
| `gem update` (all gems) | Pinned gem versions | Partial | `cp Gemfile.lock Gemfile.lock.backup` | High |
| `brew upgrade` (all) | Current versions of all Homebrew packages | Partial | `brew list --versions > brew-backup.txt` | Medium |

---

## Cross-Cutting Concerns

These patterns apply across all categories and indicate elevated risk regardless of the specific command.

| Pattern | Why It's Dangerous | Risk Level |
|---------|-------------------|------------|
| `--force` or `-f` flag on any command | Bypasses built-in safety confirmations | High |
| `--no-verify` on git operations | Skips pre-commit and pre-push hooks | High |
| Running commands as `root` or with `sudo` | Elevated privileges amplify blast radius | Critical |
| Piping `curl` output to `sh` or `bash` | Executes unreviewed remote code | Critical |
| Using `*` or glob patterns with destructive commands | Unpredictable scope; may match unexpected files | High |
| Operating on production without confirming environment | Wrong environment = catastrophic data loss | Critical |
| Chaining destructive commands with `&&` or `;` | Partial execution can leave system in broken state | High |
| Running destructive commands in loops | Multiplies blast radius per iteration | Critical |
