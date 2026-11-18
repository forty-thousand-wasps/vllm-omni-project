# OpenAI Live sessions (`/v1/live/sessions`) for native full-duplex models

The whole change in PR #9220 (https://github.com/vllm-project/vllm-omni/pull/9220), which implements RFC #8085 (https://github.com/vllm-project/vllm-omni/issues/8085) for native-VAD models.

- **Branch:** `feat/live-sessions` on the fork (`git@github.com:vraiti/vllm-omni.git`).
- **Commits:** `abbd3d910` (the feature) and `3b20aec03` (the response to amy-why-3459's review), on `1c11eef1a`. Signed off; squashed at merge.
- **State:** both commits pushed and the PR description updated (2026-10-29). You post the review replies yourself.

## 1. Motivation

From the RFC:

- `/v1/live/sessions` gives vLLM-Omni an OpenAI-compliant full-duplex WebSocket API. It has no concept of turns, so it fits native-VAD models, which decide for themselves when to listen and when to speak.
- The design is transparent to the orchestrator and the vLLM engine. It reuses existing vLLM and vLLM-Omni machinery for engine-side work and request state. The stack lives in the API server handler plus a handful of Live-specific functions per model.
- It is also a proof of concept of a simpler full-duplex architecture than the OmniDuplexSessionHandler stack (which is about 25k lines including its model adapters). The target was under 5k lines with no engine or orchestrator changes and a small maintenance burden per model. The PR adds about 3,700 lines outside tests, docs and examples, and changes no engine, orchestrator, scheduler or model-runner code.
- This architecture does not reuse the OmniDuplexSessionHandler stack, except for the pieces listed under "Shared code" in section 3.

## 2. Scope

**In scope:** native-VAD models only.

| Model | Unit per streaming update |
|---|---|
| MiniCPM-o 4.5 | the official streaming chunk: 1035 ms first, then 1000 ms, with look-ahead |
| PersonaPlex | one 80 ms frame |

**Not on this endpoint:**
- Turn-based (external-VAD) models such as Qwen3-Omni and Nemotron VoiceChat are served by `/v1/realtime`. The RFC's external-VAD machinery therefore isn't built: the SileroVAD service, the Whisper ASR service, the 1x output pacer, ASR timestamps, barge-in truncation, the generic ChatML processor and tool calling.
- AURA Omni is left for later.

**Non-goals** (from the RFC; rejected or ignored by the server):
- GPT-Live-1-dependent features: `session.thinking.append`, `session.commentary.append`, `session.delegation.client`.
- Persistent and multi-session management: `session.store`, and the `fork`, `content` and `attach` endpoints.
- Extra transports: WebRTC and SIP.
- vLLM-Omni full-duplex extensions with no OpenAI counterpart: VAD hints (`force_listen`, `is_speech`) and image or video input. A separate RFC should propose spec-respecting extensions for these.

## 3. Blast radius

Every file area the change touches.

**API server**
- `entrypoints/openai/api_server.py`: the `/v1/live/sessions` WebSocket route, and `init_live_app_state` at startup.
- New package `entrypoints/openai/live/`:
  - `handler.py`: the session handler;
  - `driver.py`: the resumable request driver;
  - `processor.py`: the `LiveSessionProcessor` base, including `trim_context` and the two sizing hooks (units per second, prefill bound);
  - `protocol.py`: events and errors, built on `openai.types.live`;
  - `session.py`: session state and history items;
  - `audio.py`: format conversion;
  - `serving.py`: app-state setup, the two startup checks (one API server process, context sizing), and the WebSocket entry point.

**Configuration**
- New `config/live_session.py`: `LiveSessionConfig`, `LiveSessionDeployConfig` and `LIVE_PREEMPTED_KEY`.
- `config/stage_config.py`: `live_session_config` on `PipelineConfig` and `DeployConfig`; `live_session_capacity`, `streaming_session_capacity` and `add_live_session_logits_processor`.
- `config/omni_config.py`: the production stage-config path calls the same two helpers.
- Deploy overlays `deploy/minicpmo_4_5_live.yaml` and `deploy/personaplex_live.yaml` (the latter raises stage 0's `max_model_len` and, through `hf_overrides`, `max_position_embeddings`; section 9).

**Metrics:** new `metrics/live.py`; the names in `metrics/definitions.py`.

**MiniCPM-o 4.5** (`model_executor/models/minicpmo_4_5/`)
- New `live/processor.py`, `live/logits_processor.py` and `live/encoder.py`.
- `minicpmo_4_5_omni.py`: the Live branch in `preprocess`, and the `live_preempted` output row in `forward`.
- `minicpmo_4_5_omni_llm.py`: the `live_duplex_unit` processor kwarg.
- `minicpmo_4_5_omni_tts.py`: the Talker's Live condition branch, per-unit budget, Live output metadata and sampler-cache reset.
- `pipeline.py`: `live_session_config`.
- `model_executor/stage_input_processors/minicpmo_4_5_omni.py`: the Live branch of `llm2tts`, and the per-unit flush in `tts2code2wav_async_chunk`.

**PersonaPlex** (`model_executor/models/personaplex/`)
- New `live/processor.py` and `live/stage0.py`.
- New `prompting.py`: helpers moved out of `duplex/`, which now imports them from there (`duplex/config.py`, `duplex/policy.py`, `duplex/stage0.py`).
- `personaplex_talker.py`: Live branches in `preprocess`, the batch encode, `make_omni_output` and `on_requests_finished`.
- `pipeline.py`: `live_session_config`, and stage 0 as a final text output (checked on GPU against the other PersonaPlex deployments, section 12).

**Dependencies:** `openai>=3.18.0` in `requirements/common.txt`, for `openai.types.live`.

**Docs and example:** `docs/serving/live_sessions_api.md` and its `docs/.nav.yml` entry; `examples/online_serving/live_sessions/livekit/agent.py`.

**Benchmark:** a `/v1/live/sessions` client for the existing full-duplex benchmark, so that one tool measures RTF and time to first audio on `/v1/realtime` duplex and on `/v1/live/sessions`, and nothing else (section 12). Where it plugs in is OQ1.

**Tests and CI**
- L1: `tests/entrypoints/openai/live/` (audio, config, handler, protocol, trim), `tests/model_executor/models/minicpmo_4_5/test_live_{encoder,logits_processor}.py`, `tests/model_executor/models/personaplex/test_live_stage0.py`, `tests/model_executor/stage_input_processors/test_minicpmo_4_5_live.py`; `/v1/live/sessions` added to `tests/entrypoints/openai_api/test_api_server_guards.py`.
- E2E: `tests/e2e/online_serving/test_{minicpmo_4_5,personaplex}_live.py`, `helpers/live_sessions.py`, and `write_deploy_overlay` in the shared `tests/helpers/stage_config.py`.
- CI: two merge-pipeline steps in `.buildkite/cuda/test-merge.yml` and their entries in `.buildkite/common/ci_source_file_dependencies.yml`.

**Shared code** reused from elsewhere, as approved: the Thinker's `get_audio_embedding_streaming`, the checkpoint's streaming mel processor, the scheduler's `replace_streaming_prompt`, PersonaPlex's model pipeline code (everything outside `personaplex/duplex/`), and `model_executor/common/duplex/pcm_buffer.py`.

**Not touched:** engine, orchestrator, scheduler and model-runner code. Existing chat, `/v1/realtime` and duplex deployments behave as before; for PersonaPlex this is checked on GPU (section 12).

## 4. Architecture

```text
API server
  WS /v1/live/sessions
    LiveSessionHandler
      client events -> session config, unsupported-feature gate, history
      audio in -> processor.push_audio -> processor.next_unit -> pending units
      pending units -> ResumableRequestDriver (one resumable request per session)
      engine outputs -> transcript deltas (MiniCPM-o), audio deltas, usage
                              |
                              v
       AsyncOmni.generate(StreamingInput generator, resumable=True)
                              |
                              v
Orchestrator (unmodified)
  Stage 0 ----------------> Stage 1 ---------------> Stage 2
   ^ Live logits processor (MiniCPM-o), installed from
     PipelineConfig.live_session_config.logits_processor on Live deployments
```

- **One resumable request per session.** The first prompt is `add_request(resumable=True)`. Each later unit is a streaming update.
- **The processor drives unit timing.** The handler pushes every input chunk to the model's processor and asks it for ready units. The base class cuts fixed `unit_ms` windows; MiniCPM-o overrides that with its official chunking.
- **History.** The API server keeps the conversation history. When it has to change (a context trim), the request is cancelled, the processor applies the change, and the result is resubmitted as a new request.
- **Model inputs** go in each update's `model_intermediate_buffer`, not `multi_modal_data`. The multimodal path caches inputs by content hash, so identical units, such as silence, would collide. The exception is MiniCPM-o's reference voice, sent once through `multi_modal_data` in the first prompt.

## 5. API surface

**Client events accepted**
- `session.start`.
- `session.update`: only `delegation` is accepted, and it changes nothing. Every other non-null field is `unsupported_session_config`. The reply is `session.updated`.
- `session.input_audio.append`.
- `session.input_audio.mute` / `unmute`: muted input is fed to the model as silence.
- `session.close`.

**Ignored without a reply:** `session.commentary.append`, which LiveKit's `GPTLiveModel` sends for `generate_reply()`.

**Rejected:**
- `unsupported_event`: `session.instructions.append`, `session.thinking.append`, `response.item.create` and `response.create`. These models have no mid-stream text slot, and their output is clocked by input audio rather than requested.
- `unsupported_session_config`: `client`, `delegation.client`, `delegation.responses.tools`, any other non-null `delegation.responses` field, and `store`.
- A `delegation` of type `responses` that delegates nothing is accepted and echoed, because LiveKit's `GPTLiveModel` sends `{type: "responses", responses: {model: ...}}` on every `session.start`. Its `model` must be the served model.
- An OpenAI built-in voice name gets `invalid_value`.
- Each `session.start` lists the unsupported events and session configs in an `info` event. A pipeline can add its own through `unsupported_features`; neither model does.

**Server events emitted:** `session.started`, `session.updated`, `session.input_audio.muted` / `unmuted`, `session.output_audio.delta`, `session.output_transcript.delta` (MiniCPM-o only, with the unit's `start_ms` / `end_ms`), `session.usage.updated` (every 60 s of input and at close), `session.closed`, `error` and `info`.

**Never emitted:** `session.input_transcript.delta` (no input transcription), `session.instructions.appended`, `session.delegation.created` and `response.event`.

**Audio formats:** PCM16 at 16 kHz and 24 kHz, G.711 μ-law and A-law, resampled to the model's rates.

**Errors and close codes**

| Situation | `error` | Close |
|---|---|---|
| `max_sessions` reached at `session.start` | `server_error` / `session_limit_reached`, with the `client_event_id`; no `session.closed` | 1013 |
| Live not enabled for the model or deployment | `unsupported` | 1008 |
| Invalid `session.start`, including a persona over the processor's limit (section 9) | the validation error | 1008 |
| Backlog limit exceeded, stage 0 preempted, or a full context on a model that doesn't trim | `server_error` / `internal_server_error`, "The server had an error while processing the session." | 1013 |
| Context overflow on the session's first unit, or a trim that drops nothing | `server_error` / `internal_server_error` | 1011 |
| Session lifetime reached | `session.closed` with `expired` | 1000 |
| Client `session.close` | `session.closed` with `close_requested` | 1000 |

## 6. Configuration

**Pipeline** (`PipelineConfig.live_session_config`, a `LiveSessionConfig`)
- `live_session_processor`: the processor's dotted path, resolved lazily by the API server.
- `logits_processor`: stage id to `module:Class` of a vLLM `LogitsProcessor`. It is appended to that stage's `logits_processors` engine arg on Live deployments only, so chat deployments of the same model keep their sampling. Both stage-config paths install it.
- `unsupported_features`: `events` and `session_configs`, on top of the server-wide list.
- The RFC's `vad`, `multi_modal_data_bypass` and `audio_buffer_ms` are dropped: Live is always native VAD, and the processor drives unit timing.
- Validation: a processor is required, and `logits_processor` keys must be existing stage ids.

**Deploy** (`live_session_config`, a `LiveSessionDeployConfig`). Its presence enables `/v1/live/sessions`. Every key is required, with no defaults. A missing key, an unknown key, `null` or a non-positive value is rejected at load with a message naming the key.
- `session_lifetime_s`: wall-clock limit on a session; fills `expires_at`.
- `max_sessions`: an integer; concurrent Live sessions on the server, which runs one API server process (section 7). It also sizes model-owned per-session state: `streaming_session_capacity()` feeds every stage's `duplex_max_sessions` (the duplex value in duplex mode, else the Live `max_sessions`, else 1).
- `max_input_backlog_s`: input audio that may queue behind the model before the session is ended (section 7).
- No deploy key marks Live requests. The handler marks its own requests through request parameters wherever the pipeline needs to know.

**Bundled overlays** (both `session_mode: turn`, so the `/v1/realtime` full-duplex stack stays off)

| | `minicpmo_4_5_live.yaml` | `personaplex_live.yaml` |
|---|---|---|
| `session_lifetime_s` | 1800 | 1800 |
| `max_sessions` | 8 | 2 |
| `max_input_backlog_s` | 10 | 10 |
| Stage 0 | `max_model_len: 16384` | `max_model_len` and `hf_overrides.max_position_embeddings`: 22,500 frames (1800 s) plus the largest prefill (the largest bundled voice prompt, two silence prefills and 1,024 persona tokens; section 9) |

- **MiniCPM-o sizing**, for one 96 GB GPU: 8 sessions × 16,384 tokens × ~144 KB of KV cache per token (36 layers, 8 KV heads, head dim 128, bf16) is ~18 GiB of stage 0 KV cache. Each session also holds up to ~150 MB of streaming encoder cache that vLLM's profiling doesn't count (~1.2 GB for 8). The overlay comment gives this arithmetic and says to lower `max_sessions` or `max_model_len` on smaller GPUs.
- **PersonaPlex:** `max_sessions` sizes the stage 0 streaming Mimi encoder rows and the Code2Wav decoders (one Mimi instance each). Each session's stage 0 KV stays at about one 3,000-frame window, since the Helium layers slide (section 9).
- There is no startup check that the sessions fit in stage 0's KV cache (section 10). There is one that a non-trimming model's `max_model_len` covers a session's length (section 7).

## 7. Session handler

**Admission**
- `SessionSlots` counts sessions on the API server. A slot is taken before `session.start` is validated and released when the session ends, however it ends. A refused start increments `live_rejected_sessions` and logs a warning.
- **One API server process only.** `SessionSlots` is per process, so with `--api-server-count N` the server would admit N × `max_sessions` and break the stage 0 sizing. When Live is enabled (the deploy has a `live_session_config`) and N ≠ 1, `init_live_app_state` refuses to start the server, with a message naming N. The docs say that Live with several API server processes is unsupported and its behaviour undefined. This replaces the per-session `multi_api_live_unsupported` refusal in `serve_live_session`, which let the server start and then failed every session.
- Processor assets (MiniCPM-o's remote-code processor and reference audio) preload in a thread at startup. Loading them in the first `session.start` blocked the event loop for about 3.6 s.

**Request driver** (`ResumableRequestDriver`)
- `start`, `append`, `abort` and `close`. Each request gets an epoch, so a stale request's outputs and errors are ignored.
- The scheduler drops the last sampled token on each streaming update, so the processor re-feeds it.
- vLLM's `_update_request_as_session` replaces the sampling params but not `Request.max_tokens`, so the first segment's `max_tokens` holds for the request's life. The first segment therefore uses the per-unit budget.
- The output loop stops as soon as its task is no longer the driver's active one. Without that, aborting from the request's own output callback didn't stop the request. Its `finally` closes the generator, which aborts the engine request.

**Units and outputs**
- Audio is converted to the model's rate and pushed to the processor. Ready units queue in `_pending` and go to the driver one segment at a time.
- Stage 0's `finish_reason` marks the end of a segment.
- Audio deltas stream as the later stages produce them; there is no output pacing.
- A warning is logged when the backlog grows past 3 units.

**Backlog limit**
- After queuing new units, the backlog is the total duration of `_pending`.
- If it exceeds `max_input_backlog_s`, the session ends as a server-side error that doesn't expose the cause (`_abort`): the generic `internal_server_error`, close 1013, one concise error log line with the queued seconds and the limit, and `live_errors{reason="backlog"}`.
- Units are never dropped, because that would silently cut audio out of the conversation.
- 10 s in both overlays. In the E2E runs, the largest healthy backlog was 21 PersonaPlex frames (~1.7 s) at session start, and no MiniCPM-o session passed 3 units.

**Preemption**
- When stage 0 runs out of KV cache, vLLM may preempt a running request. It frees the request's blocks and later recomputes it from the start of the prompt.
- A Live request's audio isn't in its prompt. Each update's `model_intermediate_buffer` replaces the last one, and the model turns that update's audio into embeddings that it uses and then drops. A recompute replays placeholder tokens with nothing behind them.
- vLLM records the preemption only in scheduler stats, so the handler can't see it. Each model detects the recompute in its own stage 0 code (sections 8 and 9) and sets a per-request `live_preempted` row in stage 0's `multimodal_outputs`.
  - The key is not declared in `omni_client_multimodal_output_keys`. For a stage 0 that isn't a connector sender, the client payload is the full payload, so the row already reaches the handler. Declaring keys would cut the client payload down to just those keys.
  - Path: the runner's per-request payload, then `OmniEngineCoreOutput.multimodal_output`, then the output processor's accumulated metadata, then the stage 0 completion's `multimodal_output`.
- The handler checks `processor.is_preempted(output)` on every stage 0 output, before forwarding anything from it. If it is set, the session ends like the backlog case, with `live_errors{reason="preempted"}`.
- Raising in the model isn't used: an exception in the forward pass fails the engine step for every session on the engine, and the handler never sees it.
- Keeping a preempted session going is out of scope: it would need every unit's input and the model's own outputs kept for the whole session, plus replaying both stateful encoders from the start.
- Up to one recomputed step's text or audio may reach the client before the flag arrives.

**Context limit**
- No predictive trim. The engine raises `context_length_exceeded` (from `OmniARScheduler._streaming_update_overflows`) when a streaming update doesn't fit.
- **Trimming is per model.** A model trims only if its processor has a `trim_context`; the base class sets `trim_context = None`, so a new model doesn't trim unless it opts in. The current generic trim stays on the base class as a helper a processor can use: drop the oldest units so that what remains fills 50% of the context left after the prefix, always keeping the prefix. After a trim the handler resubmits (`render_resume`) and re-sends the unit that overflowed.
  - If assigning `None` over a method fails mypy, the base class instead defines a `trim_context` that reports "no trim" (the same error path below), and processors that trim override it. A model that doesn't trim then just doesn't override it.
  - MiniCPM-o trims, since its Thinker uses full attention (`use_sliding_window: false`). PersonaPlex doesn't: it relies on its sliding window (section 9).
- **Overflow without a trim** ends the session through `_abort("context", ...)`: the generic `internal_server_error`, close 1013 (the client may start a new session), the cause logged, and `live_errors{reason="context"}`.
- **The two other context failures**, overflow on the session's first unit and a trim that drops nothing, stay on `_fail`, with reason `context`: the generic `internal_server_error`, close 1011, and the same metric. A new session with the same `session.start` would fail the same way, so 1013's "try again" would mislead. Their message gives the session's prefix size and stage 0's `max_model_len`.
- **History without a trim.** `render_resume` runs only after a trim, so for a processor with no `trim_context` nothing reads a unit's audio once it's sent. The handler keeps those units in `state.history` without their audio. For PersonaPlex that is 7.5 KB a frame, about 170 MB in a 30-minute session otherwise.
- **Startup sizing check for models that don't trim.** `init_live_app_state` refuses to start, with a message naming both values, when stage 0's `max_model_len` is less than `session_lifetime_s` worth of units plus the processor's largest possible prefill. The processor reports both:
  - units per second of input (PersonaPlex: 12.5);
  - an upper bound on its prefill, built from its persona limit (section 9).
  A client sending faster than real time can still exceed `max_model_len` before the session expires; it then ends as above.
- Each overflow logs two ERROR lines from upstream code (`omni_ar_scheduler`, `async_omni_base`) even though the session recovers. This is left as is.

**Lifecycle:** session expiry at `session_lifetime_s`; usage events; cleanup on close releases the slot and aborts the request.

**Ending a session on an error.** Two handler methods end a session on a server-side error. Both send the generic `internal_server_error`, count `live_errors{reason}` and log the cause.
- `_abort(reason, detail)` closes 1013: a new session may work (the backlog limit, preemption, a full context without a trim).
- `_fail(message, reason="internal", exc=None)` closes 1011: the server hit something unexpected, or a retry would fail the same way. It logs one error line, `Live session <id> failed (<reason>): <message>`, with the traceback when `exc` is passed.
  - Every caller passes a message. Before, `_fail` took only an exception and logged nothing.
  - The three callers that logged before calling it (the session's `run` loop, a failed background task, and `_on_engine_error`) drop their own log line, so each failure logs once.
  - An `error` on an engine output, unlogged before, now logs through `_fail`.

## 8. MiniCPM-o 4.5

Follows the official MiniCPM-o-Demo (`DuplexCapability`), not the checkpoint, wherever they differ.

**Prompt**
- The first prompt is the system prompt and reference voice alone: `<|im_start|>system\n{instructions}\n<|audio_start|>[HT_ref_audio.wav][\n\nprevious: {text}]<|audio_end|><|im_end|>`.
  - The reference clip comes from the checkpoint's `assets/`, resolved through a snapshot download when the model path is a repo id.
  - It is sampled with the unit budget and ends on a forced `<|listen|>` that is never fed. The logits processor returns that token before touching the session's decoded history, so it doesn't enter the repetition window, and `llm2tts` sends nothing to the Talker for a segment ending in `<|listen|>`.
  - Units follow as updates to the same resumable request, and after a trim the whole history follows as one update. They can't sit in the first prompt: on every streaming update, the orchestrator's async-chunk prewarm rebuilds each downstream placeholder from the request's first prompt and copies its `model_intermediate_buffer` (`_PREWARM_UNCOPIED_PROMPT_KEYS` in `engine/orchestrator.py` excludes only the prompt ids, multimodal data, processor kwargs and `additional_information`).
  - Stage 0 ends with the same token sequence a combined first prompt would give. The cost is one extra stage 0 round trip each time a request opens: at session start and after each trim. The orchestrator stays unchanged.
- `live_duplex_unit` drops the chat format's `<|audio_start|>` / `<|audio_end|>` around the reference audio's rows, so the prompt can write them itself with the `previous:` text in between.
- Each unit is `<unit>` followed by its audio rows, which use `<|tts_pad|>` as the placeholder token (the decoder always forbids it, so it is never fed otherwise).
- The first prompt carries a top-level `minicpmo_live` marker with the unit grammar's token ids. It isn't a buffer entry, so no engine receives it.

**Input**
- Streaming mel features in the processor, using the checkpoint's streaming mel processor in exact mode. The first unit is left-padded to 1035 ms; units are 16,560 samples, then 16,000. No samples are dropped.
- The features travel in the update's `model_intermediate_buffer` (`minicpmo_live_audio`) as raw float32 bytes, because vLLM's msgpack decoder doesn't rebuild tensors in untyped fields.
- The Thinker's `preprocess` Live branch (`live/encoder.py`) encodes each update's units once with `get_audio_embedding_streaming` and the session's own encoder cache (`LiveEncoderState`, kept in the request's buffer), and writes the embeddings over the placeholder rows wherever chunked prefill schedules them.
- Muted input is zeros through the same chunker.
- **Preemption:** a step with more placeholder rows than embeddings left (`count > available`) is a recompute. A normal step can't do that, because a segment's rows arrive in order and `cursor` tracks them. It also holds if the runner rebuilt the state. The model sets `LiveEncoderState.preempted`, logs one warning, and `live_preempted_rows` reports the flag from `forward` on every later step.

**Decode** (`MiniCPMOLiveLogitsProcessor`, stage 0, greedy)
- If the raw argmax is `<|chunk_eos|>`, take it.
- Otherwise mask `<|tts_pad|>`, the tokenizer's bad tokens and `<|chunk_eos|>`; divide the logits of the distinct tokens among the last 512 decoded tokens (special tokens included) by 1.05; apply a 1.1 length penalty to `<|turn_eos|>`; take the argmax.
- `<|listen|>` becomes `<|tts_bos|>` before the turn has ended.
- Force `<|listen|>` on the first 3 units; `<|chunk_eos|>` as the 20th token; `<|chunk_eos|>` when a non-first token would bring the unit's text to 28 characters.
- Stage 0 `repetition_penalty` 1.0. Each decision is applied as a one-hot row, and rows are skipped after their terminator, which covers an asynchronous scheduler's lookahead step.
- vLLM re-adds the request on every streaming update, so the decoded history and turn state are kept per session id (in `extra_args`) in the worker, with an LRU bound of 1024 sessions.

**History and trim**
- On a trim, the dropped units' text is carried forward as `previous:` text, capped at its newest 500 tokens (the checkpoint's `context_previous_max_tokens`). Uncapped, it grew with every trim until the prefix filled the context.

**Transcripts:** each delta's tokens are decoded as they arrive, matching the Demo's `decode(res_ids)`. Missing spaces between units are the model's own tokens.

**Talker hand-off** (`llm2tts`)
- Stateless per output: the current unit is what follows the previous stop token in `cumulative_token_ids`.
- The unit's hidden rows are found through the `latent_input_ids` / `latent_positions` ledger: the latest position-contiguous block whose inputs are the unit's fed tokens.
- A listen unit produces no speech. The unit's first token doesn't condition the Talker. A unit ending in `<|turn_eos|>` ends the turn, and a turn's first unit starts a fresh Talker context through `replace_streaming_prompt`.
- Per-unit codec budget: 26 tokens maximum; the minimum is 26 except at a turn's edges, where it is 0.

**Talker** (`minicpmo_4_5_omni_tts.py`)
- The Live condition branch builds the official condition: per spoken token, its text embedding plus its normalized projected Thinker hidden state, then `audio_bos`. It honours upstream's `_normalize` flag.
- Output metadata `minicpmo_live` / `live_turn_end` on both decode paths.
- Each Live unit's re-prefill drops the sampler's device-side caches (`if native_duplex or live_budget is not None:`), so the step counter and `min_tokens` reset per unit. That sampler only runs in the K-step decode, which is NPU-only, so CUDA wasn't affected.
- Greedy: stage 1 `temperature 0`, `top_p 1`, `top_k -1`, `min_tokens 0`.

**Code2Wav** (`tts2code2wav_async_chunk`)
- Each unit's segment end flushes its audio without retiring the request. A turn end closes the stream and starts a new codec epoch, as the official Token2Wav resets per turn.
- Fewer than 5 codec frames at a unit end wait for the next unit (native duplex's `hold_short_unit`). A one-word unit flushed alone broke Token2Wav's conformer cache and killed the stage 2 engine.

## 9. PersonaPlex

A Live-owned stage 0 path, written against the model. It doesn't use anything under `personaplex/duplex/` or `engine/duplex/`.

- **Prompting helpers** move to `personaplex/prompting.py`: frame geometry, lockstep tokens, `wrap_with_system_tags`, the voice and tokenizer loaders, `personaplex_prefill_slots`, and `personaplex_model_dir`, which resolves a repo id to its local snapshot. `duplex/` imports them, with its public names unchanged.
- **Units:** the processor cuts fixed 80 ms windows (1,920 samples at 24 kHz). Each update carries the frame's PCM in `model_intermediate_buffer["personaplex_live"]`. The first update also carries the persona and voice, and reserves the prefill rows.
- **Persona:** `instructions` and developer `input` messages form the persona; user and assistant `input` messages become a "Conversation so far" line in it. Each persona token takes one prefill frame. The processor caps the whole persona at 1,024 tokens, a processor constant, checked at `session.start` (`invalid_value`, close 1008). The limit also bounds the prefill in the startup sizing check (section 7). A persona much over 3,000 tokens would slide out of the window before the conversation started anyway. The build checks the default persona's length against the limit.
- **Voices:** `default`, or a bundled voice-prompt name with or without `.pt`, checked at `session.start`.
- **Stage 0 is a final text output**, so the API server receives each frame's sampled text token and sends it back with the next frame. The handler also relies on stage 0's outputs for segment-end pacing and the `live_preempted` row. It's set in the shared `personaplex/pipeline.py`; a GPU run checks that `/v1/realtime` duplex and chat on PersonaPlex behave as before (section 12).
- **Stage 0 per frame** (`live/stage0.py`):
  1. Mimi-encode the new frame, batched across the step's Live appends, using one shared codec with a streaming row per session slot.
  2. In frame t+1's `preprocess`, run frame t's depformer step from the stored hidden state and the fed-back text token, with Moshi teacher forcing. That gives frame t's agent codes for Code2Wav.
  3. Build frame t+1's input embedding (agent codes of t, user codebook 0 of t and 1..7 of t−1).
  - Output audio therefore trails the input by one extra frame (80 ms). No shared runner changes.
- **Codes to Code2Wav:** `make_omni_output` returns a per-request list in batch order, holding the codes of the frame that step's `preprocess` finished, or an empty tensor. The runner indexes list-valued outputs by batch position.
- **No transcripts.**
- **Context: a sliding window, no trim.** PersonaPlex matches NVIDIA's reference (`moshi/modules/transformer.py` at `3428dfd`), which keeps a 3,000-frame `RingKVCache`, attends to the last 3,000 positions, and runs a stream indefinitely; its RoPE is computed from the absolute position, so positions past 3,000 are normal. The persona and voice prompt slide out of the window after 240 s there too.
  - The processor sets `trim_context = None`, so a session never resubmits and the handler keeps no frame audio.
  - The Helium layers already pass `per_layer_sliding_window=3000`, and vLLM 0.32's `SlidingWindowManager` frees blocks behind the window, so stage 0 KV stays at about one window per session.
  - The cap is the request length: `max_model_len: 3000` (`personaplex.yaml`) and `max_position_embeddings: 3000` (`configuration_helium.py`), which sizes the RoPE cache and is what vLLM's `_get_and_verify_max_len` compares `max_model_len` against (it refuses a larger one unless `VLLM_ALLOW_LONG_MAX_MODEL_LEN=1`). The Live overlay raises both for stage 0: `max_model_len`, and `hf_overrides.max_position_embeddings`, to 22,500 frames plus the largest prefill. `configuration_helium.py` and the other PersonaPlex deployments don't change.
  - Before this is kept, a GPU run checks that vLLM accepts the overrides, the KV usage, and that the audio stays good past frame 3,000 (section 12). If it doesn't hold, that's a spec turn.
  - The earlier trim, which replayed only the persona, voice and newest frame and so forgot the conversation every 240 s, goes.
- **Preemption:** a span that starts before the current frame (`local < 0`) is a recompute. It used to raise `ValueError` inside the forward pass, failing every session on the engine. Now the session's `preempted` flag is set, one warning is logged, the span gets zeros, and `make_omni_output` reports the row. A span past the prepared frame still raises, since that's a real bug.
- **Capacity:** the stage 0 Mimi rows and Code2Wav's Mimi instances come from the Live `max_sessions`, through `duplex_max_sessions`. Sizing from `max_num_seqs` would load up to ten copies.

## 10. Known limitation: stage 0 KV deadlock

When stage 0 runs out of KV cache, Live sessions mostly stall rather than being preempted.

- **Why:** between units, a Live request waits while keeping its KV blocks, and vLLM 0.32 only preempts to make room for a running request. With the cache full, every Live request waits for one more block, so nothing runs and nothing is preempted.
- **Evidence** (2026-10-26, MiniCPM-o, temporary scheduler logging since removed): for 121 one-second samples, KV usage 1.000, 0 free blocks, `running=0`, both Live requests `WAITING` with `num_preemptions=0` and one unit (13 tokens) uncomputed.
- **What ends it:** the backlog limit closes the stalled sessions, stage 0's cache frees, and the server recovers.
- **When:** stage 0's KV cache holds less than `max_sessions` × stage 0 `max_model_len`, or other traffic on stage 0 takes the blocks. The MiniCPM-o overlay is sized so that 8 sessions fit on one 96 GB GPU.
- **A real preemption** needs two Live requests decoding at the same moment at a block boundary. It is rare, so the preemption handling is a safety net covered by CPU tests.
- **Out of scope; a separate PR should fix it.** Options:
  - a startup check that stage 0's KV cache holds `max_sessions` × `max_model_len`;
  - a stall watchdog in the handler;
  - a scheduler change letting a waiting request preempt waiting requests that hold blocks. This affects every model and likely needs an upstream vLLM discussion.

## 11. Metrics

`vllm_omni:live_active_sessions`, `live_rejected_sessions`, `live_audio_seconds{direction}`, `live_context_trims` and `live_errors{reason}`. The `reason` values include `backlog`, `preempted` and `context`.

## 12. Testing and CI

**L1 (CPU)**, `-m "core_model and cpu"`, 203 tests:
- the handler against a fake engine: session limit, protocol errors, context overflow and resubmission, overflow without a trim, the backlog limit, preemption cancellation, close;
- context trimming, and config validation including the required keys;
- audio conversion, and protocol events against the openai SDK's `ServerEvent` schema;
- the MiniCPM-o logits processor and stage hand-off;
- preemption detection in both models' stage 0;
- the full `tests/entrypoints` CPU suite (2,723 tests) also passes.

**L3 (1× GPU)**, raw-WebSocket tests:
- **Input pacing:** the E2E helper (`tests/e2e/online_serving/helpers/live_sessions.py`) sends input in 100 ms chunks paced at real time. Its send loop takes a `speed` parameter: the multiple of real time to send at, with `0` meaning unpaced (each chunk as soon as the socket takes it). The default is 1. The server has no pacing of its own: it sends each unit when the previous segment finishes, so a session moves as fast as its input arrives and the model keeps up.
- An unpaced test raises `max_input_backlog_s` in its test overlay above its clip's length. Otherwise the whole clip queues at once and the backlog limit ends the session.
- **conversation:** spoken replies, plus transcripts for MiniCPM-o;
- **context trim** (MiniCPM-o): a 1024-token context overflowed in about 100 s, then a reply after the trim;
- **long session** (PersonaPlex, unpaced): replaces the old 600-frame trim test. One session streams past frame 3,000, and replies keep coming after the window starts sliding. Its clip has enough silence after each spoken prompt for the reply to come out before the next one. The no-trim overflow (`_abort`, 1013, `live_errors{reason="context"}`) is covered by the L1 handler tests only;
- **concurrent sessions:** two at once, a third refused with `session_limit_reached` / 1013, and a closed session's slot reused;
- **KV exhaustion:** stage 0's KV cache holds one full context (MiniCPM-o 65 blocks of 16; PersonaPlex 40 blocks of 16, since vLLM keeps one or two blocks back), with `max_input_backlog_s: 10`. Two sessions talk until the server ends one or 120 s pass. A stalled session is ended with `internal_server_error` / 1013, `live_errors{reason="backlog"}` rises, no session gets another error, and a new session afterwards gets a spoken reply.
- Result: all 8 passed on 1× RTX PRO 6000 Blackwell (96 GB) in 15 min 27 s, with PersonaPlex loaded from Hugging Face.

**CI**
- Merge pipeline: "Omni · MiniCPM-o 4.5 Live Test" and "Omni · PersonaPlex Live Test" (`h100_1`, 30-minute timeout), with entries in `ci_source_file_dependencies.yml` (a new `&personaplex` anchor and `omni_personaplex_live_function`).
- The PersonaPlex test loads `PERSONAPLEX_MODEL_PATH` when set, else `nvidia/personaplex-7b-v1` from Hugging Face, which is gated. The PR description asks maintainers to confirm that `VLLM_CI_HF_TOKEN` has accepted the gate before merging.
- Out of scope: `test_personaplex_duplex.py` keeps its local-checkout skip; adding it to weekly CI is a separate PR.

**Manual:** LiveKit Agents `GPTLiveModel` (livekit-agents 1.8.3) through a local LiveKit server and the `agent-starter-react` frontend, talking to both models.

**Pre-commit:** clean on the changed files, apart from 14 mypy errors already in `engine/messages.py`.

**Before merge, on GPU** (each run needs your approval):
- **Other PersonaPlex deployments:** `/v1/realtime` duplex and chat on PersonaPlex behave as before with stage 0 as a final output: no extra text outputs reach their clients, and the audio is the same.
- **PersonaPlex past the window:** vLLM accepts the Live overlay's `max_model_len` and `max_position_embeddings`, stage 0 KV stays near one window per session, and replies stay good past frame 3,000.
- **RFC performance targets:** real-time factor and time to first audio within 5% of OmniDuplexSessionHandler on MiniCPM-o, and RTF ≤ 1.0 on PersonaPlex. The results go in the PR description.
  - **The tool:** the full-duplex benchmark, committed in this PR with a `/v1/live/sessions` client next to its `/v1/realtime` duplex one (see OQ1). It measures only these two endpoints. It isn't wired into CI.
  - **Same inputs, same definitions on both endpoints.** Both runs get the same scenario audio (the E2E scenarios' clips, which the `minicpmo-duplex-reference` recipe already feeds to `/v1/realtime?duplex=1`) and the checkpoint's `HT_ref_audio.wav` voice for MiniCPM-o.
  - **Time to first audio** is the existing `ttfp`: from the start of the input stream to the first audio packet.
  - **RTF:** which definition is OQ2.

## 13. Review response (PR #9220)

amy-why-3459's review, answered in `3b20aec03`:

| Comment | Resolution |
|---|---|
| `handler.py:464`, unbounded `_pending` | the backlog limit (section 7) |
| `minicpmo_4_5_live.yaml:13`, `max_sessions: null` | every deploy key required; MiniCPM-o sized at 8 sessions with a 16,384-token stage 0 (section 6) |
| `live/encoder.py:118`, placeholder embeddings after a recompute | preempted sessions cancelled, for both models (sections 7 to 9) |
| `minicpmo_4_5_omni_tts.py:1207`, codec caches across Live units | cleanup also for Live units (section 8) |
| Review summary: unused `segment_finished` | reverted in `engine/messages.py`, `engine/orchestrator.py`, `entrypoints/omni_base.py` and `outputs/__init__.py` |

Draft replies are in `/home/vraiti/omni/live-sessions/pr-review-replies.md`; you post them.
