# Replies to amy-why-3459's review on #9220

Draft only; nothing here is posted.

## `vllm_omni/entrypoints/openai/live/handler.py:464` (unbounded `_pending`)

Fixed in 3b20aec03. `live_session_config` now has a required `max_input_backlog_s`. When the input audio queued behind the model exceeds it, the session ends as a server-side error: the client gets the generic `internal_server_error` and close 1013, and the server logs the cause and counts it in `live_errors{reason="backlog"}`. It covers both a client sending faster than real time and a stalled stage 0. Queued units aren't dropped, since that would silently cut audio out of the conversation. Both Live overlays set 10 s, where a real-time client stays under 1 s.

## `vllm_omni/deploy/minicpmo_4_5_live.yaml:13` (`max_sessions: null`)

Fixed in 3b20aec03. `max_sessions` is now required, as are `session_lifetime_s` and `max_input_backlog_s`, so there's no unbounded mode. The capacity derived for `duplex_max_sessions` always matches what the API server admits. The MiniCPM-o overlay sets `max_sessions: 8` and a stage 0 `max_model_len` of 16384, so 8 full contexts take ~18 GiB of stage 0 KV cache on a 96 GB GPU. The overlay comment and the docs give the arithmetic, including the ~150 MB per session of encoder cache that profiling doesn't count.

## `vllm_omni/model_executor/models/minicpmo_4_5/live/encoder.py:118` (recompute after preemption)

Agreed that continuing on placeholder KV is wrong. Fixed in 3b20aec03, differently from the suggestion: a preempted Live session is cancelled rather than resubmitted.

- **Why not raise there:** `apply_live_audio` runs in the batch's forward pass. An exception there fails the engine step for every session on the engine, not just this request, and the handler would never see it. PersonaPlex's Live stage 0 already behaved like that: on a recompute it raised `ValueError` for a negative offset.
- **How it works now:**
  - Both models detect the recompute (MiniCPM-o: more placeholder rows than embeddings left; PersonaPlex: a span before the current frame) and set a per-request `live_preempted` row in stage 0's multimodal output.
  - The handler ends the session like the backlog case (generic `internal_server_error`, close 1013, cause in the log and `live_errors{reason="preempted"}`), which aborts the request. The client can start a new session.
- **Why cancel rather than resubmit:** preemption means stage 0 is out of KV cache, so a resubmitted session would likely be preempted again. With the required `max_sessions` and the stage 0 context sizing above, the admitted sessions fit, so this is a safety net.
- **What running out of KV cache actually does:** I tried to force a preemption on GPU, with a stage 0 KV cache of one full context and two growing sessions, and never got one. Scheduler logging showed why. Between units, a Live request waits while keeping its KV blocks, and vLLM 0.32 only preempts to make room for a running request. So both sessions sat waiting, each holding about half the cache and needing one more block, with nothing running: a deadlock. The backlog limit ended them and the server recovered. So preemption needs two Live requests decoding at the same moment at a block boundary, which is rare, and the cancellation path is a safety net covered by CPU tests.
- **Tests:** each model's Live test file now has a KV-exhaustion test with that setup. It checks that a stalled session is ended with `internal_server_error` / 1013 rather than hanging, that `live_errors{reason="backlog"}` rose, and that a new session afterwards gets a spoken reply.
- **The deadlock itself** is out of scope here; the PR description has a note asking for a separate PR (a startup KV sizing check, a stall watchdog, or a scheduler change). Within this PR, the overlays are sized so that the admitted sessions fit in stage 0's KV cache.
- **Also fixed:** the driver didn't stop a request when the session was aborted from inside the request's own output callback. That also affected the existing `output.error` path.

## `vllm_omni/model_executor/models/minicpmo_4_5/minicpmo_4_5_omni_tts.py:1207` (sampler caches across Live units)

Fixed in 3b20aec03 as suggested: the cleanup condition is now `if native_duplex or live_budget is not None:`, so each Live unit's re-prefill resets the sampler's device-side step counter and `min_tokens`. For the record, that sampler only runs in the multi-frame (K-step) Talker decode, which `minicpmo_4_5.yaml` enables only on NPU. CUDA Live deployments take the single-frame batched path, which was already correct, so this mattered for NPU only and the CUDA E2E runs don't exercise it.

## Review summary (`segment_finished`)

Dropped in 3b20aec03: the `segment_finished` additions to `engine/messages.py`, `engine/orchestrator.py`, `entrypoints/omni_base.py` and `outputs/__init__.py` are reverted, so the PR no longer touches engine code. The handler keeps detecting segment ends from stage 0's `finish_reason`.

## PersonaPlex CI (new in this update)

The PersonaPlex Live tests now have a merge-pipeline step ("Omni · PersonaPlex Live Test") and load `nvidia/personaplex-7b-v1` from Hugging Face. Could a maintainer confirm that the CI token (`VLLM_CI_HF_TOKEN`) has accepted that checkpoint's gate before this merges? Otherwise the step fails with an authorization error.
