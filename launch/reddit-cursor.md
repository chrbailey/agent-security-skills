# Reddit r/cursor

**Title:** 5 free security skills for Cursor (and other AI agents) -- tested against 2.8M lines of real code

**Body:**

Built a set of security and governance skills that work with Cursor, Claude Code, Codex, Windsurf, and anything supporting the Agent Skills standard.

Before publishing, I tested every detection pattern against 10 open-source repos (Next.js, Strapi, cal.com, NocoDB, etc.) -- 2.8M lines total. Found 14,900 flagged items, confirmed true positives in production code from major projects.

**The skills:**
- **securing-ai-generated-code** -- pre-commit checklist for OWASP patterns, injection, hardcoded secrets, missing validation
- **governing-destructive-operations** -- safety gate before rm -rf, force push, DROP TABLE
- **managing-agent-secrets** -- prevent .env leaks and credential exposure
- **validating-agent-claims** -- require evidence before acting on AI assertions
- **preventing-agent-overreach** -- scope guard to stop over-engineering

Each skill is a standalone SKILL.md under 400 lines. No dependencies. Just install and they load automatically.

The README includes what works AND what doesn't -- with false positive rates for every pattern. The "What Doesn't Work Yet" section is intentional.

Install: `npx skills add chrbailey/agent-security-skills`

Repo: https://github.com/chrbailey/agent-security-skills
