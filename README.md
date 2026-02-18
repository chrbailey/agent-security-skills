# Agent Security Skills

Pre-execution security and governance skills for AI coding agents. Works with Claude Code, Cursor, Codex, Windsurf, and any agent supporting the [Agent Skills](https://agentskills.io) standard.

## Tested Against 2.8 Million Lines of Real Code

We ran every detection pattern from these skills against 10 open-source repositories before publishing. Here's what we found.

**Corpus:** Next.js (1.2M lines), Strapi (467K), NocoDB (452K), cal.com (304K), Outline (245K), Juice Shop (90K), Payload (88K), Express (21K), NodeGoat (4K), Medusa (2K) -- **51,923 files total**.

### What Works

These patterns reliably catch real security issues in production code.

| Pattern | Findings | True Positive Rate | Why It Works |
|---------|----------|-------------------|--------------|
| **Security TODOs** | 204 | ~40% | Developers document what they skip. Pattern surfaces explicitly acknowledged auth gaps, missing validation, deferred fixes. Found 15+ missing auth TODOs in Medusa, missing password validation on cal.com account deletion. |
| **Hardcoded secrets** | 1,876 | ~35% | AI inlines credentials from training data. Found real API keys in NodeGoat, cleartext passwords in Juice Shop seed data, OAuth token logging in cal.com example apps. Noise source: test fixture passwords. |
| **SQL injection** | 179 | ~25% | Template literals in database queries. Caught textbook SQLi in Juice Shop login route, dynamic query construction in NocoDB column service. Noise source: migration files using template literals for table names. |
| **Console-log sensitive data** | 76 | ~25% | Logging secrets to stdout. Found OAuth access tokens and refresh tokens logged in cal.com, FCM push tokens logged in Next.js examples. |
| **Disabled tests** | 1,301 | ~60% informational | Not bugs, but coverage gaps. Found entire test suites disabled in NocoDB, skipped login/booking tests in cal.com, 889 skipped tests in Next.js. Useful for audit. |
| **Destructive DB operations** | 1,088 | ~70% informational | DROP TABLE, TRUNCATE, DELETE FROM. Most are legitimate migrations -- exactly what a governance system should flag for review. Found across all repos with databases. |

### What Doesn't Work (Yet)

These patterns are too noisy in their current form. We're documenting them honestly so you know the limitations.

| Pattern | Findings | TP Rate | Why It's Noisy | Fix Needed |
|---------|----------|---------|---------------|------------|
| **env-in-code** | 7,049 | ~2% | Matches `process.env.X` reads, which is the *correct* way to handle config. 47% of all findings, almost none actionable. | Narrow to fallback patterns where a default value bypasses the env var. |
| **Dangerous functions** | 2,282 | ~3% | `browser.evaluate()` in test frameworks inflates count massively. Next.js alone: 2,267 matches, nearly all test automation. The ~15 true positives (NodeGoat, Juice Shop) are critical though. | Exclude test automation API patterns. |
| **Command injection** | 161 | ~5% | Finds all shell execution calls, not just those with user input. Nearly all are build scripts with static strings. | Add taint analysis or narrow to calls containing request variables. |
| **Hardcoded URLs with credentials** | 53 | ~4% | Google Fonts URLs trigger the `user:pass@host` regex. 2 genuine finds buried under 46 false positives. | Exclude known API URL patterns. |

### What Works for Governance (Not Bug-Finding)

These patterns don't find bugs -- they flag things a human should review before approving.

| Pattern | Findings | What It Flags |
|---------|----------|--------------|
| **Unverified comments** | 268 | Hedging language in code: "this should", "probably", "seems to". Proxy for uncertainty. |
| **Suppressed errors** | 194 | Empty catch blocks. NocoDB has 12+ in production data services. |
| **Insecure defaults** | 138 | `cors()` with no config, `0.0.0.0` binding, `debug: true`. Strapi ships permissive CORS in its GraphQL plugin. |
| **Filesystem destructive** | 24 | `rm -rf` in scripts and CI. Not bugs, but worth flagging before execution. |

### Real Findings in Production Code

These are genuine issues we found in well-maintained, popular open-source projects.

**cal.com** -- Account deletion with no password check: `// TODO: First check password is part of input` in deleteMe.handler.ts. OAuth tokens logged to console in example apps.

**Strapi** -- `cors()` with no origin restriction in GraphQL plugin bootstrap. All project templates default to `0.0.0.0` host binding.

**NocoDB** -- 12+ empty catch blocks in data-table.service.ts silently swallowing errors. Dynamic SQL construction in columns.service.ts.

**Medusa** -- 15+ API code samples with `// TODO must be authenticated` comments acknowledging missing auth checks.

**Next.js** -- GitHub tokens embedded in git remote URLs in release scripts. 889 disabled tests.

**Juice Shop / NodeGoat** -- Intentionally vulnerable, every finding is a true positive. Validates that the patterns detect what they claim.

Full methodology, per-repo breakdowns, and raw data: **[validation/RESULTS.md](validation/RESULTS.md)**

## Skills

| Skill | What It Does |
|-------|-------------|
| [securing-ai-generated-code](skills/securing-ai-generated-code/) | Security review checklist for AI-written code -- OWASP patterns, injection, privilege escalation, secrets exposure |
| [governing-destructive-operations](skills/governing-destructive-operations/) | Pre-execution safety gate for dangerous commands -- rm, git push --force, DROP TABLE, API deletes |
| [managing-agent-secrets](skills/managing-agent-secrets/) | Prevent .env leaks, credential exposure, and API key mishandling by AI agents |
| [validating-agent-claims](skills/validating-agent-claims/) | Epistemic verification before acting on LLM assertions -- stop hallucinations from becoming actions |
| [preventing-agent-overreach](skills/preventing-agent-overreach/) | Scope guard -- stop agents from over-engineering, modifying wrong files, or exceeding task boundaries |

## Install

```bash
# All skills
npx skills add chrbailey/agent-security-skills

# Individual skill
npx skills add chrbailey/agent-security-skills/securing-ai-generated-code
```

## Why

- **322%** more privilege escalation paths in AI-generated code (Snyk 2024)
- **45%** of AI-generated code contains security vulnerabilities (Stanford/UIUC 2024)
- **96%** of developers don't fully trust AI code accuracy (Stack Overflow 2024)
- **Zero** security or governance skills in the skills.sh top 50

## Philosophy

These skills enforce **pre-execution governance** -- validating agent actions *before* they run, not auditing them after. They are:

- **Standalone** -- no external dependencies, no MCP server required
- **Cross-platform** -- works with any agent that reads SKILL.md files
- **Deterministic** -- checklists and rules, not probabilistic guardrails
- **Lightweight** -- each skill under 400 lines, loads in milliseconds

Built by the creator of [PromptSpeak](https://github.com/promptspeak/mcp-server), the pre-execution governance layer for AI agents.

## License

MIT
