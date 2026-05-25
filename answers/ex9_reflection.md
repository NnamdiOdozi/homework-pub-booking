# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In session `sess_ad5adfe19a16` (Ex7 real run), round 1 planner ticket
`tk_e05167c0` produced two subgoals. sg_2 has `"assigned_half":
"structured"` and `"description": "Hand off to the booking system with
the selected venue"`. The signal: the task text explicitly names a
booking confirmation step, and the DefaultPlanner system prompt
instructs it to use `structured` for subgoals requiring strict
rule-following. The planner read "hand off to the booking system" as
a deterministic policy step and assigned it accordingly.

This assignment is advisory, not enforced by hardware. The bridge
respects it only because both halves are wired and the executor calls
`handoff_to_structured` when sg_2 runs. If `complete_task` remained
in the registry, the LLM could bypass the handoff entirely — which
was the original failure mode before it was removed from the loop
registry.

The broader lesson: the planner's `assigned_half` field is prose
interpretation, not a physical gate. The only reliable enforcement is
code: remove affordances that let the LLM skip the structured half.

### Citation

- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/tickets/tk_e05167c0/raw_output.json`
- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/trace.jsonl`

---

## Q2 — Dataflow integrity catch

### Your answer

Session `sess_7a751e32fa97` (Ex5). `verify_dataflow` would catch a
fabrication that manual review would miss. The trace shows
`generate_flyer` was called with placeholder `event_details`:
`venue="Edinburgh Castle"`, `name="Sample Event"` — neither value
appears in any prior tool output. The real venue returned by
`venue_search` was `haymarket_tap`; "Edinburgh Castle" was never in
any fixture.

A human reviewer scanning `workspace/flyer.html` sees a formatted
flyer with a plausible Edinburgh venue name and reasonable cost
figures. Nothing looks obviously wrong. `verify_dataflow` compares
flyer content against `_TOOL_CALL_LOG` entries: "Edinburgh Castle"
matches zero tool outputs, so it returns `unverified_facts` containing
the venue name and flags the discrepancy.

The check catches this because it compares against ground truth in the
log, not against "does this look plausible." The same mechanism would
catch hallucinated cost figures: if `calculate_cost` returned £2540
but the flyer said £1800, the regex extractor finds £1800 in the flyer
text and cannot find it in any tool output → flagged. Manual review
sees a believable number and passes it.

### Citation

- `sessions/examples/ex5-edinburgh-research/sess_7a751e32fa97/logs/trace.jsonl` lines 6–7
- `sessions/examples/ex5-edinburgh-research/sess_7a751e32fa97/workspace/flyer.html`
- `starter/edinburgh_research/integrity.py` — verify_dataflow

---

## Q3 — First production failure and the primitive that surfaces it

### Your answer

The first production failure I'd expect: the Rasa webhook goes down
mid-session between the loop half completing and the structured half
acknowledging. The booking data is in the IPC file; the bridge has
handed off; but the HTTP POST to Rasa times out or returns 503. With
no retry and no durable record of the attempt, the session is stuck —
`structured` never transitions to `completed` or `escalated`, the
bridge hangs until `max_rounds`, and the booking is silently lost.

The primitive that surfaces it is the **ticket state machine**. Every
`planner.plan` and executor subgoal run produces a ticket with
`state: error | completed`. If the Rasa call fails, the structured
half's ticket transitions to `error` with `error_code:
SA_EXT_SERVICE_UNAVAILABLE`. Without the ticket, the failure is a
silent timeout — nothing in the trace distinguishes "structured half
rejected" from "structured half never got the request." With the
ticket, `state.json` in `logs/tickets/tk_*/` records the exact
failure code, timestamp, and duration, making the failure observable
and retryable.

In session `sess_ad5adfe19a16` the structured half's ticket recorded
state transitions cleanly; any 503 from Rasa would appear there first
before surfacing in the bridge outcome.

### Citation

- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/tickets/`
- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/trace.jsonl`
