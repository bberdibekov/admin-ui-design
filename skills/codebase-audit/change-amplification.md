# Change Amplification Audit

Focus: places where a single logical change forces edits in many files. Change amplification is the silent tax on every future feature — and it grows nonlinearly as the codebase ages.

This dimension overlaps with maintainability and structure, but the lens is different: you're asking "if requirement X changed tomorrow, how many places would I have to touch?"

Use the finding format and ranking rules from SKILL.md.

## What to look for

### 1. Shotgun surgery hotspots

A single logical change requiring edits in many unrelated-looking files. Adding a new user role means touching the model, the permissions check, the UI, the API serializer, the audit log, the analytics event, the email template — none of which knew about each other.

**How to find:**
- **Read git history if available.** Look at the last 10–20 commits that added a feature (not bugfixes). How many files did each touch? Which files keep showing up together?
- Look for "parallel hierarchies" — when adding a new variant means adding a class/file in 3+ different directories
- Search for any enum-like set (statuses, types, roles) and count how many files match against it

**Tells:**
- The same `switch (kind)` or `if (type === ...)` chain across multiple files
- A "list of supported X" defined separately in the backend, the frontend, the migration, and the docs
- Adding a feature requires editing both a hand-written file and a generated file (sign that the generation isn't covering enough)
- File pairs that always change together — `*Service.ts` and `*Controller.ts` always touched in the same PR

**Why it's a risk:** the cost of every change scales with the amplification factor. Worse, missing one of the N places is a silent bug that won't show up until that path is exercised.

**Smallest safe improvement:** for one identified hotspot, name the missing single-source-of-truth (an enum, a registry, a strategy table, a polymorphic dispatch). Migrate the easiest-to-test site to use it. Don't migrate them all in one pass.

**What makes this unsafe:** if the parallel sites have drifted apart (each has subtle differences), consolidating them prematurely loses information. Audit the differences before recommending consolidation.

### 2. Primitive obsession (change amplification angle)

Same concept as in maintainability, but here the lens is: "every place that uses this raw type has to know the rules." Listed here too because the *change* cost of primitive obsession is often what makes it worth fixing.

**Tells:**
- A validation rule (e.g. "phone numbers must be E.164") that has to be repeated everywhere a phone number is read
- Unit conversions sprinkled throughout (Celsius vs Fahrenheit, dollars vs cents, seconds vs milliseconds)
- ID format checks at every entry point

**Smallest safe improvement:** introduce one named type for the worst offender. Migrate boundaries first (the parse step at API ingress) so the type appears at the seam rather than internally. See `maintainability.md` section 3 for more.

### 3. Feature flag / config sprawl

Branching on feature flags, environment variables, or config values scattered throughout business logic — not centralized at any obvious decision point.

**Tells:**
- `if (process.env.NEW_BILLING_ENABLED)` or `if (config.useV2)` appearing in many files
- Feature flag names that no one remembers the meaning of
- Dead flags — flags that are always on or always off in production but the alternate branch still exists
- Configuration values being read inside business logic rather than at startup
- Flag checks at multiple nested levels for the same flag

**Why it's a risk:** the actual behavior of the system becomes impossible to reason about statically — it depends on flag state. Stale flags accumulate; the "off" branch of each flag is rarely tested and may have rotted.

**Smallest safe improvement:** identify dead flags (flip-once-and-forget) and propose removal of the unused branches. For live flags, propose collapsing the checks to a single decision point that returns a strategy/object. Don't remove flags yet — list them.

### 4. Dead code and unused exports

Code that's not reachable, or exported but never imported, or behind flags that are always off. Inflates the cognitive surface area of every change ("do I need to keep this working too?").

**How to find:**
- Tooling: `ts-prune`, `knip`, `vulture` for Python, IDE "find references" on exports
- Search for exported functions/classes and count call sites
- Look for files in unusual locations (`legacy/`, `old/`, `v1/`, `_archive/`)

**Tells:**
- Functions with zero references outside their defining file (but exported anyway)
- Whole files no one imports
- `if (false)` or `if (DEBUG)` blocks where the constant is never true in any environment
- Commits that say "deprecated" or "to be removed" older than 6 months

**Why it's a risk:** every reader has to spend cycles deciding whether dead code matters. Tools like rename-symbol act on it. Tests cover it (badly), inflating coverage stats.

**Smallest safe improvement:** delete it. Genuinely. Dead code removal is one of the safest changes possible — version control remembers if you need it back. List the candidates with high confidence.

**What makes this unsafe:** if the "dead" code is actually reachable via reflection, dynamic dispatch, a build step, or external callers (it's a library, exported via a package), deleting breaks consumers. Check for dynamic call sites and public API status before recommending deletion.

### 5. Boolean parameters and configuration objects

Functions that take boolean flags or "options" objects whose values are checked inside via more conditionals. Each new option compounds the branching.

**Tells:**
- `function doX(input, isAdmin, dryRun, verbose, ...)` — long boolean tails
- Functions taking `options: { ... }` where the options materially change the behavior (not just decorate it)
- Test files where the same function is called with many different option combinations to exercise different paths

**Why it's a risk:** the function is actually N functions in a trenchcoat; every option doubles the test combinations and the cognitive load.

**Smallest safe improvement:** identify the worst offender. Propose splitting into named variants (`doXAsAdmin`, `doXDryRun`) or, if the options compose meaningfully, a strategy object. Don't split yet — name the variants.

### 6. Schema/contract duplication

The same shape defined in multiple places: TypeScript types and a Zod/JSON schema, an OpenAPI spec and hand-written client types, a database schema and an ORM model and a DTO. Each duplicate is a place that drifts.

**Tells:**
- Two files defining "the same" interface with slightly different field names or types
- API client types that are hand-maintained alongside server-generated specs
- DTOs that are 95% the model with minor differences
- Migrations whose column types don't quite match the ORM model's types

**Smallest safe improvement:** identify the source of truth (usually the one closest to the data) and generate the others. For one pair, propose either generation or runtime validation deriving from the source. Don't migrate all schemas in one pass.

## What to deprioritize

- DRY taken too literally — two identical-looking blocks that change for different reasons should stay separate
- Small duplication in tests (often clearer than abstraction)
- Boolean parameters in trivial private helpers (only flag when public/widespread)

## Common cross-references

- Shotgun surgery + low cohesion → strong **structure** signals
- Primitive obsession + repeated validation → also a **correctness risk** signal
- Feature flag sprawl + dead code → often the same underlying problem ("we never clean up")
