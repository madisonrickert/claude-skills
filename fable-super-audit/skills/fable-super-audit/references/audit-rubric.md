# General Audit Rubric

A language-agnostic checklist for a whole-repository audit. Each check tells you what to hunt for; for every hit, record what, where (`file:line`), why it matters, fact-vs-judgment, and a severity from the table at the end.

The examples name specific languages and frameworks (JavaScript, SQL, JWT, React) only to be concrete. Translate each check to the stack in front of you: "parameterized queries" applies to any database layer, "unhandled promise" to any async model, "component teardown" to any lifecycle system.

The core stance, applied throughout: **trace execution paths and data flows; do not trust the appearance of correctness.** Reading for surface-level compliance misses the defects that matter.

## Contents

1. Architecture & design
2. Async, concurrency & state
3. Security
4. Logic & business-rule integrity
5. Code quality & maintainability
6. Testing
7. Performance
8. Dependencies & supply chain
9. DevEx, operations & documentation
10. Severity classification
11. Optional: read-only evidence tools

---

## 1. Architecture & design

Detect structural inconsistency, responsibility leakage, and coupling that will make every future change harder.

- **Module boundaries & coupling.** Map what each core module exports, imports, and is called by. Flag modules importing from many sources (a likely god module) and modules imported by very many consumers (a critical shared dependency, and your highest-priority review target). Note circular dependencies.
- **Orphan / dead modules.** Trace whether each module has at least one active, non-dead-code caller. Flag modules with zero callers. A module that is tested but never called in production logic is dead code masquerading as live.
- **Pattern consistency.** Identify the primary architectural pattern (layered, MVC, repository, hexagonal). Verify it holds across all modules. Flag deviations, which are often later additions where the pattern was lost.
- **Leaky & cosmetic abstractions.** For each interface or abstract type, ask whether removing it and using the concrete type directly would change any behavior. If not, the abstraction is cosmetic. Flag abstractions that relocate complexity rather than encapsulate it, forcing consumers to understand internals to use them safely.
- **God objects / files & layering violations.** Flag oversized files or classes accreting unrelated responsibilities, and any lower layer reaching up into a higher one.

## 2. Async, concurrency & state

The highest-risk surface in most modern codebases. Surface every edge case the happy path hides.

- **Unhandled async operations.** Map every async function, promise, and await. Verify each has a `.catch` or `try/catch`. An async operation with no handler is a defect.
- **Error-propagation trace.** For every catch block, determine what happens after it. Acceptable: rethrow, return a typed fallback, call a centralized handler, or trigger a state update that notifies the caller. Unacceptable: log-and-return-nothing, or swallow the error entirely. Flag every catch that fails to propagate a meaningful signal.
- **Race conditions & non-atomic writes.** Find every place two or more async operations write shared state (in-memory globals, files, database rows, UI state). Verify a lock, queue, or serialization exists. Watch for handlers that can fire again before a prior invocation finishes, polling loops without cancellation, and message handlers that mutate state without queuing.
- **State lifecycle.** Verify every subscription, listener, connection, or timer created on init has a matching teardown on destroy/unmount. Missing teardowns leak memory and mutate stale state.
- **Boundary conditions.** For each function processing a collection, trace empty, null, and single-item inputs. Zero-value numerics and null responses from external calls are routinely mishandled.

## 3. Security

Hunt for insecure data handling, injection surfaces, and authorization failures. Keep CWE identifiers on findings so they classify precisely.

- **Secrets & credentials (CWE-798).** Scan for API keys, database connection strings, OAuth client secrets, signing keys, and passwords assigned as literals rather than loaded from environment or a secrets manager. Any literal secret is a critical finding. Scan example-env files too; they are frequently populated with real values.
- **Injection surfaces.** Find every point where user-controlled input reaches SQL construction (CWE-89), a shell command, a filesystem path, HTML rendering (CWE-79 XSS), or template evaluation. Verify parameterized queries, safe APIs, or explicit escaping. String concatenation of user input into any query or command is a defect regardless of how "clean" the input looks.
- **Authentication & authorization.** Map every route/endpoint/handler. For each: is authentication enforced server-side; is authorization checked at the *resource* level (does this user own this specific object); are tokens validated for signature, expiry, issuer, and algorithm? Missing resource-level authorization (IDOR) is among the most common critical vulnerabilities and is nearly invisible to naive static analysis.
- **Missing input validation (CWE-20).** Flag functions that accept input and use it directly without null/type/range checks at the boundary.
- **Transport & CORS.** Flag cleartext transport where TLS is required, credentials in query strings, wildcard CORS on authenticated endpoints, and missing security headers (CSP, X-Content-Type-Options, X-Frame-Options, HSTS).
- **Cryptography (CWE-327).** Flag MD5/SHA-1 for password hashing (want Argon2/bcrypt/PBKDF2), non-cryptographic randomness for tokens (want a CSPRNG), hand-rolled crypto, and symmetric keys embedded in source.

## 4. Logic & business-rule integrity

Find code that is syntactically correct but logically wrong.

- **Conditional exhaustiveness.** For each conditional, enumerate inputs and trace which branch each takes. Flag always-true/always-false conditions, checks ordered so an earlier one wrongly wins, and assignment used where comparison was meant.
- **Return-value consistency.** Verify all paths of a function return the same type. Flag functions returning a typed value on success and nothing on the error path without that being in the signature.
- **Data-flow integrity.** Trace each user-facing input from entry to storage/output. Verify validation happens at the entry boundary (not deep in the stack), data is not re-serialized with shifted semantics, and output encoding is applied at the exit boundary.
- **Transaction atomicity.** For multi-step state changes (sequential DB writes, file ops, external calls), verify a failure at step N rolls back steps 1..N-1. Flag missing compensation logic that can leave data half-modified.
- **Concurrency invariants.** For shared state (module-level vars, singletons, caches), verify no interleaving of writes can produce inconsistent state, and that state is reset between logical operations rather than bleeding across requests.

## 5. Code quality & maintainability

Detect the structural debt that makes future defects invisible.

- **Duplication.** Flag blocks of ~10+ lines appearing more than once. Duplicates are a maintenance risk and a security risk: one copy may get a fix the other never does.
- **Complexity hotspots.** Flag functions with high cyclomatic or cognitive complexity; they are the longest, most-branched, and systematically under-tested relative to their branch count. Identify the longest/most-branched functions explicitly.
- **Dead code paths.** Identify unreachable branches, functions whose return value is never consumed, and imported symbols never referenced.
- **Logging security.** Flag log statements that emit request bodies, credentials, tokens, PII, or internal stack traces to external-facing sinks, and debug logging left in production paths.
- **Environment-config validation.** Verify a startup routine checks required environment variables before serving traffic. Referencing config throughout the code without verifying presence causes silent failures or insecure fallbacks.

## 6. Testing

- **Coverage where it counts.** Identify gaps around core business logic and critical paths, not just the raw percentage.
- **Test quality, not just presence.** Distinguish tests that assert specific behavior from tests that only check code runs without error. Flag circular validation (tests written to match the current implementation rather than the intended contract).
- **Missing test types.** Note absent unit/integration/e2e layers and untestable code (logic welded to I/O with no seam).
- **Flakiness.** Flag timing-dependent, order-dependent, or network-dependent tests.

## 7. Performance

- N+1 query patterns and missing indexes.
- Unnecessary allocations or copies on hot paths.
- Blocking calls inside async paths.
- Missing caching where recomputation is expensive and inputs are stable.
- Unbounded growth: memory, files, queues, caches with no eviction.

## 8. Dependencies & supply chain

- **Existence & health.** Verify each dependency exists on its registry, is maintained, and is free of known critical CVEs.
- **Weight & duplication.** Flag unnecessarily heavy packages for trivial needs, and multiple packages doing the same job.
- **Lockfile hygiene.** Flag a missing or stale lockfile, and version pins that were current at authoring time but are now deprecated or vulnerable.
- **License risk.** Note licenses incompatible with the project's distribution.

## 9. DevEx, operations & documentation

- **Build & setup friction.** Can a newcomer get from clone to running in the documented steps? Flag undocumented prerequisites.
- **CI/CD & enforcement.** Note missing lint/format enforcement, missing CI gates, and a hand-wavy deployment story.
- **Observability.** Assess logging, error reporting, and metrics quality for diagnosing a production incident.
- **Documentation accuracy.** Flag README claims that contradict the code, undocumented critical behavior, and stale docs. A doc that lies is worse than a missing one.

---

## 10. Severity classification

| Severity | Criteria | Recommended action (for a human) |
|---|---|---|
| **Critical** | Hardcoded secret; auth bypass; injection; IDOR; remote-code-execution surface | Treat as blocking; recommend immediate remediation |
| **High** | Swallowed async error on a production path; missing CORS restriction; weak crypto | Recommend fixing before next release |
| **Medium** | Orphan state without a null guard; missing input validation on a non-critical path; dead module | Recommend fixing within the current cycle |
| **Low** | Naming inconsistency; duplicate logic block; minor complexity | Recommend fixing in a maintenance pass |
| **Informational** | Cosmetic abstraction; speculative guard; over-specified edge case | Document; refactor when convenient |

The action column is advice for a human. This skill never applies fixes itself.

## 11. Optional: read-only evidence tools

To gather evidence faster, the orientation subagents MAY run established static tools in **report-only** mode: a secret scanner (e.g. gitleaks), a linter for dead code and unreachable branches (e.g. eslint, pyflakes), a pattern scanner (e.g. semgrep), a dependency CVE checker, or a complexity/duplication reporter. Use them only to surface candidates you then verify in the source. Do not install tools, do not modify configuration, and do not integrate anything into CI. These are optional accelerators, not a substitute for tracing the code.
