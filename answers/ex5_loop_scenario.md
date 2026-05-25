# Ex5 — Edinburgh research loop scenario

## Your answer

The Edinburgh research loop was first run in session `sess_7a751e32fa97` to
observe the loop half and the dataflow integrity check in isolation. A second
run in session `sess_0111f73b3160` added a log-recovery fix that ensures the
flyer always contains real tool output data regardless of what the LLM passes
to `generate_flyer`.

### Session sess_7a751e32fa97 — demonstrating the hallucination problem

The planner decomposed the task into five subgoals, all assigned to the loop
half. The executor began by calling `venue_search(near='Haymarket',
party_size=6, budget_max_gbp=800)`, which returned one result — Haymarket
Tap. It then fetched the weather with `get_weather('Edinburgh', '2026-04-25')`,
receiving a forecast of cloudy skies and 12°C.

At this point the executor attempted to calculate the booking cost, but made
a critical error: it called `calculate_cost` with
`venue_id='v_edinburgh_wizard_tower'` — a venue name that had never appeared
in any prior tool output. This was a hallucination. The tool returned
`success=false` with the message `"calculate_cost: unknown venue
'v_edinburgh_wizard_tower'"`. The executor recovered by issuing a second
venue search, this time `venue_search(near='Old Town', party_size=12)`, which
returned The Royal Oak. A subsequent
`calculate_cost('royal_oak', 12, 3, 'three_course_meal')` succeeded, returning
a total of £2,540 with a deposit of £762.

The final step was to generate the HTML flyer. The LLM did not propagate the
tool outputs into the `generate_flyer` call. Instead it passed placeholder
`event_details` containing `venue="Edinburgh Castle"` and `name="Sample Event"`
— neither value appeared anywhere in the tool call log. The flyer was written
to `workspace/flyer.html` with this fabricated data.

This is precisely the class of failure that `verify_dataflow` is designed to
catch. A human reviewer scanning the generated flyer would see a formatted
page, an Edinburgh venue name, and cost figures that look broadly plausible.
Nothing would obviously flag as wrong. `verify_dataflow`, by contrast,
compares every value extracted from the flyer against the ground truth in
`_TOOL_CALL_LOG`. Since "Edinburgh Castle" never appeared in any tool output,
it is returned in `unverified_facts` and the check fails. Without this
automated layer, the fabrication would have gone undetected.

### Session sess_0111f73b3160 — applying the log-recovery fix

The same hallucination pattern recurred in the second run. After successfully
calling `venue_search('Haymarket', party_size=6)` (0 results at
`budget_max_gbp=100`, 1 result at `budget_max_gbp=200`), then
`get_weather('Edinburgh', '2026-04-25')` (cloudy, 12°C), then
`calculate_cost('royal_oak', 10, 3, 'bar_snacks')` (total £1103, deposit
£330), the LLM called `generate_flyer` with
`{"venue": "The Royal Terrace", "date": "2023-08-15", "weather": "Sunny, 22°C",
"total_cost": "£750"}` — entirely fabricated, none of it from any tool output.

The fix intercepts this at the `_flyer_adapter` layer in the tool registry.
Before calling `generate_flyer`, the adapter passes the LLM's `event_details`
through `_build_event_details_from_log`, which scans `_TOOL_CALL_LOG` and
overwrites the fact-fields:

- `venue_name` and `venue_address` come from the most recent successful
  `venue_search` result: `results[0]["name"]` and `results[0]["address"]`.
- `condition` and `temperature_c` come from the most recent `get_weather`
  output dict.
- `total_gbp` and `deposit_required_gbp` come from the most recent
  `calculate_cost` output dict.

The LLM's fabricated values for those keys are silently discarded. The final
flyer shows The Royal Oak, 1 Infirmary St, Edinburgh EH1 1LT; cloudy, 12°C;
total £1103, deposit £330. `verify_dataflow` verified 4 facts against the
log.

### Why structured tool outputs make this possible

`_build_event_details_from_log` works with a single dict lookup per field:
`rec.output["results"][0]["name"]`, `rec.output["condition"]`,
`rec.output["total_gbp"]`. This is only reliable because every tool returns a
typed Python dict, not a prose string. If `venue_search` had returned
`"Found 1 venue: The Royal Oak at 1 Infirmary St"`, extracting the venue name
would require a regex and handling of typos and format variations — brittle and
unreliable. The LLM receives the human-readable `summary` field for its
reasoning; the infrastructure receives the `output` dict for reliable
programmatic extraction. Keeping those two representations separate is what
makes post-hoc log recovery feasible.

## Citations

- `sessions/examples/ex5-edinburgh-research/sess_7a751e32fa97/logs/trace.jsonl`
- `sessions/examples/ex5-edinburgh-research/sess_7a751e32fa97/workspace/flyer.html`
- `sessions/examples/ex5-edinburgh-research/sess_0111f73b3160/logs/trace.jsonl`
- `sessions/examples/ex5-edinburgh-research/sess_0111f73b3160/workspace/flyer.html`
- `starter/edinburgh_research/integrity.py` — verify_dataflow
- `starter/edinburgh_research/tools.py` — _build_event_details_from_log, _flyer_adapter
