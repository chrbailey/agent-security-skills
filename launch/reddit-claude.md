# Reddit r/ClaudeAI

**Title:** I tested security detection patterns against 2.8M lines of code and published the results -- 5 free skills for Claude Code

**Body:**

I built 5 security/governance skills for Claude Code (and other AI agents) and tested them at scale before releasing.

**The problem:** 45% of AI-generated code has vulnerabilities, privilege escalation is up 322%, and there are zero security skills in the skills.sh top 50.

**What I did:** Ran 16 detection patterns against Next.js, Strapi, cal.com, NocoDB, and 6 other repos. 2.8M lines of code. Found 14,900 things worth flagging.

**What works:**
- Security TODOs (~40% TP rate) -- catches missing auth checks, skipped validation
- Hardcoded secrets (~35% TP rate) -- catches API keys, leaked credentials
- SQL injection (~25% TP rate) -- catches template literals in queries
- Found real bugs in cal.com (account deletion with no password check), Strapi (cors with no origin restriction), NocoDB (12+ silent error suppression in data services)

**What doesn't work (yet):**
- env-in-code pattern is too noisy (matches correct process.env usage)
- Dangerous functions inflated by browser automation APIs in tests
- I document all of this honestly in the README

**The skills:**
1. securing-ai-generated-code -- OWASP patterns, injection, secrets
2. governing-destructive-operations -- safety gate for rm -rf, DROP TABLE, force push
3. managing-agent-secrets -- prevent .env leaks, credential exposure
4. validating-agent-claims -- stop hallucinations from becoming actions
5. preventing-agent-overreach -- scope guard, do what was asked and nothing more

Install: `npx skills add chrbailey/agent-security-skills`

Standalone SKILL.md files, no dependencies, under 400 lines each. Works with Claude Code, Cursor, Codex, Windsurf, Gemini CLI.

Repo: https://github.com/chrbailey/agent-security-skills

Full validation data with per-repo breakdowns in the repo. Happy to answer questions.
