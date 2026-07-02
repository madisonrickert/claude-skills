# AI-Generated Code Anti-Patterns (audit lens)

A companion lens to the general audit rubric, for codebases that were built or heavily edited by AI coding assistants. AI-generated code fails differently from human-authored code: it optimizes for linguistic and functional coherence, so it tends to *look* correct while violating security or logical contracts underneath. The defects cluster into recognizable patterns, and knowing the patterns makes them far faster to find.

Use this lens hardest on vibe-coded or AI-assisted projects. Each item below tells you what to hunt for; rate hits with the severity table in `audit-rubric.md`.

The governing principle, restated because it is where the value is: **do not trust the appearance of correctness.** Trace execution paths and data flows rather than reading for surface compliance.

## Orientation: markers of AI authorship

Before the substantive passes, gauge how AI-generated the codebase is; it calibrates how hard to lean on this lens. Search for: excessive inline comments explaining trivial logic; unresolved `TODO`/`FIXME` left in place; near-duplicate functions separated by 100+ lines; blocks that use one style convention then switch mid-file. In the history, a large codebase with very few commits (or almost no human-authored changes among them) signals heavy AI generation and a higher chance of the regression patterns in the last section.

## Architectural drift & context-window decay

As an AI assistant's context fills across a long session, its earlier decisions decay. Hunt for the fingerprints:

- **Pattern abandonment.** A pattern (repository, factory, a naming scheme) is used in early files and quietly dropped in later ones. The later files are the suspect additions.
- **Naming-convention drift.** Variable/function naming that shifts mid-file; camelCase and snake_case mixed; a convention that appears then disappears.
- **Near-duplicate utilities.** A helper is written, then a near-identical one appears later because the earlier definition fell out of context. Each divergent copy is a place a fix can land in one and miss the other.
- **Stale cross-module assumptions.** One module assumes how another behaves; as context filled, the assumption went stale and the integration edge silently broke. These inter-module seams, especially where two sides show different naming or error-handling styles, are the highest-probability location for silent contract violations.
- **Context-induced monolithism.** New functionality accreting into existing oversized files instead of new, properly scoped modules.

## The vanilla-vs-over-abstraction duality

AI code tends toward one of two opposite failures depending on how it was prompted; check for both:

- **Under-architected (vanilla).** No separation of concerns; business logic, data access, and presentation fused in single functions; no service layer.
- **Over-abstracted.** Indirection that adds no isolation: factories of factories, abstract bases with a single implementation, interfaces wrapping concrete types. The diagnostic: does the abstraction *hide* complexity or merely *relocate* it? Relocated complexity is a leaky layer.

## Superficial error handling

AI error handling is often aesthetically correct but semantically hollow. Flag:

- **Catch-and-discard.** A `try/catch` that logs and returns nothing, handing control back with no usable signal or fallback. (Trace these via the error-propagation check in the general rubric; they are endemic to AI async code.)
- **Over-verbose propagation.** Stack traces, internal paths, or schema details surfaced directly in responses.
- **Symmetric error messages.** Every failure yields the same generic message regardless of context, giving a false look of robustness while making debugging impossible.
- **Missing boundary validation (CWE-20).** Inputs consumed directly without null/type/range checks at the function boundary. This is the single most common security flaw in AI-generated code across languages.

## Phantom guards & over-specification

The inverse of missing checks: AI also adds *noise* guards. Flag checks for impossible or redundant conditions, speculative branches for inputs that cannot occur, and edge-case handling for cases the surrounding code already precludes. These clutter core logic and inflate complexity without adding safety, and they can mask the guards that are actually missing.

## Async & state failures characteristic of AI code

The general rubric covers async broadly; these are the AI-specific tendencies to check first:

- **Orphan state.** State initialized but never cleaned up, never consumed, or conditionally written yet unconditionally read. Watch for variables written on some paths and read without a null guard on others, and listeners/subscriptions registered with no teardown.
- **Stale-closure mutation.** Async operations that mutate state after the owning component or scope has been torn down.
- **Unguarded concurrent writes.** Shared state written by handlers that can fire concurrently, with no lock, queue, or atomic operation.

## Feedback-loop security degradation (iterative regression)

A counterintuitive but well-documented pattern: security tends to *degrade* as AI iterates on code, because each "improvement" cycle can quietly relax a constraint, widen a scope, or drop a control. If commit history is available, audit for it:

- **Before/after regression.** Find commits where an assistant modified security-sensitive code (auth middleware, input validation, cryptography). For each, verify the change did not remove or weaken a pre-existing control. Treat each such "improvement" as a regression candidate.
- **The security-focused regression trap.** Even code written under an explicit security prompt degrades over iterations. For any security-critical block, verify the implementation is complete, not a surface approximation. Classic partial jobs: a JWT check added without algorithm pinning (the `alg` field), or parameterization added to new queries while pre-existing raw queries in the same file are missed.
- **Context-boundary integrity.** Re-examine the inter-session seams from the context-decay section specifically for weakened or dropped validation, since those boundaries are where controls most often fall through.

## Dependency hallucination & slopsquatting

AI models fabricate plausible package names and suggest libraries with a confidence that does not track existence.

- **Hallucinated dependencies.** For every declared dependency, confirm it actually exists on its registry. A package that cannot be found is a critical supply-chain risk: attackers register these predicted names ("slopsquatting") with malicious code.
- **Dependency overuse.** Simple functionality pulling a heavy or deep dependency tree, a sign the code reached for a package where a few lines would do.
- **Reintroduced CVEs.** Version pins that were current at the model's training cutoff but are now deprecated or known-vulnerable.
