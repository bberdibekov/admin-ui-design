# Correctness Risk Audit

Focus: places where the code works today but is vulnerable to bugs that won't show up in code review — assumptions enforced by convention rather than structure, error paths that swallow signal, boundaries that trust their inputs.

This dimension straddles "maintainability" and "bug hunt" — you're not looking for *specific* bugs (that's a different exercise), you're looking for the *shapes of code* that make bugs likely.

Use the finding format and ranking rules from SKILL.md.

## What to look for

### 1. Implicit invariants

Assumptions about state that the type system doesn't enforce: "this list is non-empty by the time it reaches here," "this field is set after step 3," "these two fields are never both null."

**Tells:**
- `array[0]` accessed without checking length, where length isn't statically guaranteed
- `.get()` / `.first()` / `.find()` results dereferenced without null checks
- Comments like `// at this point, X is always Y`
- Fields that are technically nullable but "always set in practice" — usually documented in a comment instead of the type
- Code paths that only work if a prior step succeeded, but no type-level link enforces it (a `User` object that's only valid after authentication, but typed the same before and after)
- Defensive checks that are inconsistent — some call sites check, others don't, suggesting the contract is unclear

**Why it's a risk:** the invariant holds today because of how callers happen to call. A refactor, a new caller, or a change in upstream code violates it silently. The bug surfaces in production, often as a `NullPointerException` / `TypeError: Cannot read property 'x' of undefined` in a place that looks unrelated.

**Smallest safe improvement:** for one important invariant, encode it in the type (a non-empty list type, a separate type for "validated" vs "raw" input, a tagged union). Migrate the one place that produces the value to construct the new type; let the consumers benefit immediately. Don't migrate everything.

**What makes this unsafe:** if the invariant is *almost* but not always true (there's a path where the list can be empty and the code handles it by accident), encoding the type breaks that path. Find the exceptions before encoding.

### 2. Error handling inconsistency

Different parts of the codebase signal failure differently — exceptions in some places, `null` returns in others, `Result` / `Either` types elsewhere, boolean success flags in a fourth place. The inconsistency itself is the bug source: callers handle errors based on the convention they expect, not the one the callee actually uses.

**Tells:**
- A mix of `throw new Error(...)`, `return null`, `return { ok: false, error: ... }`, and `callback(err, ...)` in the same module
- Functions that throw in some branches and return `null` in others
- Async functions that swallow rejections silently (missing `await`, missing `.catch`, `Promise.all` without error propagation)
- Empty `catch` blocks (the worst — silent failure)
- `catch` blocks that just log and continue, when the operation actually failed
- Caller code that only handles some of the failure modes the callee can produce
- Errors caught at a level that has no useful context (top-level catch-all that just logs "something went wrong")

**Why it's a risk:** real failures get swallowed. Recovery logic runs when it shouldn't. Bugs are hidden in production. The error path is the least-tested part of any system, so weaknesses here are durable.

**Smallest safe improvement:** for one identified inconsistency, name the convention this module should follow and migrate the most-painful site. (Don't try to unify the whole codebase.) For empty catches, name each one and propose either deletion (let it propagate) or explicit handling.

**What makes this unsafe:** changing from "swallow error" to "propagate error" is a behavior change. Callers may have relied on the swallowing — sometimes accidentally, sometimes deliberately. Note this when recommending the change.

### 3. Boundary validation gaps

Untrusted input flowing into the system without a clear validation seam: external API responses parsed as if they match the expected shape, user input deserialized directly into domain objects, message queue payloads assumed to be well-formed.

**Tells:**
- `JSON.parse(...)` followed by direct field access, no schema validation
- API clients that cast responses to a TypeScript type without runtime checking
- Domain object constructors that accept raw JSON / dicts
- Request handlers that use body fields without validating their types
- `as` / `cast` / type assertions at trust boundaries (TypeScript `as` is the classic tell)
- Message handlers that destructure without checking field presence

**Why it's a risk:** the type checker thinks the input is safe, but at runtime it's whatever the external system actually sent. Crashes, security issues, or silent data corruption follow.

**Smallest safe improvement:** identify one boundary and propose a validation layer there (runtime schema like Zod / Pydantic / JSON Schema / similar). Just one — the seam between the external system and the domain. Don't validate everywhere; validate at the boundary.

### 4. Mutation of shared state

Functions that mutate their arguments, shared collections, or singleton state. The danger isn't mutation per se — it's *non-local* mutation, where caller assumptions about their data are broken by a callee.

**Tells:**
- Functions that take an object/array and modify it (especially when they also return a value, which suggests the caller may not realize the input was mutated)
- Sort/filter operations that mutate (e.g. `Array.prototype.sort` in-place in JS)
- Cache or registry objects passed around and added to from multiple sites
- Object spreading at one level but not deeper (shallow copy, deep mutation)

**Why it's a risk:** the bug looks like a state corruption that's impossible to reproduce, because it depends on the *order* of operations across the codebase.

**Smallest safe improvement:** for one identified mutation, switch to returning a new value rather than mutating. Migrate that one site. Naming the pattern is more important than fixing all instances.

### 5. Concurrency / ordering assumptions

Code that works because of timing rather than because of structure: assumes a callback fires before another, that two async operations don't overlap, that initialization completes before use.

**Tells:**
- `setTimeout(fn, 0)` or `setImmediate` to "wait for" something
- Booleans like `isInitialized` / `isReady` checked or set without synchronization
- Async functions called without `await` where the result matters
- Race conditions guarded by retry-with-delay rather than actual sequencing
- Tests that need `await sleep(100)` to pass
- Multiple writers to the same data without obvious coordination
- Optimistic concurrency without proper version/etag handling

**Why it's a risk:** works on the dev machine, fails under load. Or works for a year, then a runtime upgrade changes scheduling behavior and breaks.

**Smallest safe improvement:** identify one race and name the actual contract that's missing (e.g. "this operation requires X to be loaded; either await X explicitly or pass it in"). The fix is usually structural, but the audit value is naming the implicit dependency.

### 6. Logging and observability gaps

Not strictly a correctness risk, but worth a finding: critical operations that fail silently because nothing logs the failure, error paths with no context, state changes that aren't observable from outside the process.

**Tells:**
- Errors caught and logged without enough context to debug (no IDs, no input, no stack)
- Critical paths (payment, auth, data writes) with no audit log
- Logging that's so noisy nobody reads it (or so quiet the important events are lost)

**Why it's a risk:** when a correctness issue does happen, you can't tell. Bugs that happened a month ago are unrecoverable.

**Smallest safe improvement:** identify the 1–2 critical paths most lacking in observability and propose specific log/metric additions. Don't propose an observability platform.

## What to deprioritize

- Theoretical race conditions in single-threaded code (JavaScript main thread, single-worker Python without async)
- Validation gaps on inputs that are demonstrably safe (config files baked into the deploy)
- Error handling philosophy debates (exceptions vs. result types) — flag inconsistency, not the choice

## Common cross-references

- Implicit invariants often co-occur with **maintainability** (primitive obsession) — the type system can fix both
- Error handling inconsistency often co-occurs with **testability** (untested error paths)
- Boundary validation gaps often co-occur with **structure** (abstraction leaks at the boundary)
