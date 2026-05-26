# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In session `sess_ad5adfe19a16` (Ex7 working run), round 1 planner ticket
`tk_e05167c0` produced two subgoals. Subgoal sg_2 has `"assigned_half":
"structured"` and `"description": "Hand off to the booking system with
the selected venue"`. The signal the planner acted on was the task text
itself, which explicitly included a step named "hand off to the booking
system." The DefaultPlanner system prompt instructs it to assign `structured`
for subgoals requiring deterministic rule-following rather than open-ended
search, and the phrase "hand off to the booking system" was read as a
deterministic policy step: a binary committed/rejected decision with no
room for LLM improvisation.

This assignment is advisory, not enforced by the framework. We discovered
this the hard way in our earlier session `sess_9821b631a49c`. There,
`complete_task` remained registered in the loop half's tool registry.
The LLM used it at 12:25:52 to exit cleanly after finding Haymarket Tap
— before ever calling `handoff_to_structured`. The trace shows
`complete_task` called with a venue result dict, `session marked complete`,
and no structured half involvement at all. The booking was never confirmed.

The fix was `tools.unregister("complete_task")` immediately after
`build_tool_registry(session)` in `run.py`. With the tool absent, the
only valid exit from the loop half is `handoff_to_structured`. The planner's
`assigned_half: "structured"` field then becomes meaningful — not because
the framework enforces it, but because the LLM has no other way out. The
lesson: `assigned_half` is a prose interpretation, not a gate. Instructions
can be ignored; missing tools cannot.

### Citations

- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/tickets/tk_e05167c0/raw_output.json`
- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/trace.jsonl`
- `sessions/examples/ex7-handoff-bridge/sess_9821b631a49c/logs/trace.jsonl` — line 6: `complete_task` called before handoff

---

## Q2 — Dataflow integrity catch

### Your answer

Session `sess_7a751e32fa97` (Ex5). The flyer was generated with fabricated
`event_details`: `venue_name="Edinburgh Castle"`, `name="Sample Event"`.
Neither value appeared in any tool output. The real venue returned by
`venue_search` was `haymarket_tap`; "Edinburgh Castle" is not a fixture
venue and never appeared in the log. The fabricated cost figures in the
flyer also differed from what `calculate_cost` actually returned.

A human reviewer scanning `workspace/flyer.html` sees a formatted page
with a plausible Edinburgh venue name and reasonable cost figures. The
layout is correct, the HTML is well-formed, the numbers look credible.
Nothing flags as wrong. `verify_dataflow` caught it because it does not
ask "does this look plausible" — it extracts money facts (£N), temperature
facts (N°C), and condition keywords from the flyer, then calls
`fact_appears_in_log` for each one. If a value in the flyer cannot be
matched against any entry in `_TOOL_CALL_LOG`, it is returned in
`unverified_facts`. The fabricated cost figure was one such value: it
appeared in the flyer but not in any tool output record.

In our second run `sess_0111f73b3160`, the LLM again passed fabricated
`event_details` (`{"venue": "The Royal Terrace", "date": "2023-08-15",
"weather": "Sunny, 22°C"}`) to `generate_flyer`. This time the
`_build_event_details_from_log` interceptor overwrote these with real
values from `_TOOL_CALL_LOG` before the tool ran. The resulting flyer
showed The Royal Oak, cloudy 12°C, £1103 deposit £330. `verify_dataflow`
verified 4 facts. The contrast between the two sessions demonstrates the
check's value: in both cases the LLM produced a plausible-looking call;
only the integrity layer distinguished one from the other.

### Citations

- `sessions/examples/ex5-edinburgh-research/sess_7a751e32fa97/logs/trace.jsonl`
- `sessions/examples/ex5-edinburgh-research/sess_7a751e32fa97/workspace/flyer.html`
- `sessions/examples/ex5-edinburgh-research/sess_0111f73b3160/logs/trace.jsonl`
- `sessions/examples/ex5-edinburgh-research/sess_0111f73b3160/workspace/flyer.html`
- `starter/edinburgh_research/integrity.py` — verify_dataflow, fact_appears_in_log

---

## Q3 — First production failure and the primitive that surfaces it

### Your answer

The first production failure we observed directly: the loop half
consistently omitting `venue_id` from the `handoff_to_structured` payload,
causing the structured half to reject every round with the same error. In
session `sess_9821b631a49c`, this happened three times in succession. The
trace records `session.state_changed: structured → loop, rejection_reason:
"normalisation failed: missing venue_id"` at 12:28:07, 12:29:02, and
12:29:40 — three consecutive rounds, identical failure, no progress. The
bridge exhausted its round limit and the booking was silently lost.

In a production pub-booking system this is the most damaging failure mode:
the customer sees nothing (the agent appeared to be working), the pub sees
nothing, and the booking evaporates. No exception is raised, no alert fires
unless someone inspects the logs.

The primitive that surfaces it is the **ticket state machine**. Session
`sess_9821b631a49c` has 8 tickets (`tk_18a89641`, `tk_d9720b42`,
`tk_76bad39d`, `tk_e2e5a228`, `tk_ba40950b`, `tk_cc40a000`, `tk_c2e5c7c7`,
`tk_4204f70d`) across 3 rounds. Each planner and executor subgoal creates
a ticket with `started_at` and `completed_at` in `logs/tickets/tk_*/state.json`.
The bridge's trace events carry a `rejection_reason` field on every
`structured → loop` transition. Together these make the failure pattern
observable: an alert rule that fires when the same `rejection_reason` appears
on three consecutive `session.state_changed` events would have caught this
in the first minute. Without the structured ticket and trace records, an
operator sees "bridge completed in 3 rounds" with no indication that rounds
1, 2, and 3 all failed for the same preventable reason.

The fix in our working session `sess_ad5adfe19a16` was `build_forward_handoff`
in `bridge.py`, which scans `_TOOL_CALL_LOG` for the most recent successful
`venue_search` result and patches the missing `venue_id` into the payload
before forwarding. The ticket record for that session shows the structured half
completing cleanly in round 2, booking reference `BK-4B5F7858`.

### Citations

- `sessions/examples/ex7-handoff-bridge/sess_9821b631a49c/logs/trace.jsonl` — lines 14, 23, 31
- `sessions/examples/ex7-handoff-bridge/sess_9821b631a49c/logs/tickets/` — 8 tickets across 3 rounds
- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/trace.jsonl` — line 15: BK-4B5F7858
- `starter/handoff_bridge/bridge.py` — build_forward_handoff
