# Hacker News

**Title:** Show HN: I tested 16 security patterns against 2.8M lines of code -- here's what catches real bugs

**URL:** https://github.com/chrbailey/agent-security-skills

**Text (for Show HN, appears below the link):**

AI coding agents generate insecure code at measurable rates: 45% of AI-generated code contains vulnerabilities (Stanford/UIUC 2024), privilege escalation paths are up 322% (Snyk 2024), and secrets exposure is up 40% (GitGuardian 2024).

There are 63,000+ skills on skills.sh. Zero are for security or governance.

So I built 5 and tested them against real code before publishing. I ran 16 detection patterns against Next.js, Strapi, cal.com, NocoDB, Outline, Payload, Medusa, Express, and both OWASP vulnerable apps (Juice Shop, NodeGoat). 2.8M lines, 52K files, 10 repos.

Results:

- 14,900 total findings across all patterns
- Security TODOs had the highest signal (~40% true positive rate) -- developers document what they skip
- Found real issues in production code: missing password validation on cal.com's account deletion endpoint, cors() with no origin restriction in Strapi's GraphQL plugin, 12+ empty catch blocks in NocoDB's data services, 15+ "TODO must be authenticated" comments in Medusa's API
- 4 patterns are too noisy and I document why (env-in-code matches correct usage, dangerous-functions is inflated by browser automation APIs)

The "What Doesn't Work" section is intentional. Publishing false positive rates alongside true positive rates is the only honest way to release detection tooling.

The skills are standalone SKILL.md files -- no dependencies, no MCP server, works with Claude Code, Cursor, Codex, Windsurf, Gemini CLI, and anything that supports the Agent Skills standard. Each is under 400 lines.

Install: `npx skills add chrbailey/agent-security-skills`

Full validation methodology and per-repo breakdowns in the repo.
