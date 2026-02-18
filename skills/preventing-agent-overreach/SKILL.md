---
name: preventing-agent-overreach
description: Scope guard that prevents AI agents from exceeding task boundaries. Enforces the principle of minimal action — do exactly what was asked, nothing more. Use when delegating tasks to AI agents, when an agent starts modifying files beyond the original request, or when generated code includes unnecessary abstractions, features, or refactoring.
---

# Preventing Agent Overreach

## Overview

AI coding agents have an over-engineering problem. The symptoms are measurable:

| Metric | Value | Source |
|--------|-------|--------|
| Code duplication increase since AI tool adoption | 4x | GitClear 2024 |
| AI-generated PRs reverted for scope creep | 28% | Google internal, ICSE 2025 |
| Lines changed per task (AI vs human) | 3-5x more | GitClear 2024 |
| Developer time spent reviewing unnecessary changes | 23% | Stack Overflow 2024 |

The keystrokes got cheaper. The diffs got larger. Agents add features nobody asked for, refactor code that was not in scope, create abstractions for problems that do not exist yet, and "improve" files they were not told to touch. Each of these changes has a cost: review time, regression risk, merge conflicts, and cognitive load for every developer who reads the code next.

The root cause is structural. LLMs are trained on code that is already abstracted, already documented, already refactored. They reproduce those patterns reflexively, regardless of whether the current task calls for them. An agent asked to fix a typo will refactor the function. An agent asked to add a field will redesign the schema. An agent asked to update a dependency will rewrite the module.

This skill enforces a single principle: **do exactly what was asked, nothing more.**

## The Scope Contract

Before an agent touches any code, define the scope contract. Every task has one. If it is not written down, the agent will invent its own -- and it will be larger than yours.

### Template

```
TASK:        [Quote the original request verbatim]
FILES:       [List every file that should change]
NO-TOUCH:    [Everything else -- explicitly state it]
DONE-WHEN:   [Specific acceptance criteria]
```

### Example

```
TASK:        "Add a created_at timestamp to the User model"
FILES:       src/models/user.ts, src/migrations/003_add_created_at.ts
NO-TOUCH:    All other models, all routes, all tests not directly testing User
DONE-WHEN:   User model has created_at field, migration runs, existing tests pass
```

### Why This Works

The scope contract makes overreach visible. When the diff includes `src/routes/users.ts` and the contract says NO-TOUCH on all routes, the violation is objective. No judgment call required -- the file was not in scope, the change does not belong.

Without a scope contract, overreach is a matter of opinion. With one, it is a matter of fact.

## Overreach Patterns

These are the 8 most common ways agents exceed task boundaries. Each pattern looks helpful in isolation but adds cost that compounds across a codebase.

### 1. Error Handling for Impossible Cases

Agent adds defensive code for conditions that cannot occur given the program's control flow.

BAD:
```javascript
// Task: "Add a getName() method to User class"
getName() {
  if (!this) throw new Error("Instance is undefined");
  if (typeof this.name !== "string") throw new TypeError("Name must be a string");
  if (this.name === null || this.name === undefined) return "Unknown";
  return this.name ?? "Unknown";
}
```

GOOD:
```javascript
getName() {
  return this.name;
}
```

`this` is never undefined inside a method call. `name` is typed as `string` in the model. The constructor enforces it. Three of four checks are dead code.

### 2. Abstractions for One-Time Operations

Agent wraps a single-use operation in a reusable function, class, or module that will never be reused.

BAD:
```javascript
// Task: "Read config from config.json at startup"
class ConfigLoader {
  constructor(parser, validator, path) {
    this.parser = parser;
    this.validator = validator;
    this.path = path;
  }
  load() {
    const raw = fs.readFileSync(this.path, "utf-8");
    const parsed = this.parser.parse(raw);
    return this.validator.validate(parsed);
  }
}
const loader = new ConfigLoader(new JSONParser(), new ConfigValidator(), "config.json");
const config = loader.load();
```

GOOD:
```javascript
const config = JSON.parse(fs.readFileSync("config.json", "utf-8"));
```

One file. One format. One callsite. The abstraction adds three classes and zero value.

### 3. Refactoring Adjacent Code

Agent changes code near the task target that is not part of the request.

BAD:
```javascript
// Task: "Fix the off-by-one error in calculateTotal"
// Agent also renames variables, extracts helpers, and reformats the file

function computeOrderTotal(lineItems) {       // renamed from calculateTotal
  const subtotal = sumLineItems(lineItems);   // extracted helper
  const taxAmount = computeTax(subtotal);     // extracted helper
  return subtotal + taxAmount;
}

const sumLineItems = (items) => items.reduce((sum, item) => sum + item.price, 0);
const computeTax = (amount) => amount * 0.08;
```

GOOD:
```javascript
function calculateTotal(items) {
  const subtotal = items.reduce((sum, item) => sum + item.price, 0);
  const tax = subtotal * 0.08;
  return subtotal + tax;  // was: items.length + tax (off-by-one)
}
```

The fix is one line. Everything else is scope creep. Renaming, extracting, and reformatting generate noise in the diff, complicate review, and risk introducing new bugs.

### 4. Type Annotations on Unchanged Code

Agent adds or changes type annotations on code that was not part of the task.

BAD:
```typescript
// Task: "Add email field to User interface"
interface User {
  id: Readonly<string>;         // was: id: string -- agent "improved" it
  name: NonNullable<string>;    // was: name: string -- agent "improved" it
  email: string;                // the actual task
  createdAt: Readonly<Date>;    // was: createdAt: Date -- agent "improved" it
}
```

GOOD:
```typescript
interface User {
  id: string;
  name: string;
  email: string;          // added
  createdAt: Date;
}
```

Type changes on existing fields are separate tasks with separate review and separate testing.

### 5. Helper Utilities for Single-Use Operations

Agent creates utility functions, helper files, or shared modules for code used exactly once.

BAD:
```javascript
// Task: "Log the server start time"

// utils/formatters.js (new file)
export function formatTimestamp(date, format = "ISO") { /* 30 lines */ }
export function formatDuration(ms) { /* 15 lines */ }
export function formatBytes(bytes) { /* 10 lines */ }

// server.js
import { formatTimestamp } from "./utils/formatters.js";
console.log(`Server started at ${formatTimestamp(new Date())}`);
```

GOOD:
```javascript
console.log(`Server started at ${new Date().toISOString()}`);
```

One line. No new files. No utility module. No three unused exports.

### 6. Configuration for Hardcoded Values

Agent replaces working hardcoded values with configurable ones when nobody asked for configurability.

BAD:
```javascript
// Task: "Set request timeout to 30 seconds"
// config/timeouts.js (new file)
export default {
  request: parseInt(process.env.REQUEST_TIMEOUT_MS) || 30000,
  socket: parseInt(process.env.SOCKET_TIMEOUT_MS) || 60000,
  keepAlive: parseInt(process.env.KEEP_ALIVE_TIMEOUT_MS) || 5000,
};
```

GOOD:
```javascript
const TIMEOUT_MS = 30000;
```

If nobody asked to make timeouts configurable, do not make them configurable. One constant, one line, done.

### 7. Dependency Upgrades During Bug Fixes

Agent upgrades packages, changes lockfiles, or swaps libraries while fixing an unrelated bug.

BAD:
```diff
# Task: "Fix date parsing bug in report generator"
- "date-fns": "^2.30.0"
+ "date-fns": "^3.0.0"        # major version upgrade
+ "dayjs": "^1.11.0"          # added alternative lib
- const result = parse(input, "yyyy-MM-dd", new Date());
+ const result = dayjs(input).toDate();
```

GOOD:
```diff
- const result = parse(input, "yyyy-MM-dd", new Date());
+ const result = parse(input, "yyyy-MM-dd", new Date(), { locale: enUS });
```

The bug was a locale issue, not a library issue. Upgrading a major version and swapping libraries are separate tasks with separate risk profiles.

### 8. Comments and Docstrings on Untouched Code

Agent adds documentation to functions, classes, or modules that were not part of the task.

BAD:
```javascript
// Task: "Fix the return value of getUser"

/**
 * Retrieves a user by their unique identifier from the database.
 * @param {string} id - The unique user identifier (UUID v4 format)
 * @param {Object} [options] - Optional configuration
 * @param {boolean} [options.includeDeleted=false] - Include soft-deleted users
 * @returns {Promise<User|null>} The user object or null if not found
 * @throws {DatabaseError} If the database connection fails
 * @example
 *   const user = await getUser("123e4567-e89b-12d3-a456-426614174000");
 */
async function getUser(id) {
  return db.users.findOne({ id }) ?? null;  // fix: was returning undefined
}
```

GOOD:
```javascript
async function getUser(id) {
  return db.users.findOne({ id }) ?? null;  // fix: was returning undefined
}
```

The fix is `?? null`. The 10-line JSDoc block is unrelated work. If documentation is needed, it is a separate task.

## The Minimal Action Principle

Every task has a minimum viable change -- the smallest edit that satisfies the request. That is the target.

### Rules

1. **Make the smallest change that satisfies the request.** Not the smallest change you can imagine eventually needing.
2. **If in doubt, do less.** Removing unnecessary code from a PR is easier than debugging unnecessary code in production.
3. **Three similar lines are better than a premature abstraction.** Duplication is cheaper than the wrong abstraction. Refactor when the third callsite appears, not before.
4. **One behavior change per commit.** If a commit message needs "and" to describe what it does, it is two commits.
5. **Do not improve what you were not asked to improve.** Working code that is not in scope is off-limits, regardless of how much "better" you could make it.

### The Three-Question Test

Before making any change, ask:

1. **Did the user ask for this?** If no, do not do it.
2. **Does the task fail without this change?** If no, do not do it.
3. **Is this the simplest way to satisfy the request?** If no, simplify.

If a change does not pass all three questions, it does not belong in the diff.

## Scope Check

Run this checklist before every commit. If any item is checked, the commit contains overreach.

### Pre-Commit Scope Audit

- [ ] `git diff --stat` shows files NOT listed in the scope contract
- [ ] Changes exist in files not mentioned in the original request
- [ ] Net line count grew by more than 2x the minimum necessary change
- [ ] New functions, classes, or modules have exactly one caller
- [ ] New dependencies were added that the task did not require
- [ ] Existing function signatures changed when only behavior needed to change
- [ ] Test files were modified for code that was not in scope
- [ ] Comments or documentation were added to unchanged code

### Detection Commands

```bash
# Show changed files -- compare against scope contract
git diff --name-only

# Count lines changed -- is this proportional to the task?
git diff --stat

# Find new functions with one caller (JavaScript/TypeScript)
git diff --name-only | xargs rg 'function\s+\w+' -l | while read f; do
  rg -o 'function\s+(\w+)' -r '$1' "$f" | while read fn; do
    count=$(rg -l "\b$fn\b" --glob '*.{js,ts}' | wc -l)
    [ "$count" -le 1 ] && echo "SINGLE-USE: $fn in $f"
  done
done

# Find new files -- were new files part of the task?
git diff --name-only --diff-filter=A

# Find renamed identifiers -- was renaming part of the task?
git diff -M --diff-filter=R
```

### When the Audit Fails

If the scope audit reveals overreach:

1. **Identify** which changes are in scope and which are not.
2. **Separate** the in-scope changes from the overreach using `git add -p` or by editing.
3. **Commit** only the in-scope changes.
4. **Discard** the out-of-scope changes -- or file them as separate tasks for separate commits.

## When Expanding Scope Is Justified

Scope expansion is sometimes necessary. It is rarely as necessary as agents think it is.

### Legitimate Reasons to Expand

| Reason | Example | Required Action |
|--------|---------|-----------------|
| The task literally cannot work without the additional change | Adding a field requires a migration | Include in scope contract, document why |
| The change fixes a bug that the task would reintroduce | Fixing a race condition in code being modified | Flag it, get approval, separate commit |
| Security vulnerability in code being touched | SQL injection in the function being edited | Fix it, document it, separate commit |
| Build or test failure caused by the in-scope change | Type error in a dependent file after interface change | Minimal fix only -- do not refactor the dependent file |

### NOT Legitimate Reasons

- "While I'm in this file, I noticed..."
- "This would be more maintainable if..."
- "Best practice suggests..."
- "Future-proofing for when..."
- "It's a small improvement..."
- "The style was inconsistent..."

Each of these is a separate task. File it, do not do it now.

### The Expansion Protocol

When scope expansion is genuinely required:

1. **State** what additional change is needed and why.
2. **Justify** why the original task cannot be completed without it.
3. **Get approval** before making the change.
4. **Commit separately** from the original task.

If the justification is "it would be better," it is not required. Move on.

## The Bottom Line

- Every task has a minimum viable change. Make that change and stop.
- Scope contracts make overreach visible and objective.
- Agents default to over-engineering. Counter this with explicit constraints.
- Three similar lines are cheaper than the wrong abstraction.
- If it was not asked for, do not do it.
- When in doubt, do less.
