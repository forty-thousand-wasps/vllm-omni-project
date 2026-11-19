# Revisions to `live_sessions_native_plan.md`

Where the implementation on `vllm-omni` branch `feat/live-sessions-v2` (from
`origin/main` `0e6791e9c`) diverges from the plan, with the reason. Nothing is
committed.

## Generic Live layer

1. **Processors may open a request with a prefix-only segment
   (`render_prefix`).** The orchestrator re-sends a request's *first* prompt
   (`model_intermediate_buffer` included) to every downstream stage at each
   streaming update (async-chunk prewarm). MiniCPM-o's units carry mel
   features, and a resubmission carries every kept unit, so they cannot sit in
   the first prompt. MiniCPM-o's first prompt is the system prompt and
   reference voice alone, sampled with `max_tokens=1` and a forced
   `<|listen|>` that is never fed; units follow as updates. This costs one
   extra prefill round trip at session start and after a resubmission.
   PersonaPlex keeps the prefix in its first frame (its first prompt is small).
2. **`context_length_exceeded` arrives as an exception**, not an output:
   `AsyncOmni.generate` raises `RuntimeError("context_length_exceeded: ...")`
   after `OmniARScheduler._streaming_update_overflows` finishes the request.
   The driver passes the epoch with the error so a stale epoch's failure is
   ignored.
3. **No input-transcription config to reject.** The Live `SessionConfig` has
   no transcription field (`audio` has only `format` and `output.voice`), so
   the server simply never emits `session.input_transcript.delta`.
4. **`session.update` accepts nothing.** With delegation unsupported, every
   updatable field is rejected; an empty update replies `session.updated`.
4a. **A tool-free `delegation.responses` is accepted** (plan: any
   `delegation` rejected). LiveKit's `GPTLiveModel` always sends one
   (`{type: "responses", responses: {model: ...}}`), so rejecting it failed
   every LiveKit session at `session.start` (first E2E run). The model must be
   the served model; tools, a client delegation and every other non-null
   `responses` field remain `unsupported_session_config`. The delegation is
   echoed in the session resource and delegates nothing.
5. **`ProcessorContext` kept** (tokenizer, `max_model_len`, model name, model
   path): both processors use it.
6. **Session cap** is `SessionSlots` on the app state, acquired before
   `session.start` validation and released on close. A refused `session.start`
   increments `live_rejected_sessions`.
7. **Metrics trimmed** to active/rejected sessions, audio seconds, context
   trims and errors; the `vad` label is gone.

## Config

8. **The Live logits processor is installed on both config paths.** Production
   stage configs come from `omni_config.VllmOmniConfig.from_pipeline_config`,
   not `stage_config._build_engine_args`; both now call
   `add_live_session_logits_processor`.
9. **`live_session_config.max_sessions` also sizes model-owned streaming
   state.** `streaming_session_capacity(deploy)` feeds the existing
   per-stage `duplex_max_sessions` model-config field: the duplex value in
   duplex mode, otherwise the Live `max_sessions` (1 when unset). PersonaPlex's
   Code2Wav holds one Mimi instance per slot, so sizing from `max_num_seqs`
   would load up to ten copies. MiniCPM-o (`max_sessions: null`) is unchanged.

## MiniCPM-o 4.5

10. **Unit audio rows use `<|tts_pad|>` as the placeholder token** (never fed
    otherwise: the decoder always forbids it), so the Thinker's Live branch
    finds the rows by id; the reference voice keeps the multimodal `<unk>`
    rows.
11. **`live_duplex_unit` kept**: the prompt writes `<|audio_start|>` and
    `<|audio_end|>` itself so `previous: ...` text can sit between the
    reference voice and `<|audio_end|>`, as in the official context layout.
12. **Mel features travel as raw float32 bytes**: `model_intermediate_buffer`
    is `dict[str, Any]`, and vLLM's msgpack decoder does not rebuild tensors in
    untyped fields.
13. **Decode parity follows MiniCPM-o-Demo, not the checkpoint.** Per user
    decision: force-listen on the first 3 units, the 28-character unit text
    cap, the 1.1 `<|turn_eos|>` length penalty, special tokens in the
    512-token window. Each decision is applied as a one-hot row. The decoded
    history and turn state are kept per session id in the worker (vLLM re-adds
    the request on every update), with an LRU bound of 1024 sessions. Rows
    are skipped after their terminator, which covers an asynchronous
    scheduler's lookahead step.
14. **Talker handoff locates unit rows through the latent ledger**
    (`latent_input_ids` / `latent_positions`): the latest position-contiguous
    block whose inputs are the unit's fed tokens.
15. **Talker Live condition honours upstream's `_normalize` flag.**
16. **Encoder cache after preemption**: a recompute re-runs earlier units'
    rows whose embeddings are gone; they keep the placeholder embedding (a
    warning is logged).

## PersonaPlex

17. **Shared helpers moved to `personaplex/prompting.py`** (frame geometry,
    lockstep tokens, `wrap_with_system_tags`, voice/tokenizer loaders,
    `personaplex_prefill_slots`, plus `personaplex_model_dir`, which resolves
    a repo id to the local snapshot). `duplex/config.py`, `duplex/policy.py`
    and `duplex/stage0.py` import from it; their public names are unchanged.
18. **Frame codes reach Code2Wav as a per-request list from
    `make_omni_output`** (the runner indexes list-valued outputs by batch
    position), holding the codes of the frame that step's `preprocess`
    finished, else an empty tensor. No runner change.
19. **Resubmission resets the conversation**: the new request replays the
    persona and voice prefill and the newest frame only (agent codes of
    earlier frames are not kept on the API server).
20. **Voice ids**: `default`, or a bundled voice-prompt name with or without
    `.pt`; checked at `session.start` through the prefill-slot computation.

## Fixes from the first E2E runs (MiniCPM-o, g7e.2xlarge)

22. **A resumable request keeps its first chunk's `max_tokens`.** vLLM's
    `_update_request_as_session` replaces the sampling params but not
    `Request.max_tokens`, so the prefix segment's `max_tokens=1` capped every
    unit at one token (`<|speak|>`, then `length`). The prefix now uses the
    unit budget (20) and ends on its forced `<|listen|>` stop token.
23. **Processor assets preload at startup** (`LiveSessionProcessor.preload`,
    in a thread): loading MiniCPM-o's remote-code processor inside the first
    `session.start` blocked the event loop for ~3.6 s.
24. **Output transcripts decode each delta's tokens as they arrive**
    (MiniCPM-o), which matches the Demo's `decode(res_ids)`. Missing spaces
    between units ("Franceis") are the model's own tokens: depending on how
    the speech falls on unit boundaries, it sometimes starts a unit's first
    word without a leading space (`"is"`=285 instead of `" is"`=374); HF
    decoding adds no space at `<|speak|>`. A word split by the 28-character
    unit cap can also show a stray space.
25. **The real-time backlog warning logs only when the backlog grows**, not on
    every append.
26. **Debug logging is plain `logger.debug`** (driver outputs and prompt
    lengths, context trims, dropped-unit text); the recipes enable it for the
    modified modules only through `VLLM_LOGGING_CONFIG_PATH`
    (`vllm-omni-aux/logging/live-sessions-debug.json`).

E2E result (manual run on the instance, warm server): all four scenarios
answer correctly in context with audio; first audio 0.4-1.9 s after the
user's speech ends; greedy output identical across runs. The first session
after startup pays a one-time warm-up (~6 s before the first audio).

## Recipes

27. `minicpmo-e2e-test` and `minicpmo-replay-test` drop the Whisper service
    and the call tracer (`RUN_REMOTE_CALL_TRACE`, not ported); new
    `personaplex-e2e-test`; `qwen-e2e-test` removed. The aux overlay
    `deploy-configs/minicpm-o/live-1gpu.yaml` no longer gives GPU memory back
    for Whisper.

## Fixes from the context-overflow test (`minicpmo-overflow-test`)

Stage 0 at `max_model_len: 1024`, one 6-minute session of Piper utterances.

28. **MiniCPM-o's `previous` text is capped at its newest 500 tokens**
    (the checkpoint's `DuplexWindowConfig.context_previous_max_tokens`).
    Uncapped, it grew with every trim until the prefix filled the context:
    the server then trimmed on every unit (80 trims in 6 minutes) and the
    model stopped answering.
29. **Live applies native duplex's short-unit hold** (`hold_short_unit`, fewer
    than 5 codec frames at a unit end wait for the next unit). A one-word
    unit flushed alone left Token2Wav's conformer cache 5 wide instead of 6,
    and the next chunk's assertion killed the Stage 2 engine. Seen after a
    resubmission that started mid-reply; reachable without trimming too.
30. Each overflow logs two ERROR lines (`omni_ar_scheduler`,
    `async_omni_base`) although the handler recovers; upstream logging, left
    as is.

Result after both fixes: 20 trims in 6 minutes, the prefix steady at 575
tokens, 12-13 units kept per trim, every utterance answered, no errors.


## Concurrency and session-limit test (`minicpmo-concurrency-test`)

`max_sessions: 4`; 4 sessions of Piper utterances for 4 minutes, a 5th
`session.start` while they run, and a new session in session 0's slot after
it closes at 90 s.

31. Over the limit: `error` (`server_error`, `session_limit_reached`,
    `client_event_id` echoed), close 1013, no `session.closed`. The freed slot
    is reused at once. Verified in both runs.
32. Warm run: every utterance answered in all 5 sessions, transcript p95
    lag 1.1 s behind real time, GPU memory 52.7 GB idle and 68.2 GB peak.
    The short-unit hold (29) fires under load (`holding 2 codec frames`).
33. On a cold CUDA JIT cache (first run after the instance's driver
    update; Stage 1 graph capture 69 s instead of 1 s), Stage 1 stalled for
    ~50 s on the first batched requests: text in real time, no audio for
    any session until 59 s. The same cold start left PersonaPlex's first
    session silent (`personaplex-e2e-test`, Stage 1 never produced audio in
    its 12 s). Neither recurs with the cache warm; environment, not Live.
34. In the cold run one session stopped speaking at 75 s and emitted only
    `<|listen|>` for the remaining 2.5 minutes while the others kept
    answering. Not reproduced in the warm run; cause unknown.
35. Warm 8-minute MiniCPM-o rerun: every session kept answering (33/34 at
    worst), p95 lag 1.2 s; no session stuck in `<|listen|>` (34).
36. `personaplex-concurrency-test` (stock `personaplex_live.yaml`,
    `max_sessions: 2`): both sessions and the session in the freed slot
    answered every utterance; a 3rd `session.start` got
    `session_limit_reached` and 1013. Code2Wav gave the two sessions Mimi
    slots 0 and 1 and the reused session slot 0. One session reached the
    3000-frame context at 233 s, was trimmed and resubmitted (epoch 2, same
    Mimi slot) and kept producing audio. GPU memory 41.4 GB idle, 42.1 GB
    peak.

## Rebase onto upstream main (vLLM 0.32)

The branch is one commit on `da572cddb`; the recipes moved to `vllm==0.32.0`
(upstream aligned with 0.31 in #8459 and 0.32 in #9066).

37. Conflicts: `streaming_session_capacity()` returns
    `duplex_session.worker_max_sessions` for duplex deploys (new upstream);
    `duplex/policy.py` keeps only upstream's remaining re-exports, now from
    `prompting.py`, which also takes upstream's one-pass `voices.tgz` reader;
    the PersonaPlex Talker encodes Live appends before upstream's batched
    duplex preparation. `live/stage0.py` follows upstream's
    `_build_frame_embed` (no `prev_agent`) and encoder-only Mimi init.
38. **Mis-merge, fixed:** upstream #7929 moved the Talker's default output
    path to `_make_omni_output_single_frame`; the Live metadata
    (`minicpmo_live`, `live_turn_end`) had merged into the multi-frame body
    only, so the producer treated Live units as one-shot requests and every
    reply stopped after its first ~1 s chunk. `_live_output_meta()` now
    feeds both paths. The MRv2 Talker path carries no Live metadata (the
    Live deploy runs V1).
39. Pre-commit fixes: the handler narrows its session objects through
    `_started()` (mypy), `pybase64` replaces `base64`.
40. Under vLLM 0.32 stage processes configure logging from
    `vllm_config.logging_config`, so `VLLM_LOGGING_CONFIG_PATH`
    (`live-sessions-debug.json`, item 26) reaches the API server only; debug
    lines in model and stage code are silent.

## E2E tests

41. `tests/e2e/online_serving/test_minicpmo_4_5_live.py` (Stage 0 at 1024
    tokens, `max_sessions: 2`) and `test_personaplex_live.py` (600 frames,
    gated on `PERSONAPLEX_MODEL_PATH`): conversation, context trim, and two
    concurrent sessions with the over-limit refusal and slot reuse. Raw
    WebSocket client in `helpers/live_sessions.py`; overlays via
    `write_deploy_overlay()`. Merge-pipeline job "Omni · MiniCPM-o 4.5 Live
    Test". All 6 pass on the rebased branch (`live-pytest` recipe, Python 3.12
    for `vllm-omni[dev]`); they failed on the mis-merge in 38 before the fix.
