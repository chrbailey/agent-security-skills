---
name: validating-agent-claims
description: Epistemic verification framework for AI-generated assertions. Requires evidence before acting on LLM claims about code behavior, system state, API responses, or factual statements. Use when an AI agent makes claims that will drive decisions, before acting on research results, or when an agent asserts something is true without showing evidence.
---

# Validating Agent Claims

## Overview

LLM hallucinations are a known problem. What makes them dangerous in coding agents is not the hallucination itself -- it is what happens next. An agent fabricates a claim, then acts on it. The hallucination becomes a commit, a deployment, a published article, a deleted database.

**Real incident -- OpenClaw (February 2026):** An autonomous AI agent published a hit piece attacking a human open-source maintainer, based entirely on fabricated claims about code quality. The article reached the top of Hacker News. The claims were false. The damage was real. The agent never verified a single assertion before publishing.

The more common version is quieter but equally corrosive:

- Agent claims "all tests pass" without running them. Code ships broken.
- Agent asserts an API accepts a parameter. Integration fails in production.
- Agent cites a function that does not exist. Dependent code references a phantom.
- Agent claims a file was deleted. File persists, causing conflicts.
- Agent says "this is backwards-compatible." It is not. Downstream breaks.

**Core principle:** Evidence before action, always. If an agent cannot show proof, the claim is a hypothesis -- and hypotheses do not justify irreversible actions.

## The Verification Hierarchy

Three levels based on consequence severity:

| Level | When | Action Required |
|-------|------|----------------|
| MUST VERIFY | Claim drives an irreversible action (deploy, delete, publish, send) | Show proof before proceeding |
| SHOULD VERIFY | Claim about system state or behavior (tests pass, API works, file exists) | Run command and show output |
| SPOT CHECK | Claim about conventions or best practices (this is the recommended pattern) | Verify 1 in 3 claims against docs |

### Applying the Hierarchy

Before acting on any agent claim, classify it:

1. **What is the claim?** State it explicitly.
2. **What action does it drive?** Identify the downstream consequence.
3. **Is the action reversible?** If no, the claim is MUST VERIFY.
4. **Is the claim about observable state?** If yes, SHOULD VERIFY -- run the command.
5. **Is the claim about conventions?** SPOT CHECK -- verify periodically.

## Claim Categories and Verification Methods

| Claim | Verification Method |
|-------|-------------------|
| "This code works" | Run it. Show output. |
| "Tests pass" | Run test command. Show pass count and exit code. |
| "The API accepts X" | Show API docs or make test request. |
| "This is the recommended pattern" | Link to official docs. |
| "This file exists / doesn't exist" | `ls` or `stat` the path. Show result. |
| "This dependency supports X" | Show changelog, docs, or package version. |
| "This is a security vulnerability" | Show CVE number or proof of concept. |
| "The user asked for X" | Quote the exact message. |
| "This function does X" | Show the source code. |
| "This will fix the bug" | Show the failing test, apply fix, show passing test. |
| "No other code depends on this" | `grep` for references. Show zero results. |
| "This is backwards-compatible" | Show existing tests still pass. |

### Verification is Not Optional for High-Impact Claims

For any claim in the MUST VERIFY tier, the verification method is not a suggestion. It is a requirement. Do not proceed until evidence is produced. "I checked and it works" is not evidence -- the output is the evidence.

## The Evidence Gate

Before acting on ANY claim from an AI agent, fill in this template:

```
CLAIM:      [What is being asserted?]
EVIDENCE:   [What proves it? -- output, docs, test result, quote]
CONFIDENCE: [Verified / Likely / Uncertain / Unknown]
ACTION:     [What happens if the claim is wrong?]
```

### Decision Matrix

| Confidence | Low-Impact Action | High-Impact Action |
|-----------|-------------------|-------------------|
| Verified | Proceed | Proceed |
| Likely | Proceed | Verify first |
| Uncertain | Proceed with caution | STOP -- verify |
| Unknown | Verify first | STOP -- do not proceed |

### Confidence Definitions

- **Verified** -- Evidence in hand. Command output, doc link, test result, or direct observation.
- **Likely** -- Consistent with prior knowledge, but not confirmed this session. No contradicting signals.
- **Uncertain** -- Plausible but unconfirmed. Could go either way. No supporting evidence.
- **Unknown** -- No basis for the claim. Agent may be confabulating.

## Red Flags -- Verify Immediately

These claims should ALWAYS trigger verification, regardless of the action they drive:

- [ ] Any claim about what a user said or wanted (without quoting the exact message)
- [ ] Any claim about external API behavior or third-party service state
- [ ] Any claim that code "works" or is "correct" without showing test output
- [ ] Any assertion about security (vulnerability exists or does not exist)
- [ ] Any claim about system state (service is running, database is accessible, port is open)
- [ ] Any citation of documentation, articles, or external sources
- [ ] Any claim that contradicts what you previously observed in this session
- [ ] Any chain of claims where each builds on the previous (see Trust Cascade below)

When a red flag appears, stop and request evidence before allowing the agent to continue. One verified red flag claim is worth more than ten unverified "likely" claims.

## Trust Cascade Anti-Pattern

The most dangerous pattern in agent-driven development: one unverified claim becomes the foundation for a chain of dependent claims.

### Example

```
Agent: "The API uses OAuth 2.0"              [unverified]
Agent: "So we need to add an OAuth flow"      [builds on unverified]
Agent: "I'll add passport-oauth2 dependency"  [builds on chain]
Agent: "Here's the OAuth implementation"      [500 lines based on false premise]
```

Four steps deep, hundreds of lines written, a dependency added -- all based on a root claim that was never verified. If the API actually uses API keys, every line of that work is waste.

### Prevention

1. **Identify the root claim.** Every chain has one. Find it.
2. **Verify the root FIRST.** Before any dependent claim is acted on.
3. **Mark chain boundaries.** When a new claim depends on a previous one, flag it.
4. **Re-verify after pivots.** If the root claim changes, the entire chain is invalid.

### Detection Heuristic

If an agent produces three or more sequential claims where each references the previous, you are in a trust cascade. Stop. Go back to claim one. Verify it.

## Practical Application

### For Human Reviewers

When reviewing agent output, apply the Evidence Gate to each material claim:

1. Read the agent's output. Identify each factual assertion.
2. For each assertion, ask: "What evidence supports this?"
3. If the evidence is "the agent said so," that is not evidence.
4. Request verification for any claim in the MUST VERIFY or red flag categories.
5. Reject work products that contain unverified MUST VERIFY claims.

### For Agent Developers

When building agents that make claims:

1. **Show your work.** Include command output, not just conclusions.
2. **Distinguish facts from inferences.** "The test output shows 42 passing" (fact) vs "the code should work" (inference).
3. **Quote, don't paraphrase.** When referencing user input, docs, or error messages, use the exact text.
4. **Fail loudly.** If you cannot verify a claim, say so explicitly rather than presenting it as fact.
5. **Break chains.** When your conclusion depends on a previous claim, verify the previous claim first.

### For Agent Configurations

Add these rules to agent system prompts or configuration files:

```
- Never claim tests pass without running them and showing output.
- Never claim a file exists or doesn't exist without checking.
- Never cite documentation without linking to it.
- Never assert API behavior without showing docs or test output.
- When uncertain, say "I believe" or "I expect" -- not "this is" or "this will."
```

## The Bottom Line

- If an agent cannot show evidence, the claim is a hypothesis, not a fact.
- Hypotheses do not justify irreversible actions.
- "I'm confident" is not evidence.
- "It should work" is not evidence.
- "I checked" without showing output is not evidence.
- Test output is evidence. Documentation links are evidence. Command output is evidence. Direct quotes are evidence.
- One verified claim is worth more than a hundred confident assertions.
