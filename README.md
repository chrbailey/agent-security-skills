# Agent Security Skills

Pre-execution security and governance skills for AI coding agents. Works with Claude Code, Cursor, Codex, Windsurf, and any agent supporting the [Agent Skills](https://agentskills.io) standard.

## Why

- **322%** more privilege escalation paths in AI-generated code
- **45%** of AI-generated code contains security vulnerabilities
- **96%** of developers don't fully trust AI code accuracy
- **Zero** security or governance skills in the skills.sh top 50

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

## Validated at Scale

Tested against **2.8 million lines of code** across 10 open-source repositories (Next.js, Strapi, cal.com, NocoDB, Outline, Payload, Medusa, Express, OWASP Juice Shop, OWASP NodeGoat).

- **14,900 findings** across 16 detection patterns
- **All 16 patterns** produced non-zero results -- no dead patterns
- True positives confirmed in production code from cal.com, Strapi, NocoDB, and Medusa
- Known-vulnerable repos (Juice Shop, NodeGoat) validate pattern accuracy

Full results: [validation/RESULTS.md](validation/RESULTS.md)

## Philosophy

These skills enforce **pre-execution governance** -- validating agent actions *before* they run, not auditing them after. They are:

- **Standalone** -- no external dependencies, no MCP server required
- **Cross-platform** -- works with any agent that reads SKILL.md files
- **Deterministic** -- checklists and rules, not probabilistic guardrails
- **Lightweight** -- each skill under 400 lines, loads in milliseconds

Built by the creator of [PromptSpeak](https://github.com/promptspeak/mcp-server), the pre-execution governance layer for AI agents.

## License

MIT
