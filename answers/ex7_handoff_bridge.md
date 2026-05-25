# Ex7 — Handoff bridge

## Your answer

The handoff bridge was run and verified in session `sess_ad5adfe19a16`. The
task was to book an Edinburgh pub for 12 people on Friday 25 April 2026 at
19:30. The bridge completed in two rounds, ending with booking reference
`BK-4B5F7858` confirmed by Rasa at `2026-05-25T13:16:46`.

### Round 1 — venue search fails, structured half rejects

The planner (ticket `tk_e05167c0`) decomposed the task into two subgoals.
Subgoal sg_1 — `"Find a suitable venue in Haymarket for 12 people"` — was
assigned to the loop half, with a success criterion of at least one result
from `venue_search`. Subgoal sg_2 — `"Hand off to the booking system with
the selected venue"` — was assigned to the structured half and declared a
dependency on sg_1.

The executor ran sg_1 by calling `venue_search(near='Haymarket',
party_size=12)`, which returned zero results. It tried a second time with
`budget_max_gbp=1500` added, and again received zero results. With nothing
to show, the bridge still forwarded the session state to the structured half
— a deliberate design decision: the loop half must always hand off rather
than terminate, because only the structured half can make the binary
committed/rejected determination.

The structured half called `normalise_booking_payload`, which failed
immediately: the LLM had not included `venue_id` in the handoff payload.
The Rasa webhook was never reached. The trace records the rejection:
`session.state_changed: structured → loop, rejection_reason: "normalisation
failed: missing venue_id"`. The bridge re-entered the loop half for round 2.

### Round 2 — area pivot, venue found, booking confirmed

The bridge prepended the rejection reason to the task text before calling
the planner again: `"Booking rejected: normalisation failed: missing
venue_id. The booking policy allows maximum 8 people. Find an alternative."
Planner ticket `tk_4d40feda` produced a revised plan. Sg_1 was now
`"Search for alternative venues in different areas"` — broader scope, three
estimated tool calls — and sg_2 remained `"Hand off to structured half with
alternative venue details"`.

This time the executor called `venue_search(near='Old Town', party_size=4)`,
which returned one result: The Royal Oak. The party size of 4 is within the
venue's 8-person policy ceiling. The LLM again omitted `venue_id` from the
`handoff_to_structured` call, but `build_forward_handoff` in `bridge.py`
intercepted this: it scanned `_TOOL_CALL_LOG` for the most recent
`venue_search` result and injected the `venue_id` automatically before
forwarding to the structured half. Rasa normalised the payload successfully,
issued a confirmation, and the session transitioned:
`executing → completed, booking_reference: "BK-4B5F7858"` at
`2026-05-25T13:16:46`. The bridge state moved to `complete` and the session
ended.

### Three architectural fixes that made this work

Three separate problems had to be solved before the two-round recovery
could succeed.

First, the **spiral detector** blocks a loop half from entering if it
detects that the same tool pattern has been seen before in the session. The
`_TOOL_CALL_LOG` module-level list persists across bridge rounds because it
lives at import scope, not session scope. Without clearing it, round 2 was
rejected on entry — the detector saw `venue_search` already in the log and
refused to proceed. The fix was to call `clear_log()` before each
`loop_half.run()` call in `bridge.py`, resetting the detector's memory so
round 2 starts with a clean slate.

Second, the LLM **omits `venue_id`** from the handoff payload in both
rounds. This is a reliable failure mode: the model sees the venue details
in its context window but does not propagate them faithfully into the
structured argument. `build_forward_handoff` addresses this by scanning
`_TOOL_CALL_LOG` for the most recent `venue_search` result and patching the
payload before it reaches the normaliser. Without this, round 2 would have
failed with the same `missing venue_id` rejection as round 1, and the
bridge would have looped indefinitely until `max_rounds_exceeded`.

Third, the loop half's tool registry originally included `complete_task`.
This gave the LLM an affordance to end the session without ever calling
`handoff_to_structured` — and it used it. The fix was to call
`tools.unregister("complete_task")` immediately after building the registry,
before the loop half is constructed. With the tool absent from the registry,
the LLM cannot complete the task; the only valid exit from the loop half is
the handoff. This is the principle from Lecture Slide 13/15: rather than
adding an instruction that says "always hand off before completing," remove
the affordance that allows the LLM to skip the handoff. Instructions can be
ignored; missing tools cannot.

## Citations

- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/trace.jsonl`
- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/tickets/tk_e05167c0/raw_output.json`
- `sessions/examples/ex7-handoff-bridge/sess_ad5adfe19a16/logs/tickets/tk_4d40feda/raw_output.json`
- `starter/handoff_bridge/bridge.py` — build_forward_handoff, clear_log
- `starter/handoff_bridge/run.py` — tools.unregister("complete_task")
