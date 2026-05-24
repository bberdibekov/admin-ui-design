# Surface Patterns

Detailed layouts for common control-plane surface types. Each entry follows the
same shape:

- **Shape** — the canonical layout in arrow/column notation
- **Operator questions it must answer** — the questions the operator brings to the surface
- **Common gotchas** — failure modes specific to this surface
- **What not to do** — patterns to avoid

Read the entry for the surface you are building. If you are building something
not listed here, find the closest analogue and adapt — the deeper principles in
`SKILL.md` will fill the gaps.

---

## Table of Contents

1. [API Settings](#api-settings)
2. [Config / Runtime Settings](#config--runtime-settings)
3. [Tools / Integrations](#tools--integrations)
4. [Memory](#memory)
5. [Budget / Quota](#budget--quota)
6. [Traces / Logs / Audit](#traces--logs--audit)
7. [Feature Flags / Rollouts](#feature-flags--rollouts)
8. [Webhooks / Outbound Integrations](#webhooks--outbound-integrations)
9. [Secrets / Credentials](#secrets--credentials)
10. [Permissions / Roles / Access](#permissions--roles--access)
11. [Health / Status Dashboards](#health--status-dashboards)

---

## API Settings

**Shape:** `Key/provider state` → `Configuration` → `Validation result`

**Operator questions:**
- Is the key configured?
- Does it actually work right now?
- What scopes/permissions does it have?
- When was it last used, and from where?

**Common gotchas:**
- Validation that only checks syntax. "The key looks well-formed" is not validation. The surface must probe a real endpoint and report the response.
- Showing the key value by default. Mask it; require an explicit reveal action; log the reveal.
- Conflating "key configured" with "key working." A revoked key is configured but not working — the surface must show both states distinctly.

**What not to do:**
- Don't put "Save" and "Test" as separate steps where Save commits without testing. Save should imply test, or the test should run on edit.
- Don't lose the key on validation failure. The operator may want to fix one field and retry, not retype the whole key.

---

## Config / Runtime Settings

**Shape:** `Current runtime inputs` → `Active constraints` → `Reload/apply preview`

**Operator questions:**
- What is the system actually running with right now?
- What have I changed but not yet applied?
- What will reload mean — restart, hot-reload, queued for next request?

**Common gotchas:**
- The three states **edited but not applied**, **applied but not yet reloaded**, and **live** look the same. They are not the same and conflating them causes outages.
- "Apply" buttons that don't tell the operator what reload semantics they invoke. A config change that requires a process restart is different from one that takes effect on the next request.
- Stale local state. If another operator changed the config in another session, the current operator's view is wrong. Show a freshness indicator and offer a reload.

**What not to do:**
- Don't allow editing of fields whose change-semantics aren't visible. If a field requires a restart, say so on the field, not in a footnote.
- Don't auto-apply on field change for any field that affects production behavior. Always require an explicit apply step with preview.

---

## Tools / Integrations

**Shape:** `Available tools` | `Enabled tools` | `Permissions` | `Runtime availability`

These four sections must be **visually separated**. A tool can be:
- Available but not enabled (capability exists, operator hasn't turned it on)
- Enabled but not permitted (turned on, but policy blocks it for this user/scope)
- Permitted but not reachable (allowed in policy, but the underlying service is down)
- Reachable but not enabled (the system could call it, but the operator hasn't opted in)

Each combination has different remediation. Collapsing them into one
"enabled/disabled" toggle hides the diagnostic information operators need.

**Operator questions:**
- Why didn't this tool fire on the last call?
- Who is allowed to use this tool?
- Is the underlying service healthy?
- What scopes does this tool have on my data?

**Common gotchas:**
- Showing one big "enabled" switch and burying the four-way distinction behind it. The operator sees "enabled" but the tool still doesn't fire — and now they don't know why.
- Mixing first-party tools with third-party integrations in the same list with no visual distinction. Their failure modes and trust assumptions differ.
- Not surfacing the most recent invocation. The operator's first question is usually "did it run? when? what happened?"

**What not to do:**
- Don't make the enable toggle simultaneously the permission control. They are different concepts; an admin enabling a tool for the org is not the same as a user being permitted to call it.
- Don't hide reachability under a generic "error" pill. "Unreachable" is a specific diagnosable state distinct from "misconfigured."

---

## Memory

**Shape:** `Stored facts` → `Retrieval policy` → `Previewed recall` → `Actual injected context`

**Operator questions:**
- What does the system "know" about this user/entity?
- For a given query, what *would* be recalled?
- For a recent call, what *was* recalled and what made it through to the model?
- Why did this fact get retrieved and that one didn't?

**Common gotchas:**
- The single largest debugging gap in LLM systems is between "what is stored" and "what is injected for this specific query." Make this gap inspectable, not inferred.
- Confusing storage with retrieval. A fact that is stored but never retrieved is invisible to the model; the UI must make that distinction obvious.
- No way to test a hypothetical query against the current retrieval policy. The operator should be able to type a sample query and see what *would* come back.

**What not to do:**
- Don't show "memory" as a flat list. Operators need to filter by source (user-asserted vs. system-derived), recency, confidence, and topic.
- Don't allow deletion of facts without showing which past responses depended on them. Memory deletion is destructive; treat it as such.

---

## Budget / Quota

**Shape:** `Limits` | `Current spend` | `Enforcement policy` | `Projected action`

**Operator questions:**
- Am I going to hit a cap?
- When?
- What happens when I do?
- Who or what is driving the spend?

**Common gotchas:**
- Showing a current-spend number without context. "$847 spent" without a limit, a rate, or a projection is a number, not a tool.
- The **projection** ("at current rate, the soft cap fires in 6 days") matters more than the snapshot. Prioritize it.
- Confusing soft caps (alert + continue) with hard caps (alert + block) in the UI. The operator must know whether hitting the cap will degrade service or just notify someone.

**What not to do:**
- Don't bury per-resource breakdown behind a click. The first question after "am I over budget" is "what is causing it" — show the top contributors immediately.
- Don't show enforcement policy as a separate disconnected page. Operators need to see the limit, the policy, and the projection together to reason about risk.

---

## Traces / Logs / Audit

**Shape:** `Raw events` → `Interpreted decision path` → `User-actionable diagnosis`

Three layers, in this order. Most trace viewers stop at layer one and call it
done; this is why operators avoid them.

- **Layer 1 (raw events):** What the system emitted. Timestamps, IDs, payloads.
- **Layer 2 (decision path):** Why the system did what it did. Which rule fired, which model was selected, which tool was called, which fallback engaged.
- **Layer 3 (diagnosis):** What the operator should change. Specific, linked to the relevant config surface.

**Example for an LLM system:**
- Layer 1: raw token stream, tool invocation logs, latency numbers
- Layer 2: "Request matched policy `long-context-override`, was routed to model X, tool call to `search` failed with timeout, fell back to default behavior"
- Layer 3: "Request fell back to default model because the regex matcher timed out. Consider increasing the matcher budget in Settings → Routing → Matcher timeout."

**Operator questions:**
- What happened to this specific request?
- Why did the system make that choice?
- Is this pattern recurring?
- What do I change to prevent the next occurrence?

**Common gotchas:**
- Trace viewers that are just JSON pretty-printers. The operator can read JSON in any editor; the value is interpretation.
- No way to filter by *decision*, only by event type. "Show me all requests that fell back to the default model" is a more useful query than "show me all events of type X."
- No way to link a trace back to the config that produced the decision. The trace should deep-link to the relevant routing rule, policy, or override.

**What not to do:**
- Don't paginate without preserving filter state. Trace investigation is iterative; losing filters on page change destroys the workflow.
- Don't show only the request that the operator selected. Show neighboring requests too — patterns emerge in context.

---

## Feature Flags / Rollouts

**Shape:** `Flag definition` → `Targeting rules` → `Current exposure` → `Recent flips`

**Operator questions:**
- Who currently sees the new behavior?
- When was this flag last flipped, and by whom?
- Is this flag still in use, or is it stale?
- What happens if I turn it off right now?

**Common gotchas:**
- No flip history. A flag UI without a recent-changes log is a footgun — operators cannot reason about incidents tied to flag changes.
- No staleness indicator. Flags that haven't been touched in 6+ months and are at 100% should be flagged as candidates for retirement.
- Targeting rules expressed in opaque DSL with no preview. The operator should be able to enter a sample user/context and see whether the flag would be on for them.

**What not to do:**
- Don't allow flag flips without naming the actor and storing a reason. Post-incident, the question "who flipped this and why" must be answerable.
- Don't show "% rollout" as the only exposure metric. Show absolute counts too — 1% of 10M is very different from 1% of 1k.

---

## Webhooks / Outbound Integrations

**Shape:** `Endpoint` → `Subscribed events` → `Recent deliveries (with status + payload)` → `Retry policy`

Recent deliveries are the most-used part of this surface. Make them the most
prominent, not buried under endpoint configuration.

**Operator questions:**
- Are deliveries succeeding?
- What did the last failure look like — payload, response, status?
- What is the retry behavior — will it eventually succeed, or is it permanently failed?
- Can I replay a specific delivery?

**Common gotchas:**
- Showing only the most recent N deliveries with no filter for failures. The operator usually wants "show me failures in the last hour," not "show me everything."
- No replay action. When debugging an integration, the ability to replay a specific delivery against the current endpoint is invaluable.
- Payloads that are truncated without a way to expand. Operators need the full payload to diagnose.

**What not to do:**
- Don't conflate webhook delivery failures with subscriber-side errors. A 500 from the subscriber is different from a network timeout, and both differ from a malformed event the webhook system rejected before sending.

---

## Secrets / Credentials

**Shape:** `What it is` → `Where it is used` → `When it was last rotated` → `Rotation action`

**Operator questions:**
- What is this secret used for?
- What depends on it — which services, jobs, integrations?
- When was it last rotated?
- Is rotation safe right now, or will it break something?

**Common gotchas:**
- Displaying secret values by default. Mask them; require explicit reveal; **log every reveal**.
- No "where used" panel. The single most common secrets incident is rotating a key without realizing what depends on it. The surface must prevent this.
- No expiry surfacing. If the secret has a known expiry, show it as a count-down, not just a date.

**What not to do:**
- Don't allow deletion without a "where used" check. If a secret has dependents, the delete dialog must list them.
- Don't put rotation behind the same affordance as edit. They are different operations with different consequences.

---

## Permissions / Roles / Access

**Shape:** `Subject` → `Resource` → `Effective permission` → `Source of that permission`

The **source** column is non-negotiable. When permissions are inherited from
groups, roles, or policies, the operator must be able to answer "why does this
user have access?" without leaving the screen.

**Operator questions:**
- Can this user do this action on this resource?
- *Why* — what role, group, or direct grant gives them that?
- What would happen if I removed them from group X?
- Who else has this same permission, and through which path?

**Common gotchas:**
- Showing effective permissions without their source. The operator sees "user has write access" but can't tell whether it came from a group, a role, a direct grant, or a policy override.
- No "simulate removal" mode. Removing a user from a group should be previewable: which permissions will they lose? Which will they retain through other paths?
- Conflating identity (who you are) with role (what you're allowed to do) with session (currently authenticated). Three different concepts, often muddled in the UI.

**What not to do:**
- Don't show roles as flat lists divorced from the permissions they grant. The operator needs to see what a role *does*, not just what it's called.
- Don't allow permission grants without an audit trail. Every grant should record actor, timestamp, and (ideally) reason.

---

## Health / Status Dashboards

**Shape:** `What is being measured` → `Current value` → `Threshold` → `Trend`

A red light without a threshold is theater. The operator needs to know not just
that something is wrong, but how wrong, and whether it is getting worse.

**Operator questions:**
- Is the system healthy right now?
- If something is degraded, how bad is it relative to the threshold?
- Is it improving or worsening?
- What is the historical baseline — is this unusual?

**Common gotchas:**
- Status pills without numbers. "Degraded" tells the operator nothing actionable; "p99 latency is 4.2s, threshold is 2s, trending up over 30 min" is diagnostic.
- No threshold visibility. The operator can't tell whether the system is at 99% of capacity or 30%.
- Aggregate health with no drill-down. "Overall: green" is fine as a summary, but the operator must be able to click in to see the per-component view that produced the summary.

**What not to do:**
- Don't show only the current snapshot. Always show a short trend window (last 5, 30, 60 minutes) so the operator can see direction.
- Don't use color as the only signal. Operators with color-vision deficiencies and operators glancing at the screen in bright light both need the value and threshold to be readable on their own.
