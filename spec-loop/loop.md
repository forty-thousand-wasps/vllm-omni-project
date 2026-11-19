# Review Notes

## Build attempt 1 (2026-11-18)

Built from `spec.md` on `feat/live-sessions` (vllm-omni), uncommitted until the CPU suites pass. Every item below is inside an area the spec covers; nothing outside it was needed.

### Choices made inside covered areas

- **`trim_context` is a base method, not `None`.** mypy 1.11 rejects overriding a `None`-typed attribute with a method, so the spec's fallback applies:
  - the base `trim_context` raises `NotImplementedError`;
  - a classmethod `trims_context()` reports whether a processor overrides it;
  - the handler and the startup check ask `trims_context()`, and never call the base method.
- **The generic trim is `trim_to_fraction`.** MiniCPM-o's `trim_context` calls it.
- **`render_resume` is no longer abstract.** It raises unless a trimming processor overrides it, so PersonaPlex dropped its own.
- **Sizing hooks:**
  - `units_per_second()`, which defaults to `1000 / unit_ms`;
  - `max_prefix_tokens(model_path)`;
  - a third, `max_tokens_per_unit()` (default 1), so the check doesn't assume one token per unit. The obvious name, `max_unit_tokens`, is already a MiniCPM-o attribute.
- **PersonaPlex prefill bound.** It is 1,105 tokens (largest voice `VARM2.pt`: 69 rows, 2 × 6 silence frames, 1,024 persona tokens). I checked it on the local checkpoint, where the default persona is 26 tokens.
  - The overlay sets stage 0 `max_model_len: 24000` and `hf_overrides.temporal_config.max_position_embeddings: 24000`. The bound is 22,500 + 1,105 = 23,605.
  - The overlay repeats the base stage's `mimi_cuda_graphs` / `depformer_cuda_graphs`, because `hf_overrides` isn't deep-merged.
  - vLLM applies a nested dict in `hf_overrides` to the nested `temporal_config`, which is what `get_text_config()` returns.
- **New helpers in `personaplex/prompting.py`:** `personaplex_persona_tokens`, `personaplex_voice_names` and `personaplex_max_voice_rows`.
- **Persona limit errors** go out as `invalid_value` with `param="instructions"`, even when `input` messages pushed the persona over the limit.
- **The startup sizing check** is skipped when the engine client has no `model_config`, as with the test fakes.
- **E2E (PersonaPlex):**
  - The conversation, long-session and concurrency tests share one server: `personaplex_live.yaml` with `max_input_backlog_s: 600`, so the unpaced session isn't ended for backlog.
  - The 600-frame overlay is gone: the startup check would refuse it, since the prefill bound alone is 1,105.
  - The long session checks replies by **media time** (position in the output audio), because unpaced arrival times say nothing. The helper gained `speed`, `media_s`, `drain()`, `voiced_blocks_media()` and `answered_media()`.
  - **The KV-exhaustion overlay** is now 224 blocks of 16. vLLM admits a sliding-window request only with ⌈(2999 + 512)/16⌉ + 1 = 221 blocks. Two sessions should then stall after about 135 s each, so the window grew from 120 s to 300 s. This is arithmetic, not a run.
- **Docs:**
  - I also corrected an existing error: the page said every non-null `delegation` was rejected, but the code accepts a tool-free `responses` delegation.
- **Benchmark** (built by a subagent; I reviewed its report and ran the tests):
  - The backend is `openai-live`, in `omni-duplex-eval generate` and in `vllm bench serve --omni`, with the client in `vllm_omni/clients/live.py`.
  - It also touched `entrypoints/cli/benchmark/cli_args.py`, so that `vllm bench serve` accepts the dataset with the new backend, and `omni_duplex_eval_serving.py`, for Live's default instructions.
  - `stream_processing_rtf` appears only in unpaced runs. In paced runs it would be at least 1 by construction.
  - A Live run counts as drained after 5 s with no server event. Live has no end-of-response event.
  - Unpaced runs go only through `generate`; the `serve` path has always been paced.
  - Video isn't sent over Live.

### Not verified yet (GPU, each run needs your approval)

- Section 12's three checks:
  - the other PersonaPlex deployments behave as before;
  - PersonaPlex past the window, including whether vLLM accepts the overrides;
  - the RFC performance targets.
- The L3 suites, including the new long-session test, whose media-time alignment assumes the output audio starts in step with the input.
- The new benchmark backend against a real server.

# Open Questions

Answer inline after each **A:**. Each **A:** answers exactly one question.

None.
