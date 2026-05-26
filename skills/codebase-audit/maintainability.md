# Maintainability Audit

Focus: code that's correct today but expensive to change tomorrow. Maintainability problems compound — every additional reader, every additional change makes them worse. They rarely cause incidents directly, but they're what makes a codebase feel "hard to work in."

Use the finding format and ranking rules from SKILL.md. This file lists what to look for.

## What to look for

### 1. Domain rule leakage

Business rules expressed in places that shouldn't know about them: SQL queries with hard-coded statuses, UI components branching on domain logic, controllers computing derived values that belong on the model, validators duplicating constraints already enforced elsewhere.

**Tells:**
- The same business condition (`if order.status === 'shipped' && order.paid_at`) appearing in 3+ layers
- Database queries with `WHERE` clauses encoding business rules ("active customers" defined inline as `status='active' AND deleted_at IS NULL`)
- Frontend code computing things like discount eligibility or tax rules
- Validators in route handlers re-checking what the model constructor already checks

**Evidence to gather:** 2–3 example locations, ideally from different layers, showing the same rule.

**Smallest safe improvement is usually:** name the rule (extract a predicate/method on the domain object) and replace one or two of the duplicates as a proof-of-concept. Not "extract everything" — that's the refactor.

### 2. Repeated conditional logic

The same `switch` / `if-else` chain over the same discriminator (status, type, role, kind) appearing in multiple files. Each addition of a new case requires editing every site, and missing one is a silent bug.

**Tells:**
- `switch (type)` / `if (kind === ...)` chains on the same field in 3+ files
- Boolean flags that are always checked together (`if (isAdmin && !isReadonly && hasFeature)`)
- Mapper objects (`const LABELS = { ... }`) duplicated across files with slight variations

**Smallest safe improvement:** define the dispatch in one place (a map, a small polymorphic structure, a single function) and migrate the most-edited site to use it. Leave the others for a follow-up.

**What makes this unsafe:** if the conditionals look identical but have subtle differences per site (different default cases, different side effects), consolidating prematurely hides bugs. Read each instance carefully before declaring them equivalent.

### 3. Primitive obsession

Domain concepts (IDs, money, dates, durations, email addresses, percentages) represented as raw `string` / `number` / plain objects throughout the codebase. Every function that takes them has to validate or trust; mistakes silently typecheck.

**Tells:**
- Function signatures like `transfer(fromId: string, toId: string, amount: number, currency: string)` where swapping two strings or mixing currencies typechecks
- Date arithmetic done with raw numbers and inline conversions (`* 1000 * 60 * 60`)
- Money tracked as floats or as dollars in some places and cents in others
- IDs from different entities all typed as `string` or `number`

**Smallest safe improvement:** introduce a single named type for the worst offender (usually money or the most-confused ID) and migrate one boundary (one API or one entry point). Don't try to migrate the whole codebase.

### 4. Magic values

Unexplained constants, especially repeated ones. `if (retries < 5)`, `setTimeout(fn, 300)`, `if (response.code === 4711)` — what is 5? what is 300? why 4711?

**Tells:**
- Numeric or string literals appearing in multiple files
- Off-by-one-prone arithmetic (`age - 18`, `length - 1`) without comment
- HTTP status codes, error codes, or feature flag strings as bare literals

**Smallest safe improvement:** name the constant. This is genuinely a one-line fix per occurrence and rarely unsafe. The audit value is identifying *which* ones — focus on the ones that appear in multiple places, since those are also a change-amplification risk.

### 5. Names that lie or drift

Functions, classes, or variables whose names no longer match what they do — usually because they grew. `UserService.getUser()` that also updates last-seen-at and emits an analytics event. `validate()` that also normalizes. `temp` that's been there for two years.

**Tells:**
- Functions whose body has obvious sections doing different things (look for blank-line-separated blocks)
- Names with hedges: `processData`, `handleStuff`, `doWork`, `manager`, `helper`, `util`
- Comments that explain what a function "actually" does, contradicting the name
- Boolean parameters (`fetchUser(id, true)`) — almost always a name that's lying about what the function does

**Smallest safe improvement:** rename, or split into two well-named functions. Renames are usually safe with tooling; splits need a careful read of the side effects.

### 6. Comment smell

Comments that explain *what* the code does (a sign the code itself doesn't say it), commented-out code, TODOs older than a year, comments that lie because the code changed but the comment didn't.

**Tells:**
- `// loop through the users` above `for (const user of users)` — pure noise
- Large blocks of commented-out code
- `// TODO: fix this (2019)` — either fix it or delete the comment

**Smallest safe improvement:** delete or replace with intent-revealing names. Genuine *why* comments stay; *what* comments and stale TODOs go.

## What to deprioritize

- Pure formatting / lint issues (different concern)
- Single-use magic numbers in obviously throwaway code
- Long functions that are long but linear and well-named (length alone isn't a smell)
- Naming nitpicks that are subjective (e.g. `data` vs `payload`)

## Common cross-references

Maintainability findings often signal problems in other dimensions:
- Repeated conditionals → also a **change amplification** issue
- Domain leakage → also a **structure** issue (layers aren't holding)
- Primitive obsession → also a **correctness risk** (invariants not enforced by types)

Mention these in passing in the relevant findings, but don't audit them here — recommend the focused audit as a follow-up if signals are strong.
