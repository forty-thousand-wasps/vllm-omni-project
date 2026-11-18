# Review Notes

## How I read your answers

- **Last turn's OQ1 (a):** PersonaPlex gets one unpaced long-session L3 test past frame 3,000. The no-trim overflow is covered by the L1 handler tests only, and section 12's L1 list now names it.
- **Last turn's OQ2, from chat:** `_fail` logs, and every caller passes a message. Section 7 has a new "Ending a session on an error" paragraph covering both `_fail` and `_abort`. That replaces all three of OQ2's options.

## Facts I checked

- **`_fail` has five callers** in `handler.py`:
  - the session's `run` loop (`:152`) and a failed background task (`:174`), each after its own `logger.exception`;
  - the two context failures (`:550`, `:554`);
  - `_on_engine_error` (`:580`), after its own `logger.error`;
  - an `error` on an engine output (`:586`), unlogged today.

  Folding the three existing log lines into `_fail` keeps one line per failure. The two `logger.exception` sites keep their tracebacks by passing `exc`.

# Open Questions

Answer inline after each **A:**. Each **A:** answers exactly one question.

## RFC performance targets

1. (OQ1) **Where the RTF and time-to-first-audio client lives.** Nothing in the repo measures these for either endpoint today (`benchmarks/` has no duplex or Live benchmark).

     (a) A local client in `vllm-omni-aux`, run through a run-remote recipe, with the method and results in the PR description. Nothing is added to the PR.

     (b) A benchmark committed in this PR under `benchmarks/`, driving both `/v1/realtime?duplex=1` and `/v1/live/sessions`. That adds a new file area to the change and to review.

   Suggestion: (a).

   **A:**
