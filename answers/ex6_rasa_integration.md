# Ex6 — Rasa structured half

## Your answer

The Rasa structured half was implemented and verified in session
`sess_db412f3483aa`. The `RasaStructuredHalf` subclass overrides
`run()` to send a booking intent to Rasa's REST webhook and interpret
the response. The data flow works as follows: the loop half produces
raw booking data and passes it to the structured half, which calls
`normalise_booking_payload` in `validator.py` to convert it into a
well-typed, Rasa-shaped message. That message is then POSTed via urllib
to the Rasa webhook, and the response is parsed for the custom slots
`{action: committed}` or `{action: rejected}`. In this session the
booking was accepted and the trace records a state transition from
`executing` to `completed` at `2026-05-25T11:10:11`, with booking
reference `BK-7D401E9E`.

Three design decisions shaped the implementation. First, when
`normalise_booking_payload` raises `ValidationFailed` — for example
because a required field such as `venue_id` is missing — that exception
is caught inside `run()` and converted into a `HalfResult` with
`success=False`. The structured half's contract requires it to always
return a `HalfResult`, never propagate an exception up to the bridge.
Second, network errors from the Rasa endpoint return `success=False`
with error code `SA_EXT_SERVICE_UNAVAILABLE`, leaving the retry
decision to the caller rather than hardcoding it into the half. Third,
the `sender_id` passed to Rasa is computed as a hash of the venue name,
date, and time. This ensures that if the bridge retries the same booking
within a session, the Rasa conversation tracker sees a consistent
identity and does not treat the retry as a fresh, unrelated request.

For offline testing, the implementation spawns a stdlib `http.server`
thread that mimics the Rasa webhook and always returns a confirmation.
This allows the structured half to be tested without a running Rasa
container. The rejection path — where Rasa declines a booking because
the party size exceeds the venue's capacity — is exercised in Ex7,
where a party of 12 at Haymarket Tap (maximum 8) triggers the bridge's
retry logic.

## Citations

- `sessions/examples/ex6-rasa-half/sess_db412f3483aa/logs/trace.jsonl`
- `starter/rasa_half/validator.py` — normalise_booking_payload
- `starter/rasa_half/structured_half.py` — RasaStructuredHalf.run, mock server
