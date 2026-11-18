# Review Notes

## Facts I checked

- **Unpaced input already exists in the duplex benchmark.** `vllm bench omni-duplex-eval generate` takes `--pace realtime|as-fast-as-possible` (`entrypoints/cli/benchmark/omni_duplex_eval.py:100`, `benchmarks/duplex/omni_duplex_eval_runner.py:362`). The only restriction is that `server_vad` turn detection requires `realtime`, which doesn't affect native duplex. So the processing RTF needs a new metric field, not a new pacing mode. I haven't checked whether the `vllm bench serve --backend openai-realtime-duplex` path honours `--pace` too. If it doesn't, the processing RTF runs through `omni-duplex-eval generate`.
- **A side effect on CI:** the benchmark changes touch `vllm_omni/benchmarks/duplex_session_metrics.py`, `vllm_omni/benchmarks/patch/**` and the CLI. These are listed under `omni_duplex_eval_ci` in `.buildkite/common/ci_source_file_dependencies.yml`, so the PR will also run that existing MiniCPM-o duplex eval job. Adding a field to the metric report shouldn't change its baseline (`tests/dfx/perf/tests/test_minicpmo_4_5_omni_duplex_eval.json`), but I haven't read how that job compares reports.

## How I read your answers

- **OQ1 (a), OQ2 (c):** folded into sections 3 and 12. The backend name `openai-live` and the client module's name are placeholders for the build.

# Open Questions

Answer inline after each **A:**. Each **A:** answers exactly one question.

None.
