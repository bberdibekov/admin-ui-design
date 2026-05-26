# Testability Audit

Focus: code that is hard to test in isolation, fast, and reliably. Untestable code is a maintainability problem in disguise — it means changes have to be verified by running the whole system, which slows everyone down and lets bugs through.

Use the finding format and ranking rules from SKILL.md.

## What to look for

### 1. Inline side effects

Business logic that directly performs I/O, time reads, randomness, or process-level effects rather than receiving them as inputs.

**Tells:**
- `Date.now()`, `new Date()`, `performance.now()` inside non-trivial logic (not just timestamps for logging)
- `Math.random()`, `crypto.randomUUID()` inline in a function whose output should be deterministic in tests
- `fetch`, `axios`, HTTP clients called directly from domain code
- `fs.readFile`, `fs.writeFile`, database clients used inside what should be pure logic
- `console.log` / structured logger calls scattered through hot paths (less critical, but a tell)
- `process.env.X` read inside business logic rather than at startup

**Why it's a risk:** every test that exercises this code has to either mock these globals (fragile, leaks across tests) or accept nondeterminism (flaky tests). Both options degrade the test suite.

**Smallest safe improvement:** at the entry point of the affected function or class, accept the time/random/IO source as a parameter (with a sensible default). Migrate one caller — usually the test setup — to pass an injected version. This is often a tiny change with big payoff.

**What makes this unsafe:** if the function is widely called and the default behavior must not change, adding a parameter (even optional) can break callers in surprising ways in dynamically typed languages or via reflection.

### 2. Hard-to-construct objects

Classes or functions that require an elaborate setup to instantiate — long constructors, many required dependencies, deep mocking chains in the existing tests.

**Tells:**
- Constructors with 6+ parameters
- Test files where `beforeEach` is 30+ lines of mock setup
- "God objects" — central classes that everything else depends on, requiring half the system to construct
- Dependency-injection containers being used in test setup (a sign the manual version was too painful)

**Why it's a risk:** writing a new test for this code requires re-doing the setup, so people don't. Coverage suffers exactly where it's most needed (complex code).

**Smallest safe improvement:** identify *which* dependencies are actually used by the unit under test. Often the constructor takes ten things and the method under test uses three. Introduce a smaller constructor or a builder for tests; don't restructure production code yet.

### 3. Test-to-code coupling

Tests that assert on internals (private methods, internal data structures, call sequences) rather than observable behavior. Brittle to refactor, doesn't actually validate the contract.

**Tells:**
- Heavy use of `spy` / `verify` / `toHaveBeenCalledWith` on internal collaborators
- Tests that break whenever the implementation is refactored, even when behavior is unchanged
- Tests reaching into private fields (via reflection, `@ts-ignore`, or language-specific tricks)
- Tests that mirror the structure of the code 1:1 (a test per method rather than per behavior)

**Why it's a risk:** the test suite becomes a tax on change instead of a safety net for it. People stop refactoring because they don't want to fix 40 tests.

**Smallest safe improvement:** for one over-coupled test, rewrite it to assert on the observable output (return value, persisted state, message emitted) rather than on the call sequence. This shows the pattern; don't rewrite the whole suite in one pass.

### 4. Hidden dependencies (global / module state)

Functions whose behavior depends on state that isn't in their signature — singletons, module-level variables, environment variables read at import time, mutable globals.

**Tells:**
- Module-level `let` / `var` that mutates
- Singleton patterns (`getInstance()`) called from inside business logic
- Imports with side effects (the import itself registers handlers, sets globals)
- Tests that pass alone but fail when run with others (order-dependent — a classic symptom)
- `beforeEach` resetting state that "shouldn't" need resetting

**Why it's a risk:** tests interfere with each other, behavior depends on import order, and reasoning about a function requires reasoning about the whole module.

**Smallest safe improvement:** for the worst offender, scope the state to an object that callers construct, or expose a reset function for tests as a stopgap. Migrating away from singletons fully is a refactor; just make it testable in isolation first.

### 5. Untested or under-tested critical paths

Files or functions with no test coverage where the code is non-trivial and clearly important. The audit isn't a coverage report — focus on places where the *absence* of tests is most surprising or most risky.

**Tells:**
- Domain model files (the core nouns) with no corresponding test file
- Complex pure functions (parsers, calculators, state machines) with no tests
- Bug fix commits whose diff includes no test changes (read recent git history if available)

**Smallest safe improvement:** identify the single highest-value function in the untested set and propose 2–3 specific test cases (happy path, one edge case, one error path). Don't write them — point to them.

### 6. Tests that are slow, flaky, or skipped

Existing tests marked `.skip`, `.todo`, `xit`, or with conditional skips. Tests with long timeouts. Tests that retry on failure. These are signs the suite has known weak spots that the team has stopped trying to fix.

**Tells:**
- `it.skip` / `xit` / `@Ignore` annotations
- Retry decorators or wrappers on tests
- Tests with `setTimeout`-based waits longer than ~100ms
- CI configs that allow N% test failure

**Smallest safe improvement:** list the skipped/flaky tests in the finding. Either fix or delete — a permanently-skipped test is worse than no test (it's a lie about coverage).

## What to deprioritize

- Coverage percentages as a target (the metric is gameable; what matters is which code is untested)
- Snapshot tests that are stable and easy to update (controversial but generally fine in moderation)
- Integration tests that are slow because integration tests are slow (only flag if they're slow *and* doing what a unit test should do)

## Common cross-references

- Inline side effects often co-occur with **structure** issues (the layer doing IO shouldn't have business logic)
- Hard-to-construct objects often signal **change amplification** (one class touches too much)
- Hidden dependencies often signal **correctness risk** (implicit invariants across calls)
