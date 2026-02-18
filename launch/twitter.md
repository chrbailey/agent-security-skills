# Twitter/X

**Thread (4 posts):**

---

**Post 1:**

There are 63,000+ skills for AI coding agents.

Zero are for security.

So I built 5 and tested them against 2.8M lines of real code before publishing.

Here's what actually catches bugs (and what doesn't):

github.com/chrbailey/agent-security-skills

---

**Post 2:**

What works:
- Security TODOs (~40% true positive rate)
- Hardcoded secrets (~35%)
- SQL injection patterns (~25%)

Found real issues in cal.com, Strapi, NocoDB, and Medusa.

What doesn't work yet:
- env-in-code is too noisy (2% TP)
- Dangerous function detection inflated by test frameworks

I publish both.

---

**Post 3:**

The 5 skills:

1. securing-ai-generated-code
2. governing-destructive-operations
3. managing-agent-secrets
4. validating-agent-claims
5. preventing-agent-overreach

Standalone files, no dependencies, under 400 lines each.

Works with Claude Code, Cursor, Codex, Windsurf, Gemini CLI.

npx skills add chrbailey/agent-security-skills

---

**Post 4:**

Why publish false positive rates?

Because 45% of AI code has vulnerabilities and the tooling that claims to fix this never shows you the noise.

If your detection pattern has a 2% true positive rate, say so. Then fix it.

Validation data for all 16 patterns in the repo.
