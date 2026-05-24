# Operator Workflows

Cross-cutting workflows that apply across control-plane surfaces. Unlike the
surface patterns in `surface-patterns.md`, these aren't specific to one screen —
they're things operators do *across* screens: filtering, finding, personalizing,
auditing, navigating with the keyboard.

Read this when the surface you're building leans heavily on any of these
dimensions, or when you're working on the overall shell of a control-plane
product.

---

## Table of Contents

1. [Filters and Saved Views](#filters-and-saved-views)
2. [Search and Resource Resolution](#search-and-resource-resolution)
3. [Role-Based Personalization](#role-based-personalization)
4. [Keyboard Navigation and Power-User Affordances](#keyboard-navigation-and-power-user-affordances)
5. [Audit, Changelog, and Diff](#audit-changelog-and-diff)
6. [Observability vs Configuration](#observability-vs-configuration)
7. [Icons, Tooltips, and Text Discipline](#icons-tooltips-and-text-discipline)

---

## Filters and Saved Views

Operators live in filters. A trace viewer, a permissions table, a feature flag
list, a webhook delivery log — all of them are functionally useless without
robust filtering. Filters are not a feature; they are the primary interaction.

### Requirements

- **Filter state is part of the URL.** An operator must be able to send a link
  that opens the exact filtered view they were looking at. Filter state stored
  only in local component state breaks collaboration.
- **Filter state persists across pagination and navigation.** Losing filters on
  page change is one of the most common — and most destructive — failure modes
  in admin UIs.
- **Filters compose, and the composition is visible.** When five filters are
  active, the operator must see all five at a glance, with one-click removal
  for each.
- **Empty results from filtering look different from "no data."** "No traces
  match your filters" plus a "clear filters" affordance is different from "no
  traces in the last 24 hours." Conflating them sends the operator down the
  wrong diagnostic path.

### Saved Views

When a filter combination is used repeatedly, the operator needs to save it.
Patterns:

- **Personal saved views** (per operator): the operator's working set. Quick
  switcher in the UI; not shared.
- **Team saved views** (per-tenant, per-team): canonical views that codify how
  the team thinks about the data ("all failing webhooks in production," "all
  flags at less than 5% rollout"). Editable by team members; with attribution.
- **Default view**: the one the surface opens to. Sensible defaults reduce the
  cognitive load of arriving at the page.

A saved view is more than a filter combination — it also captures column
selection, sort order, and grouping. All three are part of "how the operator
wants to see this data."

### What not to do

- Don't bury filters under "More" or "Advanced." If you have to hide them, your
  default view is wrong.
- Don't auto-apply filters as the operator types in a free-text search field
  *and* require a button press for chip filters. Inconsistent commit
  semantics across filter types is disorienting.
- Don't lose saved views when the underlying schema changes. If a column is
  renamed or removed, migrate the saved view; don't silently break it.

---

## Search and Resource Resolution

Operators arrive with names, not IDs. "I need to look at the webhook for the
billing service" — not "I need to look at webhook `wh_a8c4e9...`." Search must
resolve human references to system resources.

### Requirements

- **Global search** (typically `/` or `⌘K`): a single entry point that searches
  across all resource types — users, tenants, webhooks, flags, traces, secrets.
  Results grouped by type, with the resource type visible in each result.
- **In-context search**: every list view has its own filter that searches *just
  that list*. The operator may not want to leave the page to find what they're
  looking for.
- **Search by name, ID, and recent identifiers.** Operators sometimes paste an
  ID from a log or an email. Sometimes they type a name. Sometimes they
  remember the first three letters. All three must work.
- **Recently viewed** as a fallback. When search yields nothing, "you recently
  viewed these" gives the operator a way back to their working set.

### Resolution and disambiguation

If a search term matches multiple resources of different types, show all of
them — don't guess. If it matches multiple resources of the same type, show
all matches with disambiguating context (tenant, environment, last-modified).

### What not to do

- Don't make global search modal-blocking with no escape. The operator may
  search, realize they need to look at the page they were on, and want to
  dismiss. `Esc` must always work.
- Don't show results sorted purely alphabetically. Sort by recency of use or
  relevance; alphabetical sort is almost always wrong for operator workflows.

---

## Role-Based Personalization

A control-plane product is rarely used by one role. An SRE, a finance ops
person, a customer support agent, and a platform engineer may all open the
same console with different intent. The surface should accommodate that
without fragmenting into separate products.

### Patterns that work

- **Saved views per role.** Ship default saved views named for common roles
  ("Finance: budget tracking," "SRE: error rate, p99 latency, recent flips").
  Operators discover the workflow as much as the data.
- **Permission-aware UI.** Hide controls the operator cannot use, but make
  *visible* the fact that they're hidden. "Restricted controls available to
  the admin role — request access" is better than silently omitting them.
- **Default landing per role.** If you can infer the operator's role, land
  them on the surface they're most likely to need. A finance ops user
  should not land on the trace viewer.

### Patterns that don't work

- **Full layout customization** (drag-to-reorder widgets, hide/show anything).
  This is a frequent ask but usually a trap: it shifts maintenance burden
  to operators, fragments support ("which layout were you on?"), and rarely
  improves outcomes over good defaults.
- **Per-role separate apps.** Operators move between roles — an SRE
  investigating an incident may need to look at finance impact. Separate
  apps make this hard.

### What not to do

- Don't surface the same control twice for two different roles, with subtly
  different semantics. Pick the canonical control and link to it from both
  contexts.
- Don't gate observability surfaces behind role. Anyone with access to the
  product should be able to *see* state; gating applies to *changing* it.

---

## Keyboard Navigation and Power-User Affordances

Operators are power users. They keep the console open all day. Shipping a
control plane that requires mouse-and-click for every action is friction the
operator pays many times per hour.

### Baseline keyboard support

- **`⌘K` / `Ctrl+K`** — global search / command palette
- **`/`** — focus the current page's primary filter
- **`j` / `k`** — next / previous in a list
- **`Enter`** — open selected
- **`Esc`** — dismiss modal, clear focus
- **`g` + letter** — go to (e.g., `g t` for traces, `g b` for budget)
- **`?`** — show the keyboard shortcut overlay

These exact bindings aren't sacred; the principle is that an operator should be
able to navigate the console without the mouse. Pick a convention and apply it
consistently.

### Command palette

For power-user products, a command palette (`⌘K`) that searches across:

- Pages / surfaces ("Routing settings")
- Resources ("user alex@", "webhook wh_...")
- Recent actions ("last 5 things I did")
- Quick actions ("Create a new flag", "Pause webhook xyz")

Is the single highest-leverage power-user affordance. It collapses navigation,
search, and action invocation into one surface.

### What not to do

- Don't ship keyboard shortcuts that conflict with browser shortcuts
  (`⌘W`, `⌘T`, `⌘R`). Operators close the wrong tab once and never trust the
  shortcut again.
- Don't ship inconsistent shortcuts across surfaces. If `j`/`k` moves through
  a list on one page and does nothing on another, it's worse than no shortcut.
- Don't hide the shortcut overlay. `?` should bring it up from anywhere.

---

## Audit, Changelog, and Diff

Control-plane surfaces change. Operators need to answer "who changed what,
when, and why" — sometimes post-incident, sometimes routinely. The audit
trail is a first-class feature, not an afterthought.

### Required dimensions

Every change to a configurable resource should record:

- **Actor** — the human or service principal that made the change
- **Timestamp** — UTC, with timezone hint for the viewer
- **Resource** — the specific object changed, linked
- **Before / after** — both sides of the diff
- **Reason** — optional but encouraged; some changes (production policy edits)
  should require a reason

### Diff presentation

Configuration diffs are not text diffs. They are semantic diffs.

- **Field-level diff** for structured resources (JSON, YAML, form fields). Show
  the field, the old value, the new value. Don't show a line-oriented text diff
  for a 200-line JSON document — operators cannot read it.
- **Summary diff** at the top: "3 rules added, 1 modified, 2 reordered." Let the
  operator drill into the detail.
- **Linked context.** A change to a routing rule should link to the trace
  patterns that may be affected.

### Changelog as a surface

A combined "what changed across the system" view, filterable by resource type,
actor, and time range. This is the surface operators open after an incident.
Make sure it exists, is fast, and is filterable.

### What not to do

- Don't show audit entries as a flat text dump. They are structured records;
  treat them as such.
- Don't allow audit entry deletion or editing. Even for "test" entries. The
  audit trail's value is its inviolability.
- Don't omit the "reason" field. Even if it's optional, prompting for one
  changes operator behavior in useful ways.

---

## Observability vs Configuration

A subtle but important distinction: control planes have two kinds of surfaces,
and confusing them is a common design failure.

- **Configuration surfaces** answer "what should the system do?" The operator
  is making a change. Forms, validators, previews, apply buttons.
- **Observability surfaces** answer "what did the system do?" The operator is
  reading. Tables, charts, traces, audit logs.

Some surfaces blend both — a feature flag UI shows current exposure
(observability) *and* lets the operator change targeting (configuration). When
they blend:

- **Visually separate the two modes.** The "current state" panel must look
  different from the "edit this" panel. Operators must never wonder which one
  they are looking at.
- **Default to observability.** Most visits to a control plane are to *look*,
  not to *change*. The default state of any blended surface is read-only;
  editing is an explicit detour.
- **Make the transition explicit.** A clear "Edit" button that flips the
  surface into configuration mode is better than always-editable forms that
  also happen to display state.

### What not to do

- Don't show editable fields as the default for read-mostly surfaces. The
  operator will accidentally change a value they meant to look at.
- Don't show "view-only" toggles. Either the operator has edit permission or
  they don't; the UI should reflect their actual permission rather than a
  modal state.

---

## Icons, Tooltips, and Text Discipline

Control-plane surfaces are dense. Text discipline matters more here than in
product UIs, because the operator's attention budget is consumed by the data
itself.

### Icons

- **Icons earn their place by repetition.** A status icon that appears 500
  times on a table page saves real cognitive effort. A decorative icon next to
  a button label costs cognitive effort.
- **Pair icons with labels** unless the icon is universally understood
  (search, close, refresh). Even then, a label improves accessibility.
- **Never invent icons for novel concepts.** If the concept doesn't have a
  conventional icon, use a label. An invented icon means the operator has to
  decode it on every encounter.

### Tooltips

- **Tooltips supplement, never replace, labels.** If the only place a control's
  meaning lives is in a tooltip, the control is mis-labeled.
- **Tooltips appear on focus, not just hover.** Keyboard users and touch users
  need them too.
- **Tooltips have a length budget.** ~1 sentence. If you need more, the field
  needs a `?` affordance that opens a popover, not a tooltip.

### Text in general

- **Affirmative, concrete copy.** "Requests over 8k tokens route to the
  long-context model" is better than "Long-context routing is configured."
  Tell the operator what the system does, not what the field does.
- **No marketing voice.** Control planes are not products being sold; they are
  tools being used. "Powerful," "intelligent," "seamless" — strip all of these.
- **Numbers and identifiers are monospace.** Mixed-case proportional fonts
  make long IDs hard to scan and compare. UUID-like strings always render
  in monospace.

### What not to do

- Don't use the same word for two different concepts on the same screen.
  "Enabled" sometimes meaning "the operator turned it on" and sometimes
  meaning "the system has it available" is the kind of subtle equivocation
  that erodes operator trust.
- Don't use ALL CAPS for emphasis. Use weight, color (sparingly), or position.
- Don't truncate identifiers without a copy affordance. If you have to show
  `wh_a8c4e9...`, the full ID must be one click away.
