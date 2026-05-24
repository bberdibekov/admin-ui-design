# Worked Examples

Full compositions showing how the principles in `SKILL.md` come together in
real layouts. Each example highlights:

- **What the surface is for**
- **The composition** (top to bottom, left to right)
- **What is deliberately absent** — often as instructive as what is present
- **The principles in action** — which `SKILL.md` rules each part embodies

Read at least one of these before designing your first control-plane surface.
They make the principles concrete in a way the rules alone cannot.

---

## Table of Contents

1. [Routing Policy Editor (LLM gateway)](#routing-policy-editor-llm-gateway)
2. [Budget Enforcement Console](#budget-enforcement-console)
3. [Trace Viewer with Decision Path](#trace-viewer-with-decision-path)

---

## Routing Policy Editor (LLM gateway)

**What it is for:** An operator configures how incoming requests are routed to
different models based on content, cost, latency, and tenant policy. The
operator needs to author rules, see how they interact, and confirm what will
happen before applying changes to production traffic.

### Composition

1. **Header**
   Plain noun (`Routing`). No suffix like "Routing Management." No icon-as-title.

2. **Input panel (top)**
   The policy rules the operator is editing. Each rule is a card with all five
   visual states defined: selected, available, disabled (with reason),
   derived (e.g., "this rule was auto-generated from tenant tier"), conflicting.
   Rules can be reordered by drag — ordering is part of the semantics.

3. **Context strip (just below the input panel)**
   `This policy applies to: production tenant. Last edited 2 days ago by alex@.
   3 rules currently active. 12 rules total.`
   One row, restrained typography. Answers "where am I and what is this?" in a
   glance.

4. **Constraints panel (right rail)**
   Hard limits this policy cannot violate: provider rate limits, tenant tier
   maximums, compliance rules. These are read-only and visually distinct from
   editable rules. The operator cannot edit these here; they are linked to
   their own surfaces. The right-rail position signals "context, not control."

5. **Preview panel (bottom, primary real estate)**
   A sample request → routing decision trace. The operator can edit the sample
   request inline and watch the trace update as they type. Each decision step
   shows:
   - Which rule fired
   - Why (the matching condition)
   - What the alternative would have been

   The preview is the **default view** — visible without any "enable preview"
   toggle. The operator should be debugging by default.

6. **Conflict surface (inline, contextual)**
   If a new rule would shadow an existing one, a conflict card appears in the
   preview panel:
   > **This rule never fires** — Rule #2 ("all requests") matches first.
   > → Reorder rules, or scope this rule more narrowly.

   Note the three parts: the incompatible input, the violated constraint, the
   next available fix. No scary red icon. No modal.

7. **Apply controls (bottom right, last in reading order)**
   - **Dry-run** button, large, default focus. Runs the current draft against the
     last 1,000 production requests and shows what would change.
   - **Apply** button, smaller, with a confirm dialog that summarizes the diff
     (rules added, removed, reordered) and requires the operator to type the
     tenant name for production tenants.

### What is deliberately absent

- **No "Save" button at the top of the form.** Save and Apply are different
  things; the surface uses Draft / Dry-run / Apply instead.
- **No "Settings" tab.** This is a primary workflow, not advanced configuration.
- **No green badges on editable rules.** Green pills are reserved for actual
  health state, not "this rule is configured."
- **No modal on field change.** The preview updates inline; there's no need for
  a modal.
- **No separate "Test" page.** Preview *is* test. Bifurcating them creates two
  places to look for one piece of information.

### Principles in action

- **Core loop:** the layout walks the operator through input → context →
  constraints → preview → result, in that order.
- **Preview before commit:** the preview is the default state of the screen,
  not a feature behind a toggle.
- **Conflict presentation:** structured (input, constraint, fix), not error-styled.
- **Destructive action friction:** typed confirmation for production apply.
- **State representation:** five states defined for rule cards; status badges
  are not used here.

---

## Budget Enforcement Console

**What it is for:** A finance-ops operator monitors spend across tenants and
configures enforcement policy. They need to answer "are we about to overspend
somewhere?" and "what will happen if we do?" — and to act before either becomes
an incident.

### Composition

1. **Header**
   `Budget`. No suffix.

2. **Tenant selector + projection summary (top, full width)**
   Tenant dropdown on the left. To the right, a single line:
   > Production: **$847 / $2,000 monthly cap.** At current rate, soft cap fires
   > in **6 days**, hard cap in **18 days**.

   The projection is more prominent than the snapshot, because it is more
   useful. Note the bolded values are *not* color-coded — color is reserved for
   when something is actually wrong.

3. **Spend breakdown (left two-thirds, top)**
   A table sorted by descending contribution: top users, top tools, top model
   tiers. Each row shows count, total spend, and a sparkline of the last 14
   days. The operator's question "what is driving the spend" is answerable
   without scrolling.

4. **Enforcement policy panel (right one-third, top)**
   Two clearly distinct sections:
   - **Soft cap behavior:** notify, continue serving. Configurable: who to notify, at what threshold.
   - **Hard cap behavior:** block new requests, allow in-flight to complete. Configurable: error message, fallback behavior.

   Soft and hard caps are visually distinct (different left border accents, different label colors). The operator must never confuse "notify and keep going" with "block."

5. **Projection / what-if panel (bottom, full width)**
   An editable "if I change the cap to X, here is what would happen" surface.
   The operator drags a slider; the projection updates. Shows:
   - Date soft cap would fire
   - Date hard cap would fire
   - Affected users (count, sample list)
   - Affected request volume (estimate)

6. **Apply controls (bottom right)**
   - **Save draft** (no enforcement change)
   - **Apply with effective date** (operator picks when the change takes effect; default is "next billing period," not "now")

   No "Apply immediately" as the default — budget changes are rarely actually
   urgent, and the default should reflect that.

### What is deliberately absent

- **No big "$847" hero number.** That number, in isolation, is not useful. It
  appears within the projection sentence where it has context.
- **No red color on the snapshot.** Spend is at 42% of cap; nothing is wrong;
  no warning state is appropriate. Red is reserved for actual breach or
  imminent breach.
- **No "Reset budget" button.** That is a separate, rarely-used,
  high-consequence action; it lives in a less prominent location with full
  destructive-action friction.

### Principles in action

- **Result panels:** projection answers what, why, and from what inputs.
- **Override controls:** soft vs hard cap visually distinct, scope clear,
  active state shown.
- **Density:** compact, restrained color; weight reserved for the projection
  numbers.
- **Anti-pattern avoided:** no toast-only feedback — budget changes flow
  through a draft → effective-date → apply sequence.

---

## Trace Viewer with Decision Path

**What it is for:** When something looked wrong, an operator opens a trace
viewer to reconstruct what the system did and why. The operator's job is
diagnosis, not browsing. The viewer must compress the raw event stream into a
decision path and surface candidate fixes.

### Composition

1. **Header**
   `Traces`. Just that.

2. **Filter bar (top)**
   The most-used filters are surfaced as chips: time range, decision type
   (e.g., "fell back to default model"), latency bucket, tenant, error
   presence. Free-text search to the right for everything else. Filter state
   is preserved across pagination and across sessions (saved as a view).

3. **Trace list (left rail)**
   Each row shows: timestamp, request summary (5–8 words), decision path
   summary (2–3 words like "routed → tool failed → fallback"), latency,
   outcome. Color is used **only** for outcome (error vs success), nowhere
   else. The operator scans the decision-path column to find pattern
   candidates.

4. **Selected trace, layer 1 (right pane, top)**
   Raw events: timestamps, IDs, tool invocations, latencies, payloads. Expandable; collapsed by default to avoid drowning the operator in JSON.

5. **Selected trace, layer 2 (right pane, middle — the headline)**
   Decision path, rendered as a vertical sequence:
   > 1. Request matched policy `long-context-override` (rule #4)
   > 2. Routed to model `claude-opus-4` per that rule
   > 3. Tool call to `search` timed out after 5,000ms (limit: 5,000ms)
   > 4. Fallback engaged: returned to default routing
   > 5. Re-routed to model `claude-sonnet-4`
   > 6. Response returned, total latency 8.4s

   Each step links to the relevant config surface (the rule, the timeout
   setting, the fallback policy). This is the layer the operator spends
   most of their time in.

6. **Selected trace, layer 3 (right pane, bottom)**
   Actionable diagnosis:
   > **Likely cause:** the search tool timed out, which is the proximate cause
   > of the fallback and the 8.4s latency.
   >
   > **What to consider:**
   > - Increase `search` tool timeout: currently 5,000ms → Settings · Routing · Matcher timeout
   > - Investigate why `search` is slow: 12 similar timeouts in the last hour → Filter neighboring traces
   >
   > **Pattern in last hour:** 12 traces with the same decision path. → View pattern

   Concrete, linked, ranked. The operator can act without leaving the screen.

7. **Neighborhood view (collapsed by default, expandable)**
   The 10 traces immediately before and after the selected one, summarized.
   Patterns emerge in context; isolation hides them.

### What is deliberately absent

- **No raw JSON as the default view.** That's layer 1; it's collapsed. The
  default is layer 2 (decision path).
- **No "Export" button as the primary action.** Operators are debugging here,
  not preparing reports.
- **No big red error banner.** The outcome is one column in the list; the
  detail pane shows what happened and what to do. Banners are noise.
- **No separate "Logs" tab.** Logs are layer 1 of trace inspection; surfacing
  them as a separate destination splits operator attention.

### Principles in action

- **Three-layer decomposition** of trace, log, audit: raw → interpreted → diagnostic.
- **Result panel rule:** answers what, why, what input caused it.
- **State representation:** color used only for outcome, not for everything.
- **Anti-pattern avoided:** no JSON-pretty-printer-disguised-as-viewer.
- **Operator question framing:** the surface is designed to answer "what
  happened to this request and what do I change?" — not to display events.

---

## Composing your own

When you need to design a control-plane surface not covered by an existing
worked example, walk through these checks before you start drawing:

1. **What question is the operator arriving with?** Write it down. Every panel
   should help answer it.
2. **What are the inputs, the constraints, the preferences?** Sort each field
   into one of those buckets. If a field doesn't fit any, it probably doesn't
   belong on this surface.
3. **What is the decision the system will make from these inputs?** That
   decision is the preview/result panel. Design it before you design the form.
4. **What can go wrong, and how will the operator find out?** That answer
   defines your states (empty, error, partial, conflict) and your trace path.
5. **What is the most destructive thing the operator can do here?** That
   action's friction defines the lower bound of friction for every other
   action on the surface.

If you cannot answer all five before drawing pixels, the surface is not yet
designed.
