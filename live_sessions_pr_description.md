<!-- markdownlint-disable -->
Adds a native OpenAI Live (`/v1/live/sessions`) WebSocket endpoint and serves MiniCPM-o 4.5 and PersonaPlex on it as full-duplex models.

Implements RFC #8085 for native-VAD models only

Since `/v1/live/sessions` has no concept of turns, it provides a clean abstraction over the conversation model of native-VAD models like PersonaPlex and MiniCPM-o, where the model itself is expected to choose for itself when to speak without necessarily adhering to any well-defined turn boundaries.

## Contributor intent (required human answers)

### 1. What outcome do you want to achieve?

Support OpenAI-compliant `/v1/live/sessions` for serving native fullduplex models.

### 2. What is the highest-leverage variable for that outcome?

An OpenAI-compliant `/v1/live/sessions`, which is completely absent in upstream right now.

### 3. How will this work move that variable?

This work adds `/v1/live/sessions`. Additionally, it does so via an architecture that relies on existing vLLM and vLLM-Omni capabilities and requiring almost 0 existing server code changes.

### 4. Whose customer or user needs does this address, and what are those needs?

Anyone who wishes to interface with a vLLM-Omni native VAD model via `/v1/live/sessions`

## Purpose

**Before:** native full-duplex speech models are reachable only through vLLM-Omni's own `/v1/realtime?duplex=1` stack. OpenAI Live clients, such as LiveKit Agents' `GPTLiveModel` or the OpenAI SDK's `openai.types.live` events, cannot connect to them.

**After:** a deployment whose deploy config has a `live_session_config` block serves `/v1/live/sessions`, which speaks the OpenAI Live protocol. Each session drives one resumable engine request: audio arrives as `session.input_audio.append`, is cut into the model's native units and sent as streaming updates, and the model's audio (and MiniCPM-o's transcript) streams back as `session.output_audio.delta` / `session.output_transcript.delta`. The model decides when to listen and when to speak; there is no server-side VAD or turn detection.

- **Shared stack** (`vllm_omni/entrypoints/openai/live/`):
  - the protocol layer, built on the `openai.types.live` models;
  - the session handler;
  - a resumable request driver;
  - audio format conversion (PCM16 at 16/24 kHz, G.711 μ-law and A-law);
  - a `LiveSessionProcessor` interface that each model implements.
- **Configuration:**
  - a pipeline's `live_session_config` names its processor and an optional per-stage logits processor;
  - the deploy config's `live_session_config` enables the endpoint. All three of its keys are required: `session_lifetime_s`, `max_sessions` (over the cap, `session.start` gets `session_limit_reached` and the socket closes with 1013) and `max_input_backlog_s`.
- **Context limit:** on `context_length_exceeded` the session drops its oldest units so that what remains fills half of the context left after the prefix, then resubmits. MiniCPM-o carries the dropped units' text forward as "previous" context. PersonaPlex restarts from the newest frame.
- **Server-side session errors:** a session ends with the generic `internal_server_error` and close 1013 when more than `max_input_backlog_s` of input audio is queued behind the model, or when its stage 0 request was preempted and lost its context. The client may start a new session. The cause isn't sent to the client; the server logs it and counts it in `live_errors{reason}` (`backlog`, `preempted`). Both models detect a preemption recompute and report it from stage 0. Before this, MiniCPM-o carried on with the lost units as placeholders, and PersonaPlex raised inside the forward pass, failing every session on the engine.
- **Metrics:** `vllm_omni:live_active_sessions`, `live_rejected_sessions`, `live_audio_seconds{direction}`, `live_context_trims` and `live_errors{reason}`.
- **MiniCPM-o 4.5:**
  - a processor following the official duplex demo: 1 s units, streaming mel features, capped previous text;
  - a stage-0 logits processor that enforces the model's listen/speak grammar;
  - Live-aware hand-off from the text stage to TTS;
  - its overlay admits 8 sessions with a 16,384-token stage 0 context, about 18 GiB of stage 0 KV cache, sized for one 96 GB GPU.
- **PersonaPlex:** a processor that sends one 80 ms frame per streaming update. Its prompting helpers moved to `personaplex/prompting.py`, shared with the existing duplex path, and its Mimi codec pools are sized by `max_sessions`.
- **Deploy overlays:** `vllm_omni/deploy/minicpmo_4_5_live.yaml` and `personaplex_live.yaml`.
- **Unchanged:** existing chat, `/v1/realtime` and duplex deployments behave as before. The Live logits processor and session sizing apply only when the deploy config enables Live sessions, and the AR model runner is unchanged.
- **Shared code touched outside the API server and model pipelines:**
  - Live config plumbing in `config/stage_config.py` and `config/omni_config.py`;
  - the new `config/live_session.py` and `metrics/live.py`;
  - `openai>=3.18.0` in `requirements/common.txt`, for `openai.types.live`;
  - a PersonaPlex Live step in `.buildkite/cuda/test-merge.yml`, with its entry in `ci_source_file_dependencies.yml`.

  No engine, orchestrator, scheduler or model-runner code changes.

**Docs:** `docs/serving/live_sessions_api.md`. **Example:** `examples/online_serving/live_sessions/livekit/agent.py`, a LiveKit Agents voice agent on the endpoint.

**Not supported by native-VAD models**, and therefore narrower than the RFC's event list:
- `session.instructions.append`, `response.item.create` and `response.create`, since these models have no mid-stream text slot and their output is clocked by input audio rather than requested;
- tools and function calling;
- `store`;
- OpenAI built-in voice names;
- input transcripts.

The events, tools and `store` are rejected with `unsupported_*` errors, and each session start lists them in an `info` event. A built-in voice name is rejected with `invalid_value`. Input transcripts are simply never sent.

## Known limitation: Live sessions can deadlock on stage 0's KV cache

When stage 0 runs out of KV cache, Live sessions mostly stall rather than being preempted. The backlog limit is what ends them, which is why `max_sessions` and the stage 0 context have to be sized together.

- **What happens:** the stalled sessions wait until `max_input_backlog_s` closes them; the server then recovers.
- **Why:** between units, a Live session's resumable request waits while keeping its KV blocks, and vLLM 0.32 only preempts to make room for a *running* request. With the cache full, every Live request is waiting for one more block, so nothing runs and nothing is preempted. Scheduler logging on GPU showed exactly this: KV usage 1.0, no running requests, both Live requests waiting with `num_preemptions=0`.
- **When:** stage 0's KV cache holds less than `max_sessions` × stage 0 `max_model_len`, or other traffic on stage 0 takes the blocks the Live sessions need. The bundled MiniCPM-o overlay is sized so that its 8 sessions fit on one 96 GB GPU.

Fixing this is important but out of scope here, and a separate PR should be opened for it. Options:
- a startup check that stage 0's KV cache holds `max_sessions` × `max_model_len`;
- a stall watchdog in the Live handler;
- a scheduler change letting a waiting request preempt waiting requests that hold blocks. This affects every model and likely needs an upstream vLLM discussion.

## Test Plan

**vLLM Version:** 0.32.0

**vLLM-Omni Commit:** 3b20aec03 (on 1c11eef1a)

- **L1 (CPU):** `pytest tests/entrypoints/openai/live tests/model_executor/models/minicpmo_4_5/test_live_logits_processor.py tests/model_executor/models/minicpmo_4_5/test_live_encoder.py tests/model_executor/models/personaplex/test_live_stage0.py tests/model_executor/stage_input_processors/test_minicpmo_4_5_live.py -m "core_model and cpu"`. Covers:
  - the session handler against a fake engine: session limit, protocol errors, context overflow and resubmission, the backlog limit, preemption cancellation, close;
  - context trimming and config validation, including the required keys;
  - audio conversion, and protocol events against the openai SDK's `ServerEvent` schema;
  - the MiniCPM-o logits processor and stage hand-off;
  - preemption detection in both models' stage 0.
- **L3 (1× GPU):** `pytest tests/e2e/online_serving/test_minicpmo_4_5_live.py tests/e2e/online_serving/test_personaplex_live.py`. Each model gets four raw-WebSocket tests:
  - **conversation:** spoken replies, plus transcripts for MiniCPM-o;
  - **context trim:** a 1024-token (MiniCPM-o) or 600-token (PersonaPlex) context overflowed in about 100 s, then a reply after the trim;
  - **concurrent sessions:** two sessions at once, and a third rejected with `session_limit_reached` / 1013;
  - **KV exhaustion:** a stage 0 KV cache of one full context, with two growing sessions. A stalled session is ended with `internal_server_error` / 1013 rather than hanging, `live_errors{reason="backlog"}` rises, and a new session afterwards gets a spoken reply.

  CI runs both files in the merge pipeline ("Omni · MiniCPM-o 4.5 Live Test" and the new "Omni · PersonaPlex Live Test"). The PersonaPlex test loads `nvidia/personaplex-7b-v1` from Hugging Face, which is gated. **Maintainers: please confirm that the CI token (`VLLM_CI_HF_TOKEN`) has accepted that checkpoint's gate before merging,** or the PersonaPlex step fails with an authorization error.
- **Manual:** LiveKit Agents `GPTLiveModel` (livekit-agents 1.8.3) through a local LiveKit server and the `agent-starter-react` frontend, talking to both models live.

## Test Result

- **L1:** 203 passed. The full `tests/entrypoints` CPU suite passes (2,723 tests), including `test_api_server_guards.py` with `/v1/live/sessions` added to its route manifest.
- **L3:** all 8 tests passed on 1× RTX PRO 6000 Blackwell (96 GB) in 15 min 27 s, with PersonaPlex loaded from Hugging Face. In the KV-exhaustion tests, the backlog limit ended the stalled sessions (two for MiniCPM-o, one for PersonaPlex) about 10 s into the stall, and a new session then got a spoken reply.
- **Backlog headroom:** `max_input_backlog_s` is 10 s in both overlays. Outside the KV-exhaustion tests, the largest backlog any session reached was 21 PersonaPlex frames (about 1.7 s) at session start. No MiniCPM-o session passed the 3-unit (3 s) warning threshold.
- **Manual:** several calls with each model; the agent connected, the models replied in speech, and the calls closed cleanly with no server errors.
- **Pre-commit:** clean on all changed files, except 14 mypy errors in `vllm_omni/engine/messages.py` that were already there before this change.
