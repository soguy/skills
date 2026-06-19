---
name: project-doctor
description: "Diagnoses and heals software projects end-to-end: dead code removal, bug detection, security audit, build/test sanity checks, and documentation sync. Use for code review, cleanup, audit, refactoring, or any codebase health task."
---

# Project Doctor Skill

A structured pipeline for analyzing a project, removing dead code, fixing bugs, auditing security, running sanity checks, and keeping documentation in sync with reality.

---

## Phase 0 — Orient

Before doing anything else, understand the project:

1. **Read `view /path/to/project`** to see the directory tree.
2. Look for `README.md`, `CHANGELOG.md`, `package.json` / `pyproject.toml` / `Cargo.toml` / equivalent to understand the project's purpose, stack, and entry points.
3. Identify the primary language(s), framework(s), and test runner.
4. Note any CI config (`.github/workflows`, `Makefile`, `Justfile`) — these reveal the canonical build/test/lint commands.

Summarize your findings to the user in 3–5 bullet points, then proceed immediately to Phase 0.5.

---

## Phase 0.5 — Simplify

Before deeper analysis, run the `simplify` skill on any recently changed code. This catches reuse, quality, and efficiency issues introduced by recent work before they get baked in further.

Invoke: use the `Skill` tool with `skill: "simplify"`.

When simplify completes, **do not pause or wait for user input** — proceed immediately to Phase 1.

---

## Phase 1 — Dead Code Removal

Goal: delete code that is never reached, never imported, or never called.

### 1a. Automated detection
Run language-appropriate static analysis tools if available:

| Language | Tool | Install check |
|---|---|---|
| JavaScript / TypeScript | `npx ts-prune` or `npx unimported` | `which npx` |
| Python | `vulture` | `pip show vulture` |
| Rust | `cargo udeps` | `cargo udeps --help` |
| Go | built-in: `go build ./...` with `-gcflags="-e"` | — |
| Generic | `grep`-based import tracing | always available |

If a tool isn't installed, note it but don't block — proceed manually.

### 1b. Manual patterns to check
- Exported symbols never imported outside their own file
- Commented-out blocks > 5 lines old (check git blame if `.git` exists)
- Feature flags / env vars that no longer exist in any config or CI
- Test helpers imported only in files that no longer exist
- Duplicate utility functions (same logic, different names)

### 1c. Removal rules
- **Never remove** code marked `// TODO: re-enable` or `# noqa: dead` unless the user explicitly says so.
- **Always** remove the import/require statement when you remove the last usage of a symbol.
- After removing, re-run the build/test command (see Phase 3) to confirm nothing broke.

---

## Phase 2 — Bug Detection & Fixes

### High-priority patterns to look for
1. **Off-by-one errors** — loop bounds, slice indices, pagination offsets
2. **Null / undefined dereference** — accessing properties without guards
3. **Async errors not caught** — `await` inside `try` without `catch`, unhandled Promise rejections
4. **Resource leaks** — file handles, DB connections, event listeners never cleaned up
5. **Race conditions** — shared mutable state modified in concurrent paths
6. **Incorrect error propagation** — swallowed errors (`catch (e) {}`), wrong HTTP status codes returned
7. **Type coercion surprises** — `==` vs `===`, implicit string/number conversion
8. **Hardcoded secrets or localhost URLs** — flag these; don't auto-fix, just report
9. **Missing input validation** — user-supplied data passed directly to DB queries, shell commands, or file paths
10. **Logic inversion** — `if (!error) { throw }` style mistakes

### Fix approach
- Fix bugs that are unambiguous (clear wrong behavior, clear correct behavior).
- For ambiguous cases, add a `// FIXME(review): <description>` comment and include it in the summary report.
- Do not refactor working code just because you disagree with the style — that's a separate task.

---

## Phase 3 — Security Audit

Goal: identify vulnerabilities, insecure patterns, and security misconfigurations. Focus on issues from the OWASP Top 10 and common language-specific pitfalls.

### 3a. Secrets & credentials
- **Hardcoded secrets** — API keys, passwords, tokens, connection strings in source code (not just `.env`)
- **Secrets in version control** — check `.gitignore` covers `.env`, `*.pem`, `*.key`, credential files. Run `git log --all -p -- '*.env' '*.key' '*.pem'` to check if secrets were ever committed.
- **Overly broad `.env` exposure** — environment variables containing secrets loaded into frontend bundles or logged to stdout
- **Default credentials** — admin/admin, test tokens, placeholder API keys shipped in config

### 3b. Injection vulnerabilities
- **SQL injection** — raw string concatenation in queries instead of parameterized/prepared statements. Check ORM raw queries and any `execute()` calls with f-strings or `%` formatting.
- **Command injection** — user input passed to `os.system()`, `subprocess.run(shell=True)`, `child_process.exec()`, or backtick execution without sanitization
- **Path traversal** — user-supplied filenames used with `open()`, `fs.readFile()`, or `os.path.join()` without validating they stay within expected directories
- **Template injection** — user input rendered in server-side templates without escaping (Jinja2 `|safe`, `dangerouslySetInnerHTML`, `v-html`)
- **XSS (Cross-Site Scripting)** — user-supplied data rendered in HTML/JSX without proper escaping. Check `dangerouslySetInnerHTML`, `innerHTML`, `document.write()`, and URL parameters reflected in pages.
- **Log injection** — user input written to log files without sanitization (enables log forging, CRLF injection)

### 3c. Authentication & authorization
- **Missing auth on endpoints** — API routes that should require authentication but don't enforce it
- **Broken access control** — endpoints that check auth but not authorization (e.g., any logged-in user can access admin routes)
- **Insecure session handling** — session tokens in URLs, missing `HttpOnly`/`Secure`/`SameSite` cookie flags, no session expiry
- **Weak password handling** — plaintext storage, weak hashing (MD5, SHA1), missing salt, no rate limiting on login
- **JWT issues** — `alg: none` accepted, secrets in code, no expiry, overly broad claims

### 3d. Data exposure & privacy
- **Sensitive data in API responses** — passwords, tokens, internal IDs, PII leaked in API output that the frontend doesn't need
- **Verbose error messages** — stack traces, SQL errors, or internal paths exposed to users in production
- **Missing CORS restrictions** — wildcard `*` origin in production, or overly permissive origins
- **Unencrypted sensitive data** — PII or credentials stored without encryption at rest

### 3e. Dependency & configuration security
- **Known vulnerable dependencies** — run `npm audit` / `pip audit` / `cargo audit` / `snyk test` if available. Flag critical/high severity findings.
- **Outdated dependencies** — major versions behind with known CVEs
- **Debug mode in production** — `DEBUG=true`, `FLASK_ENV=development`, verbose logging enabled by default
- **Missing security headers** — no CSP, HSTS, X-Frame-Options, X-Content-Type-Options in HTTP responses
- **Insecure TLS** — HTTP URLs for APIs or webhooks, self-signed cert acceptance, disabled cert verification (`verify=False`)

### 3f. Language-specific checks

| Language | What to check |
|---|---|
| Python | `eval()`, `exec()`, `pickle.loads()` on untrusted data, `yaml.load()` without `Loader=SafeLoader`, `subprocess` with `shell=True`, `__import__()` with user input |
| JavaScript / TypeScript | `eval()`, `Function()` constructor, `innerHTML`, prototype pollution via `Object.assign`/spread on user input, `require()` with dynamic paths, RegExp DoS |
| Go | `fmt.Sprintf` in SQL queries, unchecked `err` returns, `unsafe` package usage |
| Rust | `unsafe` blocks, `.unwrap()` on user input, `std::process::Command` with unsanitized args |
| Ruby | `send()`/`public_send()` with user input, `ERB` without escaping, `YAML.load` on untrusted data |

### 3g. Fix approach
- **Auto-fix** clear-cut issues: replace `shell=True` with argument lists, add parameterized queries, add `.gitignore` entries for secret files, set `Secure`/`HttpOnly` cookie flags.
- **Flag but don't auto-fix** issues that require architectural decisions: adding auth middleware, implementing RBAC, choosing an encryption strategy. Mark with `// SECURITY(review): <description>`.
- **Never remove security controls** — even if they look unused, they may be defense-in-depth. Ask the user first.
- **Rotate exposed secrets** — if you find a secret committed to git history, flag it with **HIGH** severity and recommend rotation. Do not just delete the current reference; the secret is already in history.

---

## Phase 4 — Sanity Checks

Run the project's own verification suite. Adapt commands to what exists:

```bash
# Try each in order, stop at first that works:
npm test          # Node / JS
npm run lint
npx tsc --noEmit  # TypeScript type check

python -m pytest  # Python
python -m mypy .

cargo test        # Rust
cargo clippy -- -D warnings

go test ./...     # Go
go vet ./...

make test         # Generic Makefile
just test         # Justfile
```

Record output. If tests fail:
1. Determine if the failure is pre-existing or caused by Phase 1–3 changes.
2. Fix failures caused by your own changes first.
3. For pre-existing failures, list them in the summary report as "Pre-existing failures."

If no test runner exists, note this as a finding and suggest adding one.

---

## Phase 5 — Documentation Updates

Goal: make docs reflect the current state of the code. **This phase is mandatory — update stale documentation directly, do not merely flag issues.**

### 5a. README.md
Check every claim against reality: listed features, installation steps, API/CLI usage examples, environment variables, badges. **Fix all stale content in-place.**

### 5b. All project documentation files
Scan the entire project for documentation files (`.md` files, `docs/` directories, spec files, plan files, deferred-issues trackers). For each file found:
- Verify every factual claim against the current codebase (scoring weights, API endpoints, field names, feature lists, file paths, architecture descriptions).
- **Update any stale content directly** — wrong numbers, missing features, outdated field names, removed capabilities, completed TODO items.
- Remove or archive documentation for features that no longer exist.
- Add documentation for features that exist in code but are not yet documented.
- If a file tracks issues/TODOs (e.g., `DEFERRED_ISSUES.md`), check off items that have been addressed and add notes about partial fixes.
- **Cross-doc consistency** — when two docs cover the same fact (e.g. README install steps vs `docs/setup.md`), reconcile contradictions. Pick the one that matches the code; update the other.
- **Internal links** — flag broken relative links (`./foo.md`, `../api.md#section`) introduced when files are renamed or moved.
- **Code snippets in docs** — sanity-check that example code matches current signatures (function names, argument order, env var names). Stale snippets are the most-copied form of stale documentation.

### 5c. Inline documentation
- Update or add docstrings/JSDoc for functions changed in Phase 2–3
- Remove doc comments for code deleted in Phase 1
- Fix `@param` / `:type` annotations that no longer match actual signatures

### 5d. CHANGELOG / release notes
If `CHANGELOG.md` exists, add an entry under `## Unreleased` using Keep a Changelog format:
```markdown
## [Unreleased]
### Removed
- Deleted unused `fooHelper` utility (dead code)
### Fixed
- Fixed off-by-one in pagination offset calculation
### Security
- Fixed SQL injection in search endpoint
```

### 5e. Architecture / design docs
If `docs/`, `wiki/`, or `.md` files describe the system architecture, check them against the actual file structure and update any paths or module names that have changed.

---

## Phase 5.5 — Codex Cross-Check (conditional)

Goal: get a second opinion from Codex on the cleaned, fixed, doc-synced state — and act on what it finds without spending Codex tokens on the fix.

This phase is **gated**. Run it only when both conditions hold; otherwise skip silently and note the skip in the summary.

### 5.5a. Detection gates

Both must pass:

1. **Host can invoke plugin slash commands.** This skill instructs the host agent to run `/codex:review`. Plugin slash commands of the form `/<plugin>:<command>` only work inside Claude Code. If you are the host agent and you cannot invoke such commands — e.g. you are running inside Codex CLI, Cursor, a non-Claude harness, or a stripped-down environment — skip this phase. (Corroborating signal, not a substitute for the capability check: `$CLAUDE_PLUGIN_ROOT` is set during plugin command execution. Do not rely on the existence of `~/.claude/plugins/` — that directory persists on disk after install regardless of which agent is currently running.)
2. **`/codex:review` is installed.** Test:
   ```bash
   ls ~/.claude/plugins/cache/openai-codex/codex/*/commands/review.md 2>/dev/null \
     || ls ~/.claude/plugins/marketplaces/openai-codex/plugins/codex/commands/review.md 2>/dev/null
   ```
   If neither path exists, skip.

If either gate fails, log one line — e.g. `Phase 5.5 skipped: codex plugin not installed` or `Phase 5.5 skipped: host cannot invoke plugin slash commands` — and proceed to Phase 6.

### 5.5b. Run the review

When both gates pass, invoke the slash command in the foreground:

```
/codex:review --wait
```

Capture Codex's findings verbatim from the command output. Do not paraphrase. Do not ask Codex to fix anything — `/codex:review` is review-only by design, and the goal here is to keep Codex's token usage to the review pass only.

If `/codex:review` errors out (auth failure, network issue, codex CLI missing), treat it as a skip: log the error verbatim into the summary's phase-status line and proceed to Phase 6.

### 5.5c. Triage and fix locally

Walk Codex's findings and classify each one:

- **Auto-fixable** — clear wrong → clear right, same bar as Phase 2/3. Fix it now, in the working tree, using your own (Claude) tokens. Do not call Codex again.
- **Needs decision** — architectural, ambiguous, or stylistic. Add to "Unfixed Findings" with severity, file:line, and Codex's reasoning as the "Why not fixed" cell. Tag the source as `codex`.
- **False positive** — Codex flagged something already correct or already fixed in earlier phases. Don't list each one; just count them and report `Codex false positives: N` in the summary.
- **Conflicts with earlier phase** — Codex's finding contradicts a fix or deletion from Phases 1–3. Defer to "Unfixed Findings" with `Why not fixed: conflicts with earlier phase decision` so the user can adjudicate.

### 5.5d. Re-verify

If Phase 5.5 made any code changes, re-run the project's test command from Phase 4 (just the test runner — full lint/typecheck only if Phase 4 caught something there). If a fix breaks tests, revert it and move that finding to "Unfixed Findings" with `Why not fixed: caused regression`.

### 5.5e. Principles

- **Codex finds, Claude fixes.** Never round-trip back to Codex for fixes — that's what defeats the token-saving point of this phase.
- **Same fix bar as Phase 2/3.** Auto-fix only the unambiguous; defer the rest to the report.
- **Verbatim findings.** When listing Codex-sourced items in the summary, preserve Codex's wording — don't rewrite it.
- **Skip silently, report the skip.** A missing plugin or a wrong host shouldn't break the pipeline; it should be a one-line note in Phase 6.

---

## Phase 6 — Summary Report

Produce a structured report in the conversation. Do **not** write it to a file unless the user asks.

```
## Code Review Summary

### Project
<name, language, framework — one line>

### Phase Status
- **Phase 5.5 (Codex cross-check)**: <ran | skipped: reason>
- **Codex false positives**: <count, only if phase ran>

### Changes Made
- **Dead code removed**: <count> items — <brief list>
- **Bugs fixed**: <count> — <brief list>
- **Security issues fixed**: <count> — <brief list>
- **Codex cross-check fixes**: <count> — <brief list>  *(omit row if Phase 5.5 skipped)*
- **Docs updated**: <which files, what changed>

### Security Findings
Severity levels: 🔴 CRITICAL | 🟠 HIGH | 🟡 MEDIUM | 🔵 LOW | ℹ️ INFO

| # | Severity | Category | Description | File:Line | Status |
|---|----------|----------|-------------|-----------|--------|

### Other Findings (not auto-fixed)
- FIXME items: <list with file:line>
- SECURITY(review) items: <list with file:line>
- Pre-existing test failures: <list>
- Missing tooling: <e.g., "no test suite found">

### Unfixed Findings
List every finding that was **not** fixed — regardless of reason (low severity, needs architectural decision, ambiguous, out of scope). Include findings from both Claude's own analysis and Codex (Phase 5.5). For each:

| # | Severity | Source | Category | Description | File:Line | Why not fixed |
|---|----------|--------|----------|-------------|-----------|---------------|
| 1 | 🔵 | claude | Config | Debug mode enabled by default | settings.py:5 | Low priority |
| 2 | 🟡 | claude | Auth | No rate limiting on login | auth.py:80 | Architectural decision needed |
| 3 | 🟡 | codex  | Logic | Possible race in cache eviction | cache.go:142 | Conflicts with earlier phase decision |

After presenting the full report, ask the user:
> "There are <N> unfixed findings above. Would you like me to fix any or all of them?"

If the user says yes (to all or some), fix them now and update the report.

### Recommended Next Steps
<3–5 actionable items, security fixes first>
```

---

## Principles

- **Minimal blast radius** — prefer small, targeted changes over sweeping rewrites.
- **Verify after each phase** — run the build/test after Phase 1 and again after Phase 2.
- **Explain, don't just change** — for non-obvious changes, add a one-line comment or mention it in the summary.
- **Ask before deleting anything ambiguous** — if you're unsure whether code is truly dead, ask.
- **Never auto-commit** — present the summary, let the user decide what to stage.
- **Security findings get priority** — list security issues first in the summary report.
- **Never suppress security controls** — don't weaken auth, validation, or sanitization without explicit user approval.
