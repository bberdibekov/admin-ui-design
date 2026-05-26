# Structure / Architecture Audit

Focus: how the codebase is organized into modules and how those modules depend on each other. Structural problems are slow-burning — they don't cause incidents, they cause the codebase to gradually become harder to reason about until people stop trying.

Use the finding format and ranking rules from SKILL.md.

## What to look for

### 1. Dependency direction violations

Imports flowing the wrong way: low-level modules importing from high-level ones, the domain importing from the infrastructure, "stable" modules importing from volatile ones.

**Tells:**
- A domain model file importing from a controller, route handler, or HTTP framework
- A core utility importing from a feature module
- "Inner" layers in a layered architecture importing from "outer" ones
- Modules that should be leaves of the dependency graph importing siblings

**How to find them:** scan import statements at module boundaries. If you have language tooling (`madge` for JS, language server for Python/TS, etc.), use it; otherwise grep for cross-boundary imports.

**Why it's a risk:** the inverted dependencies prevent the lower-level modules from being reused or tested in isolation. They also tend to cascade — once one wrong-way import exists, more follow, because "we already import from there."

**Smallest safe improvement:** for the single worst offender, identify what the lower layer actually needs from the higher one. Usually it's a small interface that can be moved or inverted. Don't fix all of them — name the principle and apply it once.

### 2. Circular dependencies

Module A imports B which (directly or transitively) imports A. Often hidden by barrel files or runtime imports.

**Tells:**
- Modules that mutually `import` each other directly
- Errors like "X is not a constructor" or "undefined is not a function" at module load (classic JS symptom)
- Heavy use of late `require` / dynamic imports inside functions (often a workaround for circularity)
- Files that import from `../index` or `../../index` (barrel imports often hide cycles)

**How to find them:** language tools. `madge --circular` for JS/TS. Static analyzers for Python. For other languages, the type checker often reports them.

**Why it's a risk:** initialization order becomes brittle, refactoring becomes risky, and the cycle hints at a missing abstraction — two modules are really one tangled thing.

**Smallest safe improvement:** identify the single most painful cycle and the missing concept. Often it's a third module that should exist (containing the shared types or interface) that both A and B import from. Don't break all cycles — name the pattern, point to the first one.

### 3. Abstraction leaks

Implementation details bleeding through layers that should hide them: SQL fragments in the domain, HTTP types in the business logic, framework types leaking up into the UI, ORM-generated classes used as domain models.

**Tells:**
- Domain code importing from `mongoose`, `sequelize`, `prisma`, `sqlalchemy`, etc.
- Business logic functions taking `Request` / `Response` objects as parameters
- UI components consuming raw API response shapes directly
- Database column names appearing in error messages or API responses
- ORM lazy-loading triggering N+1 queries from places that don't look like data access

**Why it's a risk:** changes in the lower layer (switch databases, change API shape, upgrade framework) force changes throughout. Tests of business logic require setting up a database.

**Smallest safe improvement:** at one boundary, introduce a translation step — a small mapper from ORM type to domain type, or from HTTP request to a typed command. One boundary, not all of them. The audit value is identifying the worst leak.

### 4. Low module cohesion

Files or modules that do many unrelated things. The "utils.js" / "helpers.py" / "common/" problem. Cohesion is "everything in this module changes together for the same reason"; low cohesion is "this module changes whenever anything changes."

**Tells:**
- A `utils` / `helpers` / `common` directory or file with many unrelated exports
- Files where the imports come from very different parts of the codebase
- Classes named `XManager`, `XService`, `XHelper` that have grown to 1000+ lines
- Files whose name no longer describes their contents

**Why it's a risk:** every team ends up touching the same files (merge conflicts), and the file becomes a magnet for unrelated additions ("I'll just put it here for now").

**Smallest safe improvement:** for one bloated utility module, identify the 2–3 natural clusters of functions inside it. Propose splitting along those lines. Don't split yet — name the clusters.

### 5. Missing or misplaced abstractions

Concepts that exist implicitly in the code but aren't named. The opposite of premature abstraction — this is *delayed* abstraction.

**Tells:**
- The same 3-line sequence appearing in many places (loading something, checking permission, doing the operation)
- Multiple modules implementing the same pattern slightly differently (each has its own retry logic, its own caching, its own validation pipeline)
- Configuration that's "almost the same" across services with copy-paste drift
- The same nouns appearing in different forms in different places (`UserDTO`, `UserModel`, `UserRow`, `UserPayload`)

**Why it's a risk:** changes that *should* be in one place are in many. Inconsistencies accumulate.

**Smallest safe improvement:** name the missing abstraction and point to 3 places where it's currently inlined. Don't extract it yet — naming is the audit; extracting is the refactor.

### 6. Inappropriate intimacy between modules

Modules that reach into each other's internals — accessing private fields, knowing about each other's data shape in ways that aren't part of the contract, mutating each other's state.

**Tells:**
- Code accessing fields prefixed with `_` from outside the owning class/module
- Two modules that always change together in commits (read git history if available)
- Module A's tests requiring deep knowledge of module B's internals
- Helper functions that take an object and know which fields to read/write

**Smallest safe improvement:** for one such pair, identify what the *real* contract should be — what operations does A actually need from B? — and name it. Don't reshape the API; name what the API should expose.

## What to deprioritize

- Layer-naming bikeshedding (whether to call something "service" or "interactor")
- Pedantically clean architecture in places where the code is simple and stable
- Imposing structure for its own sake — if a small codebase works fine flat, leave it flat

## Common cross-references

- Dependency violations and abstraction leaks often co-occur with **maintainability** (domain leakage)
- Low cohesion often co-occurs with **change amplification** (shotgun surgery)
- Circular dependencies often co-occur with **testability** (hard to construct, hard to mock)
