# Replies on #9220: drafts for the comments still unanswered

Draft only; nothing here is posted. The two replies already posted on 2026-10-29 (`handler.py`: unbounded `_pending`; `minicpmo_4_5_live.yaml`: `max_sessions: null`) are not repeated.

## linyueqian, review of 2026-11-15 (merge main)

> Thanks for this, and sorry for the delay. The branch now conflicts with main in 5 of its 54 changed files (`.buildkite/common/ci_source_file_dependencies.yml`, `.buildkite/cuda/test-merge.yml` and 3 more), and main has moved 590 commits past its base since your last update on 10-29. Is this still planned? If so, please merge main and I will review the new head. If it has been superseded or you have moved on, closing it is fine.

Yes, still planned, thanks. I'll merge main and ping you when the new head is up.

Since 10-29 the branch has two more commits:

- 68730c5d9 makes context handling per model.
  - MiniCPM-o still trims.
  - PersonaPlex no longer trims. Like the reference implementation, stage 0 attends to a sliding window of the last 3,000 frames, and the Live overlay sizes `max_model_len` for a whole 30-minute session.
  - A model that doesn't trim must fit a whole session in stage 0, or the server refuses to start.
  - The server also refuses to start with Live and `--api-server-count` other than 1.
  - It adds an `openai-live` backend to the existing duplex benchmark.
- 2b7d193ba fixes the PersonaPlex Live test overlay.

The L3 Live suites pass on 2b7d193ba: 8/8 on one RTX PRO 6000 Blackwell, including a PersonaPlex session past frame 3,000.

## amy-why-3459: `vllm_omni/model_executor/models/minicpmo_4_5/live/encoder.py:118`

> After preemption, the recompute re-runs earlier units' rows, which then keep the placeholder token embedding, and only a warning is logged. The request carries on with KV that was computed from placeholders instead of audio, so the model silently loses those units. Since the handler keeps the full unit history and can already re-render it (`render_resume`), raising here so the session resubmits seems better than continuing with corrupted context. Note that the handler currently resubmits only on `context_length_exceeded`, so that path would need extending too.

Agreed that continuing on placeholder KV is wrong. Fixed in 3b20aec03, differently from the suggestion: a preempted Live session is cancelled rather than resubmitted.

- **Why not raise there:** `apply_live_audio` runs in the batch's forward pass. An exception there would fail the engine step for every session on the engine, and the handler would never see it.
- **What happens now:**
  - Both models detect the recompute and set a per-request `live_preempted` row in stage 0's multimodal output.
  - The handler then ends the session with the generic `internal_server_error` and close 1013, logs the cause and counts it in `live_errors{reason="preempted"}`. The client can start a new session.
- **Why not resubmit:** preemption means stage 0 is out of KV cache, so a resubmitted session would likely be preempted again.
- **What running out of KV cache actually does:** I couldn't force a preemption on GPU. Between units a Live request waits while keeping its KV blocks, and vLLM 0.32 only preempts to make room for a running request. So when the cache fills, the sessions stall rather than being preempted, and the `max_input_backlog_s` limit ends them.
  - Each model's Live tests include a KV-exhaustion test that checks this.
  - The cancellation path is a safety net, covered by CPU tests.
- **The stall itself** needs a scheduler-side fix and is out of scope here; the PR description notes it.

## amy-why-3459: `vllm_omni/model_executor/models/minicpmo_4_5/minicpmo_4_5_omni_tts.py:1207`

> The Live TTS request spans the whole session and is re-prefilled for each unit, but only the `native_duplex` branch below drops `_request_codec_device_states` / `_request_codec_device_inputs` for the request. [...] Should that cleanup be `if native_duplex or live_budget is not None:`?

Yes, fixed in 3b20aec03 as suggested, so each Live unit's re-prefill resets the sampler's step counter and `min_tokens`.

That sampler only runs in the multi-frame Talker decode, which `minicpmo_4_5.yaml` enables only on NPU. So this mattered on NPU only; CUDA Live deployments were already correct.

## amy-why-3459: review summary (`segment_finished`)

> One more small one: `segment_finished` is plumbed through `messages.py` / `orchestrator.py` / `omni_base.py` / `outputs`, but the only reader I can find is the driver's debug log; the handler detects segment end from the stage-0 `finish_reason`. Either use it there, or drop the shared-code change from this PR.

Dropped in 3b20aec03. The `segment_finished` changes to `engine/messages.py`, `engine/orchestrator.py`, `entrypoints/omni_base.py` and `outputs/__init__.py` are reverted, so the PR no longer touches engine code.

## New top-level comment: PersonaPlex CI token

The PersonaPlex Live tests have a merge-pipeline step ("Omni · PersonaPlex Live Test") and load `nvidia/personaplex-7b-v1` from Hugging Face. Could a maintainer confirm that `VLLM_CI_HF_TOKEN` has accepted that checkpoint's gate before this merges? Otherwise the step fails with an authorization error.
