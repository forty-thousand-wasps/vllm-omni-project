# Review Notes

## How I read your answers

- **Last turn's OQ1, "(a), but refuse to start if --api-server-count N != 1, declaring it UB":** I read this as dropping multi-process support. When Live is enabled and N ≠ 1, the server refuses to start, and the docs call Live with several API server processes unsupported, with undefined behaviour. Everything that depended on the split is gone from the spec: the `max_sessions / N` split, the divisibility check, and the `livesum` gauge mode. If you meant to keep the split code behind the refusal, say so, though nothing would run it.
- **Last turn's OQ2 (a):** the persona limit is a 1,024-token processor constant. The overlay's `max_model_len` is 22,500 + the largest bundled voice prompt + two silence prefills + 1,024. The build works out the exact number.
- **The `playback_speed` idea from chat** became a `speed` parameter on the E2E helper's send loop (0 = unpaced), not a `live_session_config` key. The server has no pacing to speed up. This is in section 12, and OQ1 is rewritten around it.

## Facts I checked

- **The E2E helper paces input at real time** (`tests/e2e/online_serving/helpers/live_sessions.py:7,141`: 100 ms chunks, with `asyncio.sleep` to hold the pace). The handler sends each unit as soon as the previous segment finishes, and has no clock.
- **A no-trim overflow before expiry needs a model faster than real time.** The startup sizing check makes `max_model_len` cover `session_lifetime_s` of real-time input, and the lifetime is wall-clock. An unpaced client overflows the context after about `max_model_len / (12.5 × speed-up)` seconds. With a real-time factor of 1 or more, the session expires first however fast the client sends.
- **`init_live_app_state` runs in `init_app_state` at startup**, after `api_server_count` is set on `app.state`. I believe an exception there stops the server before it listens, as the startup failure should. I haven't run it.

## Things to check

- **The unpaced tests' replies.** With `speed=0`, PersonaPlex hears its input faster than real time. Its replies are still generated one frame per input frame, but a test that checks a spoken reply follows the user's speech needs enough silence after the speech for the reply to come out before the clip ends.
- **PersonaPlex's L3 KV-exhaustion test** still needs its cache sizing redone for the sliding window, as a test detail during the build.

# Open Questions

Answer inline after each **A:**. Each **A:** answers exactly one question.

## PersonaPlex tests

1. (OQ1) **The L3 tests that replace the 600-frame trim test.** Both options use the helper's new `speed` parameter.

     (a) One long-session test, unpaced (`speed=0`): stream one session past frame 3,000 and check that replies keep coming after the window starts sliding. At a real-time factor of 0.5 that's about 2 minutes instead of 4.5. The no-trim overflow path (`_abort`, 1013, `live_errors{reason="context"}`) is covered by L1 only.

     (b) As (a), plus an unpaced L3 overflow test. Its test overlay has a small stage 0 `max_model_len` (for example 600 frames), a `session_lifetime_s` that just passes the startup check, and a large `max_input_backlog_s`. It checks for `internal_server_error` / 1013 and `live_errors{reason="context"}`. At a real-time factor of 0.5, it overflows in about 24 s. It only works if PersonaPlex runs comfortably faster than real time on the CI GPU (`h100_1`). That's unmeasured until the RFC performance run, and if it doesn't hold, the test is dropped and the overflow stays L1-only.

   Suggestion: (b).

   **A:**

## Context failures

2. (OQ2) **Logging for the two context failures that close 1011.** `_fail` ignores its exception and logs nothing. A session ended by an overflow on its first unit, or by a trim that drops nothing, leaves only a metric increment.

     (a) Log at those two call sites only: one error line with the session, the prefix size and `max_model_len`.

     (b) Make `_fail` log its exception for every caller. Engine errors would then log twice, since `_on_engine_error` already logs before calling it.

     (c) Leave them unlogged.

   Suggestion: (a).

   **A:**

## RFC performance targets

3. (OQ3) **Where the RTF and time-to-first-audio client lives.** Nothing in the repo measures these for either endpoint today (`benchmarks/` has no duplex or Live benchmark).

     (a) A local client in `vllm-omni-aux`, run through a run-remote recipe, with the method and results in the PR description. Nothing is added to the PR.

     (b) A benchmark committed in this PR under `benchmarks/`, driving both `/v1/realtime?duplex=1` and `/v1/live/sessions`. That adds a new file area to the change and to review.

   Suggestion: (a).

   **A:**
