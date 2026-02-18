---
name: managing-agent-secrets
description: Prevents AI agents from exposing secrets, API keys, and credentials. Enforces rules for handling .env files, environment variables, and sensitive configuration. Use when working with environment variables, API keys, database credentials, or any sensitive configuration that AI agents might accidentally log, commit, or expose.
---

# Managing Agent Secrets

## Overview

AI coding agents leak secrets. Not occasionally -- routinely. The mechanisms are predictable: an agent prints an API key for "debugging," commits a `.env` file, suggests a curl command with a bearer token inline, or writes a config file with real database credentials.

The problem is structural. Claude Code silently loads `.env` files into its context (documented by Knostic security research, January 2025). GitHub Copilot autocompletes credentials from training data patterns. Every AI coding tool that reads your project files has access to every secret in those files -- and no built-in restraint against repeating them.

The numbers confirm it. GitGuardian's 2024 report found a 40% increase in secrets exposure since AI coding tools became prevalent. Secrets in source code are the single most exploited attack vector in supply chain compromises. One leaked AWS key costs an average of $28,000 in unauthorized compute charges before detection.

This skill prevents secret leakage at the agent level. It defines what agents must never do with secrets, what they must always do instead, and how to detect violations before they reach a commit.

## The Rules

These are non-negotiable. No exceptions for convenience, speed, or "just testing."

1. **NEVER** include secrets in code, configs, commit history, or output.
2. **NEVER** log, print, or echo secret values -- not even for debugging.
3. **NEVER** pass secrets as CLI arguments (visible in process list via `ps`).
4. **NEVER** create files containing real credentials (even "temporary" ones).
5. **ALWAYS** use `.env` files with restrictive permissions (`chmod 600`).
6. **ALWAYS** verify `.gitignore` includes `.env*` before ANY commit.
7. **ALWAYS** reference secrets via environment variable names, not values.
8. **ALWAYS** use placeholder values in examples and documentation.

## Pre-Action Checklist

Before ANY operation involving configuration, environment, or credentials, run every check in this table. Do not skip checks. Do not assume prior checks still hold.

| Check | How |
|-------|-----|
| `.gitignore` includes `.env*`? | `grep -q '\.env' .gitignore` |
| Secrets loaded from env, not hardcoded? | Search for patterns: `API_KEY=`, `password=`, `token=` in source |
| Will this command expose secrets in output? | Check if command prints env vars or config values |
| Will this file be committed? Contains secrets? | `git diff --cached` -- review every line |
| Are example files using placeholder values? | Check `.env.example` for real values vs descriptive placeholders |
| `.env` file permissions restrictive? | `stat -f '%Lp' .env` should show `600` (macOS) or `stat -c '%a' .env` (Linux) |

## Common Agent Mistakes

These are the patterns AI agents produce most frequently. Each one is a secret leak waiting to happen.

### 1. Printing env vars for "debugging"

BAD:
```javascript
console.log(process.env.API_KEY);
console.log("Config:", JSON.stringify(process.env));
```

GOOD:
```javascript
console.log("API_KEY:", process.env.API_KEY ? "SET" : "UNSET");
console.log("Config keys loaded:", Object.keys(process.env).filter(k => k.startsWith("APP_")).length);
```

### 2. Creating config files with inline credentials

BAD:
```yaml
# database.yml
production:
  password: s3cretPassw0rd!
  host: prod-db.internal.company.com
```

GOOD:
```yaml
# database.yml
production:
  password: ${DATABASE_PASSWORD}
  host: ${DATABASE_HOST}
```

### 3. Suggesting curl commands with API keys inline

BAD:
```bash
curl -H "Authorization: Bearer sk-abc123def456ghi789" https://api.example.com/data
```

GOOD:
```bash
curl -H "Authorization: Bearer $API_KEY" https://api.example.com/data
```

### 4. Committing .env.example with real values

BAD:
```bash
cp .env .env.example  # copies real credentials as "examples"
```

GOOD:
```env
# .env.example -- all values are placeholders
API_KEY=your-api-key-here
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
SECRET_KEY=generate-a-random-string-here
```

### 5. Using secrets directly in docker-compose.yml

BAD:
```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD: myRealPassword123
```

GOOD:
```yaml
services:
  db:
    env_file: .env
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

### 6. Including secrets in error messages or logs

BAD:
```javascript
throw new Error(`Auth failed with key: ${apiKey}`);
logger.error(`Connection failed: ${connectionString}`);
```

GOOD:
```javascript
throw new Error("Auth failed -- check API_KEY env var");
logger.error(`Connection failed to ${hostname}:${port} (credentials redacted)`);
```

### 7. Hardcoding tokens in test fixtures

BAD:
```javascript
const testConfig = {
  apiKey: "sk-live-abc123def456",  // "test" key that works in staging
  dbPassword: "staging-pass-789"
};
```

GOOD:
```javascript
const testConfig = {
  apiKey: "sk-test-fake-000000",  // non-functional placeholder
  dbPassword: "fake-password"
};
// For integration tests, load from env:
// const apiKey = process.env.TEST_API_KEY;
```

### 8. Exposing secrets via process arguments

BAD:
```bash
python script.py --api-key=sk-real-key-here
# Anyone running `ps aux` can see this
```

GOOD:
```bash
export API_KEY="sk-real-key-here"
python script.py
# Or inline for a single command:
API_KEY="sk-real-key-here" python script.py
```

## Safe Patterns

Quick reference for correct secret handling by language and context.

| Context | Pattern |
|---------|---------|
| **Node.js** | `dotenv` package + `.env` file + `.gitignore` entry |
| **Python** | `python-dotenv` or `os.environ.get("KEY")` with default handling |
| **Docker** | `env_file` directive in compose, or Docker secrets for Swarm/K8s |
| **CI/CD** | Repository secrets (GitHub Actions `secrets.*`, GitLab CI variables) |
| **Production** | Secret manager: AWS Secrets Manager, HashiCorp Vault, 1Password CLI |
| **Shell scripts** | Source from `.env` file: `. ./.env` or `set -a; source .env; set +a` |
| **Terraform** | `TF_VAR_*` environment variables, never in `.tf` files |

## Detection Commands

Run these before every commit. Each targets a specific class of secret leak.

### Hardcoded API keys

```bash
rg -in '(api[_-]?key|secret[_-]?key|access[_-]?key)\s*[=:]\s*["\x27][A-Za-z0-9+/=_-]{16,}' --glob '!{*.lock,node_modules/**,dist/**,.git/**}'
```

### Hardcoded passwords

```bash
rg -in '(password|passwd|pwd)\s*[=:]\s*["\x27][^"\x27]{4,}' --glob '!{*.lock,node_modules/**,dist/**,.git/**,.env}'
```

### Connection strings with embedded credentials

```bash
rg -n '(postgresql|mysql|mongodb|redis)://[^:\s]+:[^@\s]+@' --glob '!{*.lock,node_modules/**,dist/**,.git/**,.env}'
```

### Base64-encoded secrets (common obfuscation)

```bash
rg -n '["\x27][A-Za-z0-9+/]{40,}={0,2}["\x27]' --glob '*.{js,ts,py,rb,go,java,yaml,yml,json}'
```

### Private keys

```bash
rg -rn '-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----' --glob '!{*.lock,node_modules/**,dist/**,.git/**}'
```

### AWS-specific patterns

```bash
rg -n '(AKIA[0-9A-Z]{16}|aws[_-]?secret[_-]?access[_-]?key\s*[=:])' --glob '!{*.lock,node_modules/**,dist/**,.git/**}'
```

## Emergency Response

If secrets were already committed, every minute counts. Follow these steps in order.

1. **Rotate the credential IMMEDIATELY.** Do this before anything else. The secret is compromised the instant it was pushed. Generate a new key, token, or password and update all services that use it.

2. **Remove from git history.** A new commit that deletes the file is not enough -- the secret remains in history.
   ```bash
   # Using BFG Repo-Cleaner (preferred -- faster and simpler)
   bfg --delete-files .env
   bfg --replace-text patterns.txt  # file containing literal secrets to redact

   # Using git filter-repo (built-in alternative)
   git filter-repo --path .env --invert-paths
   ```

3. **Force push the cleaned history.** This is one of the few justified uses of force push.
   ```bash
   git push --force --all
   git push --force --tags
   ```

4. **Check access logs.** Review the compromised service's access logs for unauthorized usage between the time of the commit and rotation. For cloud providers, check billing dashboards for unexpected charges.

5. **Add to .gitignore.** Prevent recurrence.
   ```bash
   echo ".env*" >> .gitignore
   git add .gitignore && git commit -m "fix: add .env to gitignore"
   ```

6. **Notify affected parties.** If the secret provided access to user data or third-party services, follow your organization's incident response procedure.

## The Bottom Line

- Secrets in source code are the most common supply chain attack vector.
- AI agents have no innate understanding of secret sensitivity.
- Every rule in this skill exists because an agent violated it.
- Prevention is a checklist problem, not an intelligence problem.
- Run the detection commands. Trust the output, not the agent's assurance.
