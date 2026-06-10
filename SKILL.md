---
name: peak-close-audit
description: Sensor-verified QA of user-supplied PEAK tickets — action tickets, alert tickets, or a mix — to decide which actions can be confidently closed. Auto-detects each ID's type, resolves alerts to their parent action, and assesses only active alerts (fault/recovered) linked to open actions. Uses four evidence sources (alert recovery → change history → comment context → raw favourite timeseries) with raw sensor data as the primary arbiter and comments as interpretive context. Only invoke when the user explicitly runs the /peak-qa-actions slash command. Do not auto-trigger on related keywords, topics, or close-out questions. Once invoked, remain active for the rest of the session to handle follow-up drilldowns and batch closures without re-invocation.
---

# PEAK QA Actions Skill

## Purpose

Given a list of PEAK ticket IDs — **action tickets, alert tickets, or a mix** — decide which **actions** can be **confidently closed** based on layered evidence. Raw sensor data is the primary arbiter. Comments provide interpretive context but do not gate the workflow.

The unit of closure is always the **action ticket**. Alert IDs the user supplies are resolved to their parent action and assessed there, because an action can only close once all of its active linked alerts are genuinely recovered. The platform has no "close alert" operation in this skill's toolset — closing the action is the outcome.

**Scoping rule (non-negotiable):** only assess **active alert tickets** (`status` = `fault` or `recovered`) that are **linked to open actions** (`status` = `open`, `in_progress`, or `on_hold`). Closed/ignored alerts, alerts not linked to any action, and actions already in a terminal state (`closed`, `not_doing`) are out of scope — surface them, then drop them.

The naive "alert is recovered → close action" heuristic produces roughly an 80% false-close rate in practice. This skill prevents that by checking four independent signals before recommending closure.

## Trigger

ONLY invoke when the user explicitly types `/peak-qa-actions`. Do not auto-trigger on phrases like "audit actions", "find closeable tickets", or any close-out language. The slash command is the only valid invocation.

Once invoked, stay active for the rest of the session. The user can ask follow-up drilldowns and request batch closures without re-invoking.

## Required input

A list of ticket IDs. They may be **action ticket IDs, alert ticket IDs, or any mix** — the user does not need to label which is which. The skill auto-detects each ID's type (Step 1) and resolves everything to a deduplicated set of open action tickets to assess. That list is the only required input.

If the user invokes `/peak-qa-actions` without supplying ticket IDs, ask for them in a single question and stop. Do not infer scope or pull candidates from a portfolio.

## The four-layer evidence model

| Layer | Signal | Role |
|---|---|---|
| 1 | Alert recovery status | Hard gate. Any active linked alert in `fault` keeps the action open. |
| 2 | Alert change history flip-flop | Hard gate. More than 2 fault↔recovered transitions in last 14 days keeps the action open. |
| 3 | Action comment thread | **Context only.** Informs the verdict but never disqualifies on its own. |
| 4 | Raw favourite timeseries (7-day) | **Primary arbiter.** Sensor reality decides the verdict for tickets that clear layers 1 and 2. |

Layers 1 and 2 are hard gates because they are deterministic, recent, and machine-verifiable.

Layer 3 is context. Comments enrich the reasoning behind a verdict but do not gate the workflow. A comment from 3 months ago saying "contractor coming next Tuesday" is almost certainly stale — the contractor came and went, and the question is whether the equipment is actually behaving now. That is a Layer 4 question.

Layer 4 is the arbiter. Equipment either is or is not behaving correctly right now. Sensor data is the ground truth.

## Step-by-step workflow

### Step 1: Resolve the input to a scoped set of open actions

The input may contain action IDs, alert IDs, or both, and the user has not labelled them. Auto-detect by probing **both** services in parallel with the full input list:

```
search_action_tickets(ticket_ids=<full input list>, limit=200, include_archived=true)
search_alert_tickets(ticket_ids=<full input list>, limit=200, include_archived=true)
```

Each call returns only the IDs that match its type. Ticket IDs are unique across both services, so every input ID resolves to exactly one of these buckets:

- **Matched as action** → candidate action.
- **Matched as alert** → input alert (must be resolved to its parent action).
- **Matched by neither** → "not found". Surface it and drop it.

**Resolve input alerts → parent actions, applying the scoping rule.** *Only assess active alert tickets that are linked to open actions.*
- Keep an input alert only if its `status` is **active** (`fault` or `recovered`). Drop `closed` / `ignored` — record as "skipped — inactive alert".
- Take each surviving alert's `linked_action_ids`. An alert with no linked action is out of scope — record as "skipped — alert not linked to an action" and drop.
- Fetch the parent actions: `search_action_tickets(ticket_ids=<collected linked_action_ids>, limit=200, include_archived=true)`.

**Build the canonical action set.** Union the candidate actions with the actions resolved from input alerts, then **dedupe by `ticket_id`** — a user may paste both an action and one of its own alerts, and it must only be assessed once.

**Keep only open actions.** An action is in scope only if its `status` is `open`, `in_progress`, or `on_hold`. Drop `closed` / `not_doing` — record as "skipped — action already <status>" (you cannot close what is already terminal). This enforces the "linked to open actions" half of the scoping rule.

For each surviving open action, collect its `linked_alert_ids`; note any action with multiple linked alerts (ALL of its active alerts must be recovered to close). Record, per action, **which input ID(s) pointed at it** — the action itself and/or specific alert IDs — so the output can trace each verdict back to what the user supplied.

If the canonical set is empty after scoping, report why (not found / inactive alert / unlinked alert / action already terminal) and stop.

### Step 2: Layer 1 — alert recovery gate

Fetch the status of every linked alert across the canonical action set in one call (reuse the alert rows already pulled in Step 1 where you have them):

```
search_alert_tickets(ticket_ids=<all linked_alert_ids across the canonical set>, limit=200)
```

For each action, consider **only its active linked alerts** (`status` = `fault` or `recovered`). Ignore `closed` / `ignored` linked alerts entirely — they are out of scope per the scoping rule.

- If an action has **no active linked alerts**, there is nothing in scope to verify. Mark it ⚪ OUT OF SCOPE (no active alert) and skip the remaining layers.
- Mark an action "recovery-clean" only if **every** active linked alert has `status = "recovered"`. Any action with one or more active linked alerts still in `fault` skips the remaining layers and is kept open with verdict 🚨 KEEP (active fault).

### Step 3: Layer 2 — change history flip-flop gate

Run only on the **active linked alerts** of actions that cleared Layer 1. Pass those alert IDs as `$ids` to `execute_graphql_query` on the `tickets` service:

```graphql
query History($ids: [String!]!, $start: DateTime!) {
  tickets(ticket_ids: $ids, limit: 200) {
    ticket_id
    fault_status
    last_fault_transitioned_at
    change_history(limit: 200, start_ts: $start) {
      ts
      fields { field_id old_value new_value }
    }
  }
}
```

Count `fields[].field_id == "fault_status"` transitions per ticket over last 14 days. More than 2 = flip-flopping → 🚨 KEEP (unstable), skip remaining layers.

### Step 4: Layer 3 — comment context (do not disqualify here)

For each remaining candidate:

```
search_action_comments(ticket_id=<id>, limit=50)
```

Parse comments to build context, NOT to gate closure. Do not drop tickets from the candidate pool based on comments alone — always proceed to Layer 4.

**Filter out automated reminders.** Comments authored by **Agent Hannah** (or any author with `user_type` indicating automation) are AI-generated reminders, not human input. Surface them as background but never treat them as "a human is waiting on this".

**Apply staleness weighting.** Categorise human comments by age:
- **Fresh (< 14 days):** Strong context. A recent "contractor booked for Tuesday" or "tag @Matt to confirm" genuinely indicates work in flight.
- **Stale (14–90 days):** Background context. Likely the action either happened or stalled. Use as a tiebreaker only when Layer 4 is ambiguous.
- **Old (> 90 days):** Largely irrelevant. The work was done, abandoned, or forgotten. Layer 4 decides.

**What to extract from comments (for context):**
- Pending external dependencies (contractor visit, tenant notification, vendor quote)
- Unresolved questions tagged to a specific person
- Reported sensor issues ("readings look wrong", "sensor jumping")
- Reported equipment-specific context that explains the sensor pattern

This context will shape the **evidence one-liner** in the output table, not the close/keep decision.

### Step 5: Layer 4 — raw favourite timeseries (the arbiter)

Always run Layer 4 on every ticket that cleared Layers 1 and 2, regardless of what comments said.

Map rule → task sensors → fav_ids. The `rule_id` is on each active linked alert (from Step 2). For multi-alert actions, run Layer 4 across the rules of all active linked alerts. Use `execute_graphql_query` on `tasks` service:

```graphql
query GetTaskSensors($ids: [Int!]!) {
  tasks(task_ids: $ids, limit: 50) {
    task_id
    task_name
    template { template_name }
    sensors { fav_id equipment_id metadata_id name }
  }
}
```

The `task_id` here equals the `rule_id` from the tickets service. Same number, different name.

Pull 7-day history on `platform` service. Batch all fav_ids in one query:

```graphql
query Week($ids: [Long!]!, $start: DateTime!, $end: DateTime!) {
  history(fav_ids: $ids, start: $start, end: $end) {
    fav_id ts data
  }
}
```

Sampling is roughly 15 minutes per fav.

### Step 6: Analyse timeseries offline

The history response will almost always exceed inline tool result cap. Save the JSON and aggregate with Python via `bash_tool`:

```python
import json, statistics
from collections import defaultdict

with open('<tool_result_path>') as f:
    raw = json.load(f)
inner = json.loads(raw[0]['text'])
points = inner['results'][0]['data']['history']

by_fav = defaultdict(list)
for p in points:
    by_fav[p['fav_id']].append((p['ts'], p['data']))

for fav, recs in by_fav.items():
    vals = [r[1] for r in recs]
    nonzero = sum(1 for v in vals if v > 0.001)
    distinct = len(set(round(v, 2) for v in vals))
    # min/max/mean, non-zero %, distinct values, day-1 vs day-last mean
```

For schedule-related rules (overnight operation, economy mode), bucket by site local hour. Australian sites use AEST (UTC+10) in winter; Sydney overnight 22:00–06:00 = UTC 12:00–20:00.

For cumulative meters (water, energy), compute hourly deltas, not raw values.

## Verdict assignment

Sensor data (Layer 4) is the primary driver. Comments (Layer 3) inform the wording and edge cases.

| Verdict | Criteria |
|---|---|
| ✅ CLOSE | Layer 4 sensor data positively confirms the recovery pattern. Comments are stale or supportive. For overnight rules: sensor = 0 across all overnight hours for the full 7 days. For economy override: override flag = 0 for >95% of readings. For unit-fail rules: status sensor varies in expected schedule. |
| 🚨 KEEP (active fault) | Failed Layer 1 — at least one active linked alert still in fault. |
| 🚨 KEEP (unstable) | Failed Layer 2 — flip-flopping. |
| 🚨 KEEP (sensor problem) | Cleared L1+L2 but Layer 4 shows sensor flatlined at 0 for the full 7 days (dead sensor masquerading as recovery). |
| ⚠️ INVESTIGATE | Layer 4 contradicts the "recovered" classification. E.g. fan running 24/7 but overnight-energy-waste rule recovered, or unit reading 0 for 7 days but "not operating" rule recovered. Exposes rule logic gaps worth surfacing to product. |
| ⏳ HOLD | Apply sparingly. Use only when: sensor data is genuinely ambiguous AND there is a fresh (< 14 days) human comment from a real person (not Agent Hannah) about pending work. Do NOT apply based on stale comments alone — let the sensor decide. |
| ⚪ OUT OF SCOPE | Resolved by the scoping rule in Step 1/2, not by sensor evidence: not found, inactive alert, alert not linked to an action, action already terminal, or open action with no active linked alert. Never closed. |

If sensor data shows clean recovery and a comment from 60 days ago says "service booked", verdict is ✅ CLOSE — the service almost certainly happened.

If sensor data is ambiguous and a comment from 3 days ago from a human says "contractor coming Friday", verdict is ⏳ HOLD.

## Output format

A single markdown table of the **assessed open actions** — one row per action (not per input ID; an action reached via several alerts still gets one row). Columns:

| # | Action link | Site / equipment | Verdict | Evidence one-liner |

The **Action link** is always the parent action, even when the user supplied an alert ID. When an action was reached via one or more alert IDs the user pasted, note that provenance in the evidence one-liner, e.g. "(via alert 12345)".

The evidence one-liner should combine the Layer 4 sensor finding with any relevant Layer 3 context. Example: "SAF status = 0 for all 224 overnight readings over 7 days; sibling comment thread shows BMS schedule fix applied May 8."

Use the verbatim action-link pattern: `https://ace.cimenviro.com/tickets/escalated/<site_id>/<ticket_id>`.

Below the table, add a short **Out of scope / skipped** list (only if non-empty) so the user can see every ID they passed is accounted for — each with the input ID and a one-phrase reason: not found, inactive alert, alert not linked to an action, action already `<status>`, or no active linked alert.

End with a one-sentence offer: "Close the ✅ ones?" — wait for confirmation, never close unilaterally.

## Closure action

On confirmation, batch-close in a single call. Always close the **action** ticket IDs (the resolved parents), never alert IDs — even when the user originally supplied alert IDs:

```
update_action_tickets(field="status", value="closed", ticket_ids=[<all ✅ CLOSE action ids>])
```

Verify success per ticket. Do not close 🚨 / ⚠️ / ⏳ / ⚪ verdicts.

## Pitfalls and heuristics

**Mixed / unlabelled input.** Never assume the supplied IDs are all actions. Probe both `search_action_tickets` and `search_alert_tickets` with the full list and partition by which service returns each ID. IDs are unique across both services, so each lands in exactly one bucket; anything in neither is "not found".

**Always close the action, not the alert.** When the user pastes an alert ID, it is a pointer to a parent action. Resolve it via `linked_action_ids`, assess that action, and — if it clears — close the action. There is no alert-close operation here.

**Active alerts only.** Closed/ignored linked alerts are out of scope: never let them keep an action open, and never count them in Layers 1–4. An open action whose linked alerts are all closed/ignored has nothing active to verify → ⚪ OUT OF SCOPE, not ✅ CLOSE.

**Open actions only.** You can only close what is still open. An action already `closed` or `not_doing` — including one reached via an alert — is skipped, not re-closed. Surface it so the user knows it was seen.

**Dedupe across input types.** A user often pastes an action and one of its own alerts together, or two alerts of the same action. Resolve everything to parent actions and dedupe by action `ticket_id` before assessing — one action, one row, one verdict.

**Tool-result truncation.** Any history query covering more than 5 favs over 7 days will overflow inline display. Plan for offline Python aggregation from the first call.

**Status sensor vs setpoint.** A VSD speed reading of "54.35" for 720 consecutive readings is a setpoint stuck in BMS memory, not actual operation. For "is the unit running" verdicts, prefer the `_ENA` or `_St` (status) sensor over `VSD`/`Spd`/`Cmd`. Cross-check that the ENA signal varies day-to-day.

**Cumulative vs instantaneous.** Water and energy meters are cumulative. A flat reading of "62.2" for 4 hours doesn't mean the meter is dead; it means no consumption. Compute deltas before judging.

**One-day "recovery" is not recovery.** A rule that flipped to recovered in the last 24 hours but flip-flopped daily for the prior 4 weeks should stay open. Use a 14-day window minimum for change_history.

**Multi-alert actions.** If an action ticket has multiple active linked alerts and even one is still in fault, the whole action stays open regardless of how clean the others look — including alerts the user did not paste. Resolving from a single alert still requires assessing every active alert on its parent action.

**Comments are context, not gates.** This is the most common reasoning error. A pending-contractor comment from 60 days ago does not block closure if the sensor is now showing clean recovery — the contractor came. Let Layer 4 decide.

**Agent Hannah is automation.** Treat Agent Hannah comments as background reminders, not as a human waiting on something. They reflect what the system already knows.

**Don't trust a single quiet day.** Always use a 7-day minimum window for Layer 4.

## Schema map (quick reference)

```
ACTION (tickets) ◀───────────────┐
  └─ linked_alert_ids ──→ ALERT (tickets)
       (ALERT.linked_action_ids ──┘  ← reverse link: alert → parent action)
                              └─ rule_id ──→ TASK (tasks)
                                                 └─ sensors[]
                                                      └─ fav_id ──→ HISTORY (platform)
```

The link is bidirectional: actions carry `linked_alert_ids`, alerts carry `linked_action_ids`. Input that arrives as alert IDs is walked **up** the reverse link to the parent action; everything else is walked **down** from the action as before.

Three GraphQL services:
- `tickets`: action tickets, alert tickets, change_history, comments
- `tasks`: rules (called "tasks" internally), sensor wiring, templates
- `platform`: favourites, history timeseries, equipment, sites