---
name: peak-close-audit
description: Sensor-verified QA of user-supplied PEAK ticket IDs (action tickets, alert tickets, or a mix) to decide which action tickets can be confidently closed. Only invoke when the user explicitly runs the /peak-close-audit slash command — never auto-trigger on related keywords or close-out questions. Once invoked, stay active for the rest of the session for follow-up drilldowns and batch closures.
---

# PEAK Close Audit

## Job

Given ticket IDs — action tickets, alert tickets, or an unlabelled mix — decide per **action ticket** whether it can be confidently closed, with raw sensor data as the deciding evidence. Present verdicts, then batch-close only on the user's confirmation.

Why this exists: the naive "alert says recovered → close the action" heuristic is wrong roughly 80% of the time. Recovery flags lag, flip-flop, and mask dead sensors. Closure must be earned with sensor evidence.

## Scope policy (non-negotiable)

- The unit of closure is the **action ticket**. Alert IDs resolve up to their parent action(s) via `linked_action_ids`; there is no close-alert operation in this toolset.
- Assess only **active** alerts (`fault` / `recovered`) that are linked to **open** actions (`open` / `in_progress` / `on_hold`).
- Everything else is out of scope and must be surfaced with its reason, never assessed or closed: IDs found in neither service, closed/ignored alerts, alerts with no linked action, actions already terminal, open actions with no active linked alert.
- One verdict per action, no matter how many input IDs point at it — dedupe before assessing, and record which input ID(s) led to each action so the output accounts for every ID supplied.
- An action may only close if **all** of its active linked alerts pass — including alerts the user never mentioned.
- If invoked with no ticket IDs, ask for them once and stop. Never pull candidates from a portfolio on your own.

## Evidence model

Five signals per action, cheapest first; stop at the first hard failure. The execution gate (3) is the decisive check and **must pass before** the expensive sensor pull (5).

1. **Recovery gate (hard).** Every active linked alert must be `recovered`. Any alert still in `fault` → 🚨 KEEP (active fault).
2. **Stability gate (hard).** An alert whose `fault_status` changed more than twice in the last 14 days is flip-flopping → 🚨 KEEP (unstable). A rule that recovered yesterday after weeks of daily flapping is not recovered; one quiet day is not recovery.
3. **Execution gate (hard, decisive).** A `recovered` flag is only real if the rule is still running and evaluating. For every active linked alert's rule (`rule_id` = the task's `task_id`), read `tasks.task_history` over the audit window (`task_ids` + `task_start`):
   - **Zero executions in the window → 🚨 KEEP.** The rule went silent, so `recovered` is stale, not earned. Do **not** trust `rule_state: running` for this — a rule can read "running" yet have stopped executing (e.g. its feeding sensor died); only task_history proves it actually ran.
   - **Any Fault run (`status: true`) in the window → 🚨 KEEP.** The condition recurred since the flag flipped. `status` is the rule's own per-run verdict; a run that also shows `alert_status`/`scheduled_alert: true` escalated to a customer alert (unambiguous KEEP), and a sustained rate of bare `status` faults is likewise decisive — an isolated `status` blip with no alert may be debounce noise, so let the sensor pull settle that one rather than auto-KEEP.
   - **Live and every run OK → recovery confirmed by the rule itself** → proceed to sensor verification.
   Read the shape, not the row count: one execution can emit several rows (one per `point_ts`) and `task_ts` clusters during back-fills, so judge liveness by recent `point_ts`. Frequent `task_status: false` (execution errors) means a struggling rule whose OK runs aren't trustworthy → lean ⚠️/KEEP.
4. **Comments — context, never a gate.** Read the action's comment thread to understand what's pending and why, but never block or grant closure on comments alone. Weight by age: under ~14 days is live context; older is background (a 60-day-old "contractor booked" comment usually means the contractor already came — let the sensors say whether it worked). Comments from automation (e.g. **Agent Hannah**) are system reminders, not a human waiting.
5. **Sensor verification — the arbiter (gated).** Only for actions that cleared the execution gate, pull about 7 days of raw history for the sensors wired to each surviving alert's rule and verify the equipment actually behaves recovered. Derive what "recovered" must look like from the rule's intent (template name/category) — e.g. an overnight-operation rule needs the unit demonstrably off across the site's overnight hours every night; a broken-sensor rule needs readings plausible, in range, and varying. This is also where the dead-sensor masquerade is caught: a rule can run and report OK precisely because its input flatlined, so confirm the feeding sensors are fresh, in range, and varying — passing the execution gate is necessary, not sufficient.

## Verdicts

| Verdict | Meaning |
|---|---|
| ✅ CLOSE | Gates passed and sensor data positively confirms the recovered pattern. |
| 🚨 KEEP | Failed a gate (active fault, unstable, rule gone silent, or re-faulting in-window), sensors show the fault persists, or the "recovery" is a dead sensor (flatline masquerading as quiet). |
| ⚠️ INVESTIGATE | Sensors contradict the recovered classification — likely a rule-logic gap worth surfacing to product. |
| ⏳ HOLD | Sparingly: sensor evidence is genuinely ambiguous AND a fresh (<14 days) human comment shows work in flight. |
| ⚪ OUT OF SCOPE | Excluded by the scope policy, with the reason stated. |

When torn between CLOSE and anything else, don't close.

## Data map

Three GraphQL services sit behind the PEAK MCP tools. Use the dedicated `search_*` tools where they fit; for the rest, discover with `describe_graphql_query` and call `execute_graphql_query`.

- **tickets** — action and alert tickets (`search_action_tickets` / `search_alert_tickets`; links are bidirectional via `linked_alert_ids` / `linked_action_ids`), comments (`search_action_comments`), and per-ticket `change_history` — count entries where `field_id == "fault_status"` for the stability gate.
- **tasks** — rules and their execution log. An alert's `rule_id` **is** the task's `task_id` (same number, different name across services). A task's `sensors[]` carries the rule's wired points and their `fav_id`s. `tasks.task_history` (filter `task_ids` + `task_start`) is the per-run record behind the execution gate — `status` is Fault/OK per run, `point_ts` the data time, `task_ts` the run time. It has **no pagination and a hard response-size cap**, so query a short recent window (≈24–48h) with minimal fields (`task_id, point_ts, status`) per rule or in small `task_ids` batches; only widen the window when a rule looks silent and you need to date its last run.
- **platform** — `history(fav_ids, start, end)` returns raw timeseries at roughly 15-minute samples; `search_sites` gives the site's IANA timezone for schedule bucketing.

Work batched per layer, not per ticket: one typed lookup of the full input list against both ticket services (IDs are unique across them, so each ID lands in exactly one bucket or is "not found"), one alerts fetch, one tasks fetch, then a short-window `task_history` execution-gate pass over the candidate rules — and **only then** one history call for the fav_ids of the rules that survive the gate. Both the task_history and history responses can exceed inline tool-result limits — save and aggregate offline (Python) rather than eyeballing raw points.

## Domain gotchas

- **Status vs setpoint.** "Is it running" comes from status/enable points (`_ENA`, `_St`), not speed/command points — a VSD reading frozen at 54.35 is a setpoint in BMS memory, not operation. Multi-state status points often idle at 1 rather than 0; judge "off" by the point's semantics, not by `== 0`.
- **`rule_state` is not liveness.** A rule can report `rule_state: running` while it has actually stopped executing — only `tasks.task_history` reveals whether it ran. And `status` (the rule's per-run fault verdict) is not `alert_status` (whether a customer alert was raised): a rule can log many `status: true` runs that never escalate.
- **Dead sensor masquerade.** A sensor flatlined for the whole window is a sensor problem, not a recovery.
- **Cumulative meters.** Water and energy meters accumulate — judge hourly deltas, not raw values.
- **Schedule rules need local time.** Bucket overnight/occupancy analysis by the site's timezone (from `search_sites`), not UTC.
- **Siblings on the same rule.** Multiple open actions/alerts can point at the same rule and equipment. Assess them coherently: instability or an active fault on one is evidence about the equipment, and duplicate actions deserve a note, not contradictory verdicts in silence.

## Output and closure

One markdown table of assessed actions — one row per action:

| # | Action link | Site / equipment | Verdict | Evidence one-liner |

Action link is always the parent action: `https://ace.cimenviro.com/tickets/escalated/<site_id>/<ticket_id>`; when reached via a supplied alert ID, say so in the evidence line. The evidence one-liner pairs the sensor finding with any live comment context.

Below the table, list **Out of scope / skipped** input IDs with a one-phrase reason (only if non-empty) so every supplied ID is accounted for.

End with a one-line offer to close the ✅ set and wait. On confirmation, for each ✅ action **post a bespoke closing comment, then close**.

**Closing comment.** One line per action, written from *that action's* evidence — ~6–10 words, neutral, naming the specific fault that cleared and that it held across the window. No raw data values, no jargon beyond the unit name; it's earned by the assessment, not a canned string. Patterns by rule intent:

- Mismatch alarm → "Enable/status mismatch no longer present; stable over the past week."
- Overnight / weekend operation (energy waste) → "Unit confirmed off out-of-hours; no recurring waste."
- Unit not operating → "Unit now running normally on schedule for the past week."
- Broken / flatlined sensor or meter → "Sensor reporting plausible, varying readings again; data restored."
- Critical sensor outside range → "Reading back within expected range and stable."

Post the line with `update_action_tickets(field="comment", value="<line>", ticket_ids=[<that one action>])` — comment text is per-ticket, so post bespoke lines one action at a time (a single call only batches when the text is identical). Then close the **action** IDs (never alert IDs) with `update_action_tickets(field="status", value="closed", ticket_ids=[...])` and verify per ticket. Comment and close only the ✅ set — never 🚨 / ⚠️ / ⏳ / ⚪.
