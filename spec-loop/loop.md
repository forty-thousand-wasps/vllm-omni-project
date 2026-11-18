# Review Notes

## How I read your answers

- **Last turn's OQ1, "RTF and TTFA should live in benchmarks/ but only be used to test /v1/realtime and /v1/live/sessions":** I read this as (b) with a narrow scope. The measuring tool is committed in this PR as benchmark code. It measures those two endpoints and nothing else, and isn't wired into CI. If "only be used to test" meant something else, such as a CI perf job, say so.

## Facts I checked

- **A correction:** last turn I said nothing in the repo measures these metrics. That's wrong. `vllm_omni/benchmarks/` (the package, not the top-level `benchmarks/` folder) already has a full-duplex benchmark for `/v1/realtime`:
  - `vllm bench serve --backend openai-realtime-duplex --dataset-name omni-duplex-eval` (`benchmarks/patch/patch.py:605,869`, and `entrypoints/cli/benchmark/omni_duplex_eval.py`);
  - per-model clients in `vllm_omni/clients/` (`duplex.py`, `minicpmo_4_5.py`, `personaplex.py`);
  - session metrics in `vllm_omni/benchmarks/duplex_session_metrics.py`: `ttfp` ("input stream start to first audio packet") and `stream_rtf` ("input stream start-to-last-audio receive time divided by total emitted audio duration; includes concurrent realtime input");
  - a CI perf case: `omni_duplex_eval_ci` in `ci_source_file_dependencies.yml`, and `tests/dfx/perf/tests/test_minicpmo_4_5_omni_duplex_eval.json`.
- **`stream_rtf` with real-time input sits near 1 by construction.** A full-duplex model emits about as much audio as it hears, and a paced client takes the clip's length to send it. So with paced input the metric mostly measures the clip's length, and "RTF ≤ 1" is hard to fail or to read. That's OQ2.
- **The RFC copy I saved in the scratchpad is gone** (the scratchpad was cleared). The spec's summary of the RFC targets stands, but I can't re-read the RFC's wording on how RTF was meant to be measured. If you want, I'll fetch #8085 again.

# Open Questions

Answer inline after each **A:**. Each **A:** answers exactly one question.

## Benchmark

1. (OQ1) **Where the `/v1/live/sessions` client plugs in.**

     (a) Extend the existing benchmark: a new `vllm bench serve` backend (for example `openai-live`) beside `openai-realtime-duplex`, a Live client in `vllm_omni/clients/`, and the same `duplex_session_metrics.py` report. Both endpoints then share the dataset, the media clock and the metric code. That touches shared benchmark code (`benchmarks/patch/patch.py`, the CLI) and makes the PR bigger.

     (b) A standalone script in the top-level `benchmarks/` folder (for example `benchmarks/live_sessions/`), with its own minimal clients for both endpoints and its own `ttfp` and RTF code that copies the existing definitions. Nothing shared changes, but the two endpoints' numbers come from separate code, and the `/v1/realtime` client is duplicated.

   Suggestion: (a).

   **A:**

2. (OQ2) **Which RTF.**

     (a) The existing `stream_rtf` on both endpoints, with input paced at real time. It's comparable with the numbers the duplex benchmark already produces, but it sits near 1 whenever the server keeps up, so it shows little beyond "kept up or not".

     (b) A processing RTF: send the input unpaced (as the E2E helper's `speed=0` does), and divide the time from stream start to the last audio packet by the input clip's duration. Values below 1 mean faster than real time, which is what "RTF ≤ 1.0 on PersonaPlex" asks. It needs a new metric, and I haven't checked that `/v1/realtime` duplex accepts unpaced input without its own backlog or pacing getting in the way.

     (c) Report both.

   Suggestion: (c).

   **A:**
