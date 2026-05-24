---
name: control-plane-ux
description: >
  Design and build admin panels, settings surfaces, configuration UIs, dashboards
  for complex systems, and any interface that controls policy, routing,
  permissions, budget, memory, model choice, quotas, feature flags, secrets,
  webhooks, integrations, or automated decisions. Use this skill whenever the
  user asks for an admin UI, a settings page, a control panel, a configuration
  surface, a trace/log/audit viewer, a tool/API/memory/budget management view,
  or a "console" for an internal system — even if they don't use those words.
  Also trigger for requests like "show the state of my system", "let me configure
  X", "build a debugger for Y", "I need a way to manage Z", "an internal tool
  for our ops team", or anything that exposes the guts of a system to a human
  operator. This skill is about semantic clarity, decision transparency, and
  safe interaction patterns — not visual expressiveness. Use it alongside
  frontend-design when both apply: this skill governs information architecture
  and interaction semantics; frontend-design governs aesthetics. When the two
  conflict on structure, state representation, or safety affordances, this skill
  wins.
---

# Control-Plane UX Skill

This skill guides the design and implementation of **admin, settings, and
control-plane surfaces** — UIs where a human operator configures a system,
reviews automated decisions, manages policy, debugs behavior, or inspects state.

These surfaces require a different discipline than product UIs. The user is not
a customer browsing a catalog; they are an operator answering a question or
making a change with consequences. Control-plane UIs must **explain the system**,
not just expose controls. A control-plane surface that only renders forms has
failed at its real job.

## How to use this skill

This SKILL.md contains the always-relevant principles. Three reference files
contain material that's relevant only for specific surfaces or workflows. Load
them as needed:

- **`references/surface-patterns.md`** — Detailed layouts for specific surface types (API settings, tools, memory, budget, traces, feature flags, webhooks, secrets, permissions, health). Read the entry for the surface(s) you are building.
- **`references/worked-examples.md`** — Full compositions showing how the principles compose into real layouts (routing policy editor, budget enforcement console, trace viewer). Read at least one before designing your first control-plane surface.
- **`references/operator-workflows.md`** — Cross-cutting workflows that apply across surfaces: filters and saved views, role-based personalization, search, keyboard navigation, audit/changelog/diff, observability vs configuration boundaries. Read when the surface is heavy in any of these dimensions.

If you only have time for one reference file, read `worked-examples.md` — it
anchors the principles in concrete layouts.

---

## The Core Design Loop

For any control-plane surface, design around this five-step loop:

1. **What input is being configured, tested, or inspected?**
2. **Which parts are facts, constraints, and preferences?**
3. **What will the system do with those inputs?**
4. **Why did it make that decision?**
5. **What can the user safely change next?**

Every section of the UI should answer at least one of these questions. If a
section answers none of them, cut it. If it answers more than one, consider
splitting it.

A useful reframing: every operator arrives with a **question in mind** —
*"why was this request routed to the cheap model?"*, *"why is this user locked
out?"*, *"is my budget cap about to fire?"* — and the surface either answers
that question quickly or it doesn't. Forms-first design tends to optimize for
configuration and starve inspection. Resist that gravity.

---

## Information Architecture

### Tab and Section Naming
- Use **plain nouns**: `API`, `Config`, `Routing`, `Tools`, `Memory`, `Budget`, `Traces`
- Avoid repeated suffixes: not `Tool Management`, `Budget Management`, `Memory Management` — just `Tools`, `Budget`, `Memory`. The repetition adds noise without adding meaning.
- Put advanced controls in **Settings**, not primary workflow navigation.
- Reserve the top-level nav for surfaces the operator visits **often**. One-time setup goes one level down.

### Chronological Layout Order
Within any panel or section, present information in this sequence:
1. **Input** — what the user or system is providing
2. **Context** — relevant state or history
3. **Constraints** — hard limits and policy
4. **Preferences** — tunable options
5. **Result / Preview** — what the system will do or has done

Never reverse this order. Never interleave result blocks above input blocks. The
operator's eye should be able to trace a single downward path from cause to
effect.

### Conceptual Separation
These concepts must never look interchangeable. Distinguish them visually *and*
verbally:

| Concept | What it is | What it is **not** |
|---|---|---|
| Tool | A callable capability | A permission |
| Permission | Whether the tool **may** be called | Whether the tool **can** be called right now |
| Runtime availability | Whether the tool is reachable right now | A policy decision |
| Policy | A rule that governs behavior across calls | A one-time override |
| Override | A user-asserted exception to policy | A preference |
| Status | Current observed system state | A user-set value |
| Preference | A tunable setting with no hard enforcement | A constraint |

Conflating any pair on this list is a primary cause of operator confusion. A tool
that is *permitted but unavailable* must look different from a tool that is
*available but not permitted*, even though both end-states are "the tool will not
run."

---

## Visual Design Principles

### Density
- **Compact density, not loud density.** Dense UIs should rely on alignment, grouping, whitespace, and restrained borders — not color saturation or excessive borders.
- Reserve heavy visual weight (bold, color, badges) for things that genuinely need attention. If everything is highlighted, nothing is.
- A good test: squint at the screen. The things that remain visible should be the things that matter most.

### State Representation
Every selectable, configurable, or reportable item needs **five distinct visual states**:

| State | When to use |
|---|---|
| Selected / active | User has chosen this |
| Available | Can be selected |
| Disabled | Cannot be selected — and the UI must say **why** |
| Derived | Set automatically by the system, not by the user |
| Conflicting | Incompatible with another current selection |

Ordinary selectable chips must not look like status badges. Status badges are
reserved for actual observed system state. A common failure: making a green
"enabled" toggle look identical to a green "healthy" status pill. The operator
cannot tell whether they are *seeing* state or *setting* state.

### Help and Explanation
- Put deeper explanations behind a single predictable `?` affordance per field or section.
- Never duplicate explanation blocks across the same surface — the operator will doubt which copy is canonical.
- Keep visible copy **concrete and affirmative**: say what a field does and exactly how the system uses it, not what it is in abstract.
- Prefer "Requests over 8k tokens route to the long-context model" to "Configures long-context routing behavior."

---

## Interaction Patterns

### Preview Before Commit
Any surface that controls **policy, routing, permissions, money, memory, model choice, or anything that affects future automated decisions** must have a **preview/simulation mode** before it has a commit/apply mode.

The preview must answer all three:
- **What will happen** under the proposed configuration?
- **With what inputs** is this being evaluated?
- **Under what constraints** is the system operating?

A preview that shows only the new value of a field is not a preview — it is an
echo. A real preview projects the configuration against a representative input
and shows the resulting behavior.

### Safe Inspect Mode
Every advanced surface needs a **safe inspect mode** before it has a **commit/apply mode**. Users should be able to explore the system's state and decisions — pan through traces, expand records, test rules — without risk of applying changes. Treat the inspect mode as the default and the apply mode as the explicit detour.

### Destructive and Irreversible Actions
Destructive actions (delete, revoke, rotate, purge, force-reload, replay-in-production) require **friction proportional to blast radius**:

| Blast radius | Friction |
|---|---|
| Reversible, scoped to user | Single click, undoable toast |
| Reversible, affects others | Confirm dialog with named consequence |
| Irreversible, scoped | Confirm dialog + summary of what will be lost |
| Irreversible, affects others / production | Typed confirmation (the resource name) + summary + actor logging |

Never use a plain "Are you sure?" dialog for an irreversible production action.
The dialog must restate **what is being deleted**, **what depends on it**, and
**what cannot be recovered**. If the system can offer a dry-run or soft-delete
path, the dialog should surface it.

### Bulk Operations
Admin surfaces routinely operate on many items at once. Bulk operations need:

1. **Visible selection state** — the count and a way to review the selection before acting
2. **A heterogeneity warning** — if the selected items don't all support the action, say so before the click, not after
3. **Per-item result reporting** — never collapse 200 outcomes into one "Done." Show successes, partials, and failures with reasons
4. **Resumability** — if a bulk action partially fails, the user must be able to retry only the failures

### Result Panels
After an action or automated decision, results must answer:
- **What happened?** (the outcome)
- **Why?** (the rule, model, or path that produced it)
- **What input caused it?** (so the operator can reproduce or alter it)

A result panel that shows only an outcome forces the operator to context-switch
to logs to reconstruct the cause. That is the failure mode this skill exists to
prevent.

### Conflict Presentation
Never present conflicts as scary error states. Structure them in this order:
1. **The incompatible input** — what the user just set
2. **The violated constraint** — what rule or other setting it collides with, linked
3. **The next available fix** — a concrete action, not a generic "review your settings"

**Bad:** `Error: Configuration is invalid.`
**Better:** `"Streaming" requires "Beta features" to be enabled. → Enable beta features, or turn off streaming.`

---

## Override Controls

Overrides imply power. The UI must communicate four things for every override:

1. **Soft or hard?** — Is this a preference (can be superseded by other rules) or a hard requirement (enforced unconditionally)?
2. **Scope** — What does this affect? One request, one user, one tenant, globally?
3. **Other settings affected** — Which downstream behavior changes as a consequence?
4. **Active state** — Is the override currently in effect, or just configured?

Use distinct visual treatment for override controls — they should never look
like ordinary inputs. A reasonable convention: a left border accent, an "Override"
label, and a visible "revert to default" action that is always one click away.

**Bad:** A toggle labeled "Force model: gpt-4" sitting in a list of preferences.
**Better:** A bordered override card: *"Override active: forcing `gpt-4` for all requests in this session. Supersedes routing policy. [Revert]"*

---

## States Every Surface Owes the Operator

Every list, table, or panel must define behavior for:

- **Empty** — Not just "no data" but *why*: never configured, filtered out, recently cleared, or actually no events. Each warrants different copy and different next-action affordance.
- **Loading** — Distinguish initial load from refresh from background poll. Don't show skeleton screens for data the operator has already seen.
- **Partial** — Some data loaded, some failed. Show what loaded; don't hide it behind a blanket error.
- **Stale** — Data is older than the operator probably expects. Show the timestamp and offer refresh.
- **Error** — What failed, what was tried, what the operator can do. Never just "Something went wrong."
- **Forbidden** — The operator is authenticated but not authorized. Say what role or permission is required and who can grant it.

---

## Anti-Patterns

These are the most common failure modes. Avoid them:

- **Forms with no preview.** A configuration form that commits on save with no projection of what will change. The operator is forced to apply-then-observe.
- **Status pills that are also buttons.** Visual ambiguity between *displayed state* and *selectable option*. Pick one role per control.
- **Toast-only feedback for consequential actions.** A 4-second toast is not adequate confirmation that a destructive bulk operation succeeded.
- **Hidden derivation.** A field is auto-computed from other fields but looks user-editable. Either show it as derived or let the user edit it directly with a clear override.
- **Mystery disabled controls.** A greyed-out button with no tooltip explaining the prerequisite. Disabled state must always include a reason.
- **Modal stacking.** A confirm dialog opening another confirm dialog. Collapse the decision into one informed prompt.
- **"Advanced" as a dumping ground.** A section labeled Advanced that contains both genuinely advanced controls and ordinary settings the designer didn't want to surface. Hide things for a reason; if you can't articulate the reason, don't hide them.
- **Save buttons that are sometimes Apply.** If "Save" sometimes means "Save and apply now" and sometimes means "Save for next reload", split the buttons.
- **Showing the system's internal IDs as the primary identifier.** Operators don't think in UUIDs. Show the human name and put the ID behind a copy affordance.
- **One-shot validation.** Validating on form submit instead of as the operator types. Catch the error at the field, not the form.

---

## Surface Pattern Index

For detailed layouts of each surface type, see `references/surface-patterns.md`.
Quick index:

| Surface | One-line shape |
|---|---|
| API settings | Key state → Config → Validation result |
| Config / Runtime | Current inputs → Active constraints → Reload preview |
| Tools / Integrations | Available \| Enabled \| Permitted \| Reachable (four distinct columns) |
| Memory | Stored → Retrieval policy → Previewed recall → Actual injected |
| Budget / Quota | Limits \| Spend \| Enforcement \| Projection |
| Traces / Logs / Audit | Raw events → Decision path → Actionable diagnosis |
| Feature flags | Definition → Targeting → Current exposure → Recent flips |
| Webhooks | Endpoint → Events → Recent deliveries → Retry policy |
| Secrets | Identity → Where used → Last rotated → Rotate action |
| Permissions | Subject → Resource → Effective → Source |
| Health / Status | Measure → Value → Threshold → Trend |

For cross-cutting workflows (filtering, saved views, role personalization, audit
history, search), see `references/operator-workflows.md`.

---

## Reusable Rules

> If a surface controls **policy, routing, permissions, money, memory, model choice, or any automated decision the operator will later have to debug**, it needs a **preview/explanation panel**, not just forms.
>
> If a surface reports **what the system did**, it needs a **decision-path view**, not just a log.

---

## Relationship to any `frontend-design` Skills

This skill handles **information architecture and interaction semantics**.
`frontend-design` handles **aesthetics, typography, color, and motion**.

When building a control-plane UI artifact, use both:
1. Read this skill first to establish structure, hierarchy, and interaction patterns.
2. Then apply user-preferred `frontend-design` skill guidance for visual execution — while respecting the semantic constraints here (e.g., don't use bold color on ordinary chips just because it looks good).

When the two skills conflict, **this skill takes precedence** on:
- State representation (the five states above)
- Section order (the chronological layout)
- Explanation placement (single `?` per field, no duplication)
- Preview-before-commit requirements
- Destructive-action friction
- Conflict and error presentation

`frontend-design` retains precedence on typography, palette, spacing scale,
motion, and iconography — within the semantic guardrails above.
