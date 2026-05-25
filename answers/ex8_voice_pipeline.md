# Ex8 — Voice pipeline

## Your answer

The voice pipeline was run and verified in both modes. In text mode
(session `sess_ff793177bc67`), a five-turn conversation was conducted
via stdin. In voice mode (session `sess_8f34b2852254`), a full
end-to-end voice round-trip completed: speech was captured from the
microphone, transcribed by Speechmatics, sent to the manager persona,
and the reply was synthesised by ElevenLabs and played back through
the speakers.

The voice session comprised four turns. In turn 0, the user spoke a
booking request for six people on Tuesday at 6pm and provided a contact
number; Speechmatics transcribed this to `"Hi. Good evening. I'd like
to make a booking for tomorrow. Tuesday for six people. And it's at
6 p.m. My telephone number is 07768 868891."` The manager (Alasdair
MacLeod, backed by Llama-3.3-70B-Instruct on Nebius) responded in
character: `"Aye, we can do that. I'll pencil you in for Tuesday at
6 p.m. What's the deposit?"` In turn 1 the deposit of £200 was
confirmed — within the £300 ceiling — so the manager accepted. The
conversation concluded naturally at turn 3 with `"Aye, see you
tomorrow."` All eight trace events carry `mode: "voice"`, distinguishing
this session from text-mode runs.

Both modes share an identical trace contract: every utterance produces
a `voice.utterance_in` event (actor: user) followed by a
`voice.utterance_out` event (actor: manager), each with payload fields
`{text, turn, mode}`. This means downstream grading and log analysis
work without knowing which transport was active. The `mode` field is
the only difference between a voice trace and a text trace.

The `ManagerPersona` class maintains a full conversation history and
passes it on every LLM call, so the manager remembers details across
turns — party size, deposit, and contact number are all present in
context by the time the booking is confirmed. Graceful degradation is
implemented: if `SPEECHMATICS_KEY` is missing or the `speechmatics`
package is not installed, `run_voice_mode` logs a visible warning and
falls through to `run_text_mode` rather than crashing.

## Citations

- `sessions/homework/ex8/sess_8f34b2852254/logs/trace.jsonl` — voice mode, 4 turns
- `sessions/homework/ex8/sess_ff793177bc67/logs/trace.jsonl` — text mode, 5 turns
- `starter/voice_pipeline/voice_loop.py` — run_text_mode, run_voice_mode, graceful degradation
- `starter/voice_pipeline/manager_persona.py` — ManagerPersona.respond, history management
