---
name: securing-ai-generated-code
description: Reviews AI-generated code for security vulnerabilities before commit. Checks for injection flaws, privilege escalation, hardcoded secrets, insecure defaults, and missing input validation. Use when reviewing code written by AI coding agents, after code generation, or before committing AI-assisted changes.
---

# Securing AI-Generated Code

## Overview

AI coding tools generate insecure code at predictable, measurable rates:

| Metric | Value | Source |
|--------|-------|--------|
| Code containing vulnerabilities | 45% | Stanford/UIUC 2024 |
| Increase in privilege escalation paths | 322% | Snyk 2024 |
| Increase in secrets exposure | 40% | GitGuardian 2024 |
| Developers who don't fully trust AI output | 96% | Stack Overflow 2024 |

AI code fails differently than human code. Humans make mistakes from fatigue or ignorance. AI makes mistakes from pattern-matching training data without understanding security context. The result: AI-generated code consistently produces the same categories of vulnerability, making them detectable with deterministic checks.

This skill is the security gate between code generation and commit. Run it every time.

## The Pre-Commit Security Gate

Run this checklist on EVERY AI-generated or AI-modified file before committing.

### Mandatory Checks

- [ ] **Secrets scan** -- No API keys, tokens, passwords, or connection strings in source
- [ ] **Input validation** -- All data crossing system boundaries is validated (HTTP params, file uploads, DB results, env vars)
- [ ] **Auth/authz** -- Every endpoint, route, and API handler has authentication and authorization checks
- [ ] **Injection** -- All database queries use parameterized statements; no shell commands with string concatenation
- [ ] **Error handling** -- No stack traces in production responses; no sensitive data in error messages
- [ ] **Dependencies** -- No packages with known CVEs added; lockfile updated
- [ ] **Permissions** -- File permissions, CORS policy, CSP headers, and network bindings are restrictive by default

If ANY check fails, fix the code before committing. Do not commit with a TODO to fix later.

## AI-Specific Vulnerability Patterns

These are the 8 patterns AI coding tools produce most frequently. For extended examples in Python, Go, Java, and Ruby, see [references/vulnerability-patterns.md](references/vulnerability-patterns.md).

### 1. Hardcoded Secrets

AI inlines credentials from training data patterns instead of reading from environment.

BAD:
```javascript
const client = new S3Client({
  credentials: { accessKeyId: "AKIA...", secretAccessKey: "wJalr..." }
});
```

GOOD:
```javascript
const client = new S3Client({
  credentials: fromEnv()  // reads AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
});
```

### 2. Missing Input Validation

AI generates the happy path. It does not validate boundary conditions, type constraints, or malicious input.

BAD:
```javascript
app.post("/users", (req, res) => {
  const { email, age } = req.body;
  db.insert({ email, age });  // No validation
});
```

GOOD:
```javascript
app.post("/users", (req, res) => {
  const { email, age } = req.body;
  if (!isEmail(email)) return res.status(400).json({ error: "Invalid email" });
  if (!Number.isInteger(age) || age < 0 || age > 150) return res.status(400).json({ error: "Invalid age" });
  db.insert({ email, age });
});
```

### 3. SQL/NoSQL Injection

AI uses string interpolation for queries because training data is full of it.

BAD:
```javascript
const user = await db.query(`SELECT * FROM users WHERE id = '${req.params.id}'`);
```

GOOD:
```javascript
const user = await db.query("SELECT * FROM users WHERE id = $1", [req.params.id]);
```

### 4. Command Injection

AI passes user input to shell commands without sanitization.

BAD:
```javascript
app.get("/lookup", (req, res) => {
  execSync(`nslookup ${req.query.domain}`);
});
```

GOOD:
```javascript
app.get("/lookup", (req, res) => {
  if (!/^[a-zA-Z0-9.-]+$/.test(req.query.domain)) return res.status(400).send("Invalid domain");
  execFileSync("nslookup", [req.query.domain]);
});
```

### 5. Path Traversal

AI does not validate file paths from user input, allowing `../` to escape intended directories.

BAD:
```javascript
app.get("/files/:name", (req, res) => {
  res.sendFile(path.join("/uploads", req.params.name));
});
```

GOOD:
```javascript
app.get("/files/:name", (req, res) => {
  const safeName = path.basename(req.params.name);
  const fullPath = path.join("/uploads", safeName);
  if (!fullPath.startsWith("/uploads/")) return res.status(403).send("Forbidden");
  res.sendFile(fullPath);
});
```

### 6. Insecure Defaults

AI generates permissive configurations that work in development but are dangerous in production.

BAD:
```javascript
const app = express();
app.use(cors());  // origin: * -- allows any domain
app.listen(3000, "0.0.0.0");  // binds to all interfaces
```

GOOD:
```javascript
const app = express();
app.use(cors({ origin: process.env.ALLOWED_ORIGINS?.split(",") }));
app.listen(3000, "127.0.0.1");  // binds to localhost only
```

### 7. Missing Auth Checks

AI generates functional endpoints without authentication or authorization middleware.

BAD:
```javascript
app.delete("/api/users/:id", async (req, res) => {
  await db.query("DELETE FROM users WHERE id = $1", [req.params.id]);
  res.json({ deleted: true });
});
```

GOOD:
```javascript
app.delete("/api/users/:id", authenticate, authorize("admin"), async (req, res) => {
  await db.query("DELETE FROM users WHERE id = $1", [req.params.id]);
  res.json({ deleted: true });
});
```

### 8. Verbose Error Responses

AI returns full error objects including stack traces, SQL statements, and internal paths.

BAD:
```javascript
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack, query: err.sql });
});
```

GOOD:
```javascript
app.use((err, req, res, next) => {
  console.error(err);  // log full error server-side
  res.status(500).json({ error: "Internal server error" });
});
```

## The Review Process

Follow this order for every AI-generated changeset.

### Step 1: Automated Scan

Run the detection commands from the Quick Detection Commands section below. Fix anything they find before proceeding.

### Step 2: Pattern Match

For each changed file, check against all 8 vulnerability patterns above. AI-generated code clusters vulnerabilities -- if you find one pattern, check harder for the others.

### Step 3: Boundary Validation Audit

Identify every point where data crosses a trust boundary:
- HTTP request parameters, headers, body
- File system reads and writes
- Database query inputs and outputs
- Environment variables and config files
- Third-party API responses
- WebSocket messages

Confirm each boundary has validation. If it does not, add it.

### Step 4: Auth/Authz Audit

For every new or modified route/endpoint:
- [ ] Authentication middleware is applied
- [ ] Authorization checks verify the caller has permission for the specific resource
- [ ] Rate limiting is configured for public endpoints
- [ ] CSRF protection is active for state-changing operations

### Step 5: Error Handling Audit

Confirm:
- [ ] No stack traces reach the client in production mode
- [ ] Error messages do not reveal database schema, file paths, or internal service names
- [ ] Failed auth attempts return generic messages (not "user not found" vs "wrong password")

### Step 6: Dependency Audit

For every new package added:
- [ ] Check for known CVEs: `npm audit` / `pip audit` / `cargo audit`
- [ ] Verify the package is actively maintained (last publish < 12 months)
- [ ] Confirm the package name is correct (not a typosquat)
- [ ] Review the package's permission requirements

## Red Flags -- Instant Reject

These patterns MUST NOT pass review under any circumstances. If found, reject the change and fix before re-review.

| Pattern | Why |
|---------|-----|
| Hardcoded credentials (API keys, passwords, tokens) | Credentials in source get committed, pushed, and leaked |
| `eval()` or `new Function()` with dynamic input | Arbitrary code execution |
| Shell commands with string concatenation | Command injection |
| SQL with template literals or concatenation | SQL injection |
| Disabled security features (CSRF off, helmet removed) | Removes existing protection |
| `Access-Control-Allow-Origin: *` | Allows any domain to make authenticated requests |
| `console.log` / `print` of passwords, tokens, or PII | Sensitive data in logs |
| `chmod 777` or world-writable permissions | Anyone can read/write/execute |
| `--no-verify` on git operations | Bypasses security hooks |
| JWT with `algorithm: "none"` | Disables token verification |

## Quick Detection Commands

Run these against the changeset before manual review. Each command targets a specific vulnerability class.

### Secrets Detection

```bash
rg -in '(api[_-]?key|secret|password|token|credential|auth)[\s]*[=:]\s*["\x27][^"\x27]{8,}' --glob '!{*.lock,node_modules/**,dist/**,.git/**}'
```

### SQL Injection Patterns

```bash
rg -n '(query|exec|execute|raw)\s*\(\s*(`[^`]*\$\{|["\x27][^"\x27]*\+)' --glob '*.{js,ts,py,rb,go,java}'
```

### Command Injection Patterns

```bash
rg -n '(execSync|spawnSync|system|popen|subprocess)\s*\(\s*(`[^`]*\$\{|["\x27][^"\x27]*\+|.*\breq\b)' --glob '*.{js,ts,py,rb,go,java}'
```

### Insecure Defaults

```bash
rg -n '(cors\(\)|origin:\s*["\x27]\*|0\.0\.0\.0|debug\s*[=:]\s*[Tt]rue|NODE_ENV.*development)' --glob '*.{js,ts,py,rb,yaml,yml,json,toml}'
```

### Security TODOs

```bash
rg -in '(TODO|FIXME|HACK|XXX).*(security|auth|secret|password|credential|inject|sanitiz|validat)' --glob '!{node_modules/**,dist/**,.git/**}'
```

### Dangerous Functions

```bash
rg -n '\b(eval|Function)\s*\(' --glob '*.{js,ts}' | rg -v 'node_modules|\.test\.|\.spec\.'
```

## The Bottom Line

- Run the mandatory checklist on EVERY AI-generated file before committing
- AI-generated vulnerabilities cluster -- finding one means check harder for others
- Secrets, injection, and missing auth are the top three categories
- Automated scans catch the obvious cases; manual review catches the rest
- Fix before committing. Never commit with a TODO to fix later
- If ANY red flag pattern is found, reject the change entirely
