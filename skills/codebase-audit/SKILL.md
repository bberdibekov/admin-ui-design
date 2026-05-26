---
name: codebase-audit
description: Audit a codebase for maintainability, testability, structural, change-amplification, or correctness-risk issues — either a broad sweep across all dimensions or a focused deep-dive on one. Use this skill whenever the user asks to audit, review, assess, evaluate, or critique a codebase, module, or directory for code quality, maintainability, design issues, technical debt, refactoring opportunities, or "what's wrong with this code" — even if they don't use the word "audit." Also use when they ask for code smells, anti-patterns, structural problems, hotspots, or what to clean up. Do not use for line-by-line code review of a single small change (use normal review for that), bug hunting in specific failing code, or performance profiling.
---

# Codebase Audit

A structured audit workflow that produces ranked, evidence-backed findings without refactoring. Two modes share the same principles:

- **Broad sweep** — wide and shallow across all dimensions, surfaces hotspots, recommends where to go deep.
- **Focused** — one dimension, deep, with concrete evidence and smallest-safe-improvement per finding.

The principles below apply to both modes. The reference files in `references/` define how each mode/dimension actually executes.

---

## Step 1: Pick the mode

Read the user's request and decide:

- **Broad sweep** if the request is general ("audit my codebase", "what's wrong with this code", "review this repo for maintainability"). When in doubt, default here — it orients the user and the next focused pass is one prompt away.
- **Focused** if the user names a dimension or follows up on a broad sweep ("audit testability", "go deep on the domain leakage findings", "look at error handling").

Then load the matching reference file (and only that one):

| Mode / dimension | Reference file |
|---|---|
| Broad sweep | `references/broad-sweep.md` |
| Maintainability | `references/maintainability.md` |
| Testability | `references/testability.md` |
| Structure / architecture | `references/structure.md` |
| Change amplification | `references/change-amplification.md` |
| Correctness risk | `references/correctness-risk.md` |

If the user names a dimension that doesn't cleanly fit one of these (e.g. "security", "performance"), say so — this skill doesn't cover it well, and recommend a real security/perf review instead of forcing it.

---

## Step 2: Establish scope before auditing

A scopeless audit on a large codebase produces a shallow sweep that helps nobody. Before doing real work:

1. **If the user gave scope** (named directories, files, or a constraint like "files changed in the last 90 days"), use it.
2. **If no scope was given and the codebase is small** (under ~50 files / ~5k lines), audit all of it.
3. **If no scope was given and the codebase is large**, propose a scope and confirm before continuing. Good defaults:
   - The directory most central to the domain (often where the domain models live).
   - Files changed most often or most recently (real hotspots, where audit findings have the highest payoff).
   - A specific subsystem the user has mentioned in passing.

   Offer 2–3 concrete scope options rather than asking "what should I look at?" — the user often doesn't know either.

Audits without scope discipline waste tokens and produce findings the user can't act on.

---

## Step 3: Gather evidence, then write findings

The reference file for the chosen mode/dimension describes *what to look for*. This section governs *how to report it*.

### Every finding must include

Each finding is a small block of prose with these elements present (ordering and exact phrasing can adapt to the finding):

- **Title** — short, names the problem, not the solution. "Order state machine duplicated across services," not "Extract order state machine."
- **Location(s)** — file path(s) and line range(s). If a pattern repeats, list at least 2–3 examples and indicate how many total instances exist.
- **Evidence** — the smallest snippet that demonstrates the issue, quoted. If it's a structural finding (e.g. circular dependency), describe the structure precisely instead of quoting.
- **Why it's a risk** — what specifically goes wrong: bugs slip through, changes ripple, tests get flaky, onboarding slows. Be concrete, not abstract ("violates SRP" is not a risk).
- **Smallest safe improvement** — the minimum change that meaningfully reduces the risk. Not the ideal refactor — the smallest step in the right direction. If the smallest safe step is "do nothing yet, but note it before the next change to X," say that.
- **What would make this improvement unsafe** — hidden coupling, callers you can't see, runtime assumptions, anything that means "don't just do it." If nothing comes to mind, say "no obvious blockers" — but actually look.
- **Risk** — H / M / L. Risk = severity × reach × likelihood-of-biting. A medium problem in 30 files outranks a severe one in 1 obscure file.
- **Effort** — S / M / L for the smallest safe improvement.

### Rank findings, don't just list them

Sort by **Risk × inverse Effort** — high-risk, low-effort findings first (quick wins), then high-risk-high-effort (structural work to plan for), then lower-risk items grouped at the end. The user should be able to read top-down and stop whenever they've seen enough.

### Don't refactor

This skill produces findings, not code changes. If the user asks "and can you fix it?", finish the audit first and then offer to do the fix as a separate step. Mixing audit and refactor in the same pass produces neither well.

### Don't pad

If only three real findings exist, return three. Inventing weak findings to hit a number trains the user to ignore the output. Better to end with "no other significant issues in this dimension within the scoped area" than to list nitpicks.

---

## Step 4: Close with next steps

After the findings, end with a short section the user can act on:

- **For broad sweeps**: name the 1–2 dimensions where the strongest signals appeared and recommend the focused audit to run next (e.g. "the change-amplification signals here are strong — `codebase-audit references/change-amplification.md` would be the next deep-dive").
- **For focused audits**: if findings cluster around a specific subsystem, recommend scoping the next audit there. If you noticed signals in a *different* dimension while auditing this one, mention them briefly so they're not lost — but don't audit them; that's a different pass.

---

## What this skill is not

- Not a security review. Don't try to assess auth, crypto, injection risks, or supply-chain issues here — recommend a real security audit if those come up.
- Not a performance review. Hotspot identification for perf needs profiling data, not static reading.
- Not a style review. Formatting, naming consistency-as-style, and lint-level issues are noise in an audit at this level; mention them only if they're symptomatic of something deeper.
- Not a refactor. See Step 3.
