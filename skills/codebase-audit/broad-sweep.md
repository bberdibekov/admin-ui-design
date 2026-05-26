# Broad Sweep

A wide, shallow pass across all five audit dimensions. The goal is **orientation, not depth** — you're producing a map the user can use to decide where to invest a focused audit next.

## How a broad sweep differs from a focused audit

- **Coverage over depth.** Touch every dimension. 1–3 findings per dimension is plenty; 5+ means you're going too deep.
- **Signals over conclusions.** "Possible domain leakage in `billing/`" is fine in a broad sweep; the focused pass confirms or rejects it.
- **Hotspot identification.** Note files/directories that show up across multiple dimensions — those are the real audit targets.
- **Less evidence per finding is acceptable** — one example per finding is enough. Focused passes do the multi-example pattern work.

The output format rules from SKILL.md still apply (title, location, evidence, risk, effort, smallest safe improvement). Just expect the snippets to be shorter and the "what would make this unsafe" sections briefer.

## How to execute

1. **Map the codebase first.** Read the top-level structure (`ls`, `tree`, or equivalent). Identify the major directories, the entry points, and roughly what each subsystem does. Note: language, framework, testing setup, package boundaries.

2. **Read the highest-signal files.** You can't read everything. Prioritize:
   - Entry points (main, server setup, route definitions)
   - Domain model files (whatever holds the core nouns of the business)
   - The 5–10 largest files (size is a smell on its own and often a hotspot)
   - Any file with "util", "helper", "manager", "service", "common" in the name (these tend to accumulate the worst issues)
   - Test files for the same modules — testing pain is a tell

3. **Scan for each dimension's top smells** (see below). Don't go deep — you're looking for the signal that says "there's something here worth a focused look."

4. **Cross-reference.** If the same file or directory keeps coming up across dimensions, that's the audit target, not a finding — call it out explicitly.

## What to look for in each dimension (quick signals only)

For each, you're scanning for **the top 2–3 cheapest-to-spot smells**. The focused audit files have the full list — don't try to recreate them here.

### Maintainability
- Long functions / files (>300 lines, >50-line functions)
- Repeated conditional blocks (the same `if (status === ...)` chain in multiple places)
- Domain concepts represented as raw strings/numbers everywhere
- Comments explaining *what* the code does (means the code itself isn't saying it)

### Testability
- Direct calls to `Date.now()`, `Math.random()`, `fetch`, `fs`, DB clients inside business logic
- Constructors that do work (not just assignment)
- Files with no corresponding test file, or test files that mock heavily

### Structure
- Imports going "the wrong way" (low-level importing high-level, or sibling subsystems reaching across)
- Files importing from many different layers
- `index.js` / barrel files that re-export everything (often hides circular deps)
- A `utils/` or `common/` directory that everything imports from

### Change amplification
- Enum-like string sets defined or matched in multiple places
- Feature flags / config branches scattered through code
- Domain IDs typed as plain strings/numbers
- Multiple files that all need editing for a typical feature (read recent PRs/commits if available)

### Correctness risk
- Empty catch blocks, or catches that just log
- Mix of throw / return null / return Result in the same module
- Functions that mutate their arguments
- Trust boundaries (request handlers, message consumers) without obvious validation

## How to structure the broad-sweep output

1. **One-paragraph orientation** — what kind of codebase this is, its rough shape, the scope you audited.
2. **Findings grouped by dimension**, each with 1–3 items. Use the standard finding format from SKILL.md but keep each item compact.
3. **Hotspot summary** — directories or files that surfaced in multiple dimensions. These are the prime targets for a focused audit.
4. **Recommended next step** — which 1–2 focused audits would have the highest payoff, and on what scope. Be specific: "Run a focused testability audit scoped to `src/billing/` — it surfaced in 3 of 5 dimensions and the test setup there looks heavy."

## When to stop

A broad sweep should produce roughly 8–15 findings total across all dimensions. If you're past 20, you're not doing a sweep anymore — you're doing five shallow focused audits at once, which helps nobody. Trim to the highest-signal items per dimension.
