# OpenAI Live (`/v1/live/sessions`), native VAD only: implementation plan

Status: draft for review. Branch: `feat/live-sessions-v2` (upstream `0e6791e9c`).

## Scope

`/v1/live/sessions` serves only **native-VAD** models: the model decides when to
listen and speak, unit by unit. Turn-based (external-VAD) models are served by
`/v1/realtime`, which now supports them fully, so this endpoint drops the
external VAD service, the output pacer, ASR-based token timestamps, barge-in
truncation, tool delegation, the turn-based processors and input transcription
(Whisper).

Models:

| Model | Status | Unit |
|---|---|---|
| MiniCPM-o 4.5 | in scope | official streaming chunks (1035 ms, then 1000 ms, with look-ahead) |
| PersonaPlex | in scope (Live-owned stage-0 path, see 6) | 80 ms frame |
| Nemotron VoiceChat | deferred | 80 ms frame |
| AURA Omni | deferred | to check |

Supersedes the external-VAD parts of `spec/live_sessions_plan.md`;
`spec/minicpmo_live_talker_plan.md` and `spec/minicpmo_live_parity_plan.md`
carry over as written, against the new upstream code. A fresh revisions doc
(`spec/live_sessions_native_revisions.md`) records divergences from this plan.

Rules that carry over:

- No code from the OmniDuplexSessionHandler stack is reused, except what the
  user approved: the thinker's `get_audio_embedding_streaming`, the checkpoint's
  streaming mel processor, the generic scheduler `replace_streaming_prompt`,
  PersonaPlex's model pipeline code (everything outside `personaplex/duplex/`)
  and `model_executor/common/duplex/pcm_buffer.py`. Sharing anything else, or
  refactoring that stack into shared code, is proposed to the user first.
- Thinker and Talker decode greedily (temperature 0) on Live sessions.
- Testing is E2E on AWS (run-remote recipes), no local test harnesses.

## Source material

The first implementation lives on `AUTOCOMMIT/feat/live-sessions` (base
`6daf5b30f`). Files with no upstream churn since then port as is; the rest are
re-applied by hand:

| Re-apply by hand (upstream changed) | Port as is (new files / no churn) |
|---|---|
| `minicpmo_4_5_omni_tts.py` (Talker; #8222, #7968) | `entrypoints/openai/live/{protocol,session,audio,driver,serving}.py` (trimmed, below) |
| `stage_input_processors/minicpmo_4_5_omni.py` (`llm2tts`, `tts2code2wav_async_chunk`) | `metrics/live.py` (trimmed) |
| `config/stage_config.py` (`live_session_config`) | `engine/messages.py`, `outputs/__init__.py`, `entrypoints/omni_base.py` (`segment_finished`) |
| `engine/orchestrator.py` (`segment_finished` forwarding) | `minicpmo_4_5_omni_llm.py` (`live_duplex_unit`; may become unnecessary, see 3) |
| `entrypoints/openai/api_server.py` (route, app state) | `examples/online_serving/live_sessions/livekit/` (headless, agent, piper) |
| `minicpmo_4_5/pipeline.py`, `personaplex/pipeline.py` | `docs/serving/live_sessions_api.md` (rewritten for native only) |

Not ported: `live_vad_service.py`, `vad_client.py`, `asr_client.py`,
`pacer.py`, `timestamps.py`, `tools.py`, the Qwen3-Omni Live processor and
overlay, the `engine/duplex/vad.py` message fix, the PersonaPlex Live processor
(rewritten, see 6). The `openai>=3.18` pin is kept (`openai.types.live`).

The fork branch is the source material (`fork/feat/live-sessions`, the same
commit as `AUTOCOMMIT/feat/live-sessions`).

## 1. Config

- `LiveSessionConfig` (pipeline): `live_session_processor` (dotted path),
  `logits_processor: dict[int, str]`, `unsupported_features`. Drop `vad` (always
  native), `multi_modal_data_bypass`, and `audio_buffer_ms` (the processor now
  drives unit timing, see 2).
- `LiveSessionDeployConfig` (deploy YAML `live_session_config`):
  `session_lifetime_s` and `max_sessions: int | None = None` (concurrent Live
  sessions per API server; `None` is unbounded). Drop the `external_vad` block
  and the ASR keys (`asr_service_url`, `asr_timeout_s`,
  `audio_transcription_interval_ms`).
- Validation in `PipelineConfig.get_validation_errors`: a pipeline with
  `live_session_config` must name a processor.

## 2. Processor protocol: the processor drives unit submission

One base class, `LiveSessionProcessor` (the old `NativeVadProcessor` folded in):

```python
class LiveSessionProcessor(ABC):
    input_sample_rate: ClassVar[int]
    output_sample_rate: ClassVar[int]

    def __init__(self, context: ProcessorContext) -> None: ...
    def resolve_voice(self, voice) -> str | None: ...
    def sampling_params_list(self, base) -> list: ...
    def stage0_sampling_params(self, state, base): ...

    def push_audio(self, samples: np.ndarray, start_ms: float) -> None: ...
    def push_silence(self, duration_ms: int) -> None: ...      # muted input
    def next_unit(self) -> NativeUnitItem | None: ...          # None: not ready

    @abstractmethod
    def render_unit(self, state, unit, *, first, last_sampled_token) -> RenderedPrompt: ...
    @abstractmethod
    def render_resume(self, state) -> RenderedPrompt: ...

    def observe_stage0_output(self, state, token_ids, output) -> None: ...
    def spoken_text(self, token_ids, text) -> str: ...
    def observe_multimodal_output(self, state, output) -> str: ...
    def trim_context(self, state, budget_tokens) -> None: ...
```

`ProcessorContext` is kept only if the processors need it (model path,
tokenizer, stage configs); otherwise the constructor takes what it uses.

- The handler pushes every input chunk to the processor and asks for ready units;
  it no longer cuts fixed `audio_buffer_ms` windows.
- The default `push_audio`/`next_unit` implement a fixed window
  (`unit_ms: ClassVar[int]`), so a frame-locked model (PersonaPlex, Nemotron at
  80 ms) only sets `unit_ms`; MiniCPM-o overrides them with the official
  streaming chunking (padding, look-ahead gating, consumed-samples bookkeeping).
- A unit carries what the processor needs to render it (audio or features,
  `start_ms`/`end_ms` for transcript timing).

## 3. MiniCPM-o 4.5

From the earlier work, re-applied on the new upstream:

1. **Prompt**: official duplex layout; system prompt
   `<|im_start|>system\n{instructions}\n<|audio_start|>[HT_ref_audio.wav]<|audio_end|><|im_end|>`
   (reference clip from the checkpoint's `assets/`, resolved with
   `hf_api().snapshot_download` when the model path is a repo id).
2. **Input parity** (`spec/minicpmo_live_parity_plan.md`): streaming mel features
   per unit in the Live processor (first unit left-padded to 1035 ms, 16560
   then 16000 samples per unit, no samples dropped); unit audio leaves
   `multi_modal_data` and travels in the streaming update's
   `model_intermediate_buffer`; a Live branch in the thinker's `preprocess`
   runs `get_audio_embedding_streaming` with the session's encoder cache kept in
   the request's buffer. Muted input is zeros through the same chunker (the
   Demo's mobile mute).
3. **Decode parity** with MiniCPM-o-Demo's `DuplexCapability` (greedy): raw
   argmax `<|chunk_eos|>` is taken as is; otherwise mask `<|tts_pad|>`, the
   tokenizer's bad tokens and `<|chunk_eos|>`, divide the logits of the distinct
   tokens among the last 512 decoded tokens (special tokens included) by 1.05,
   apply the 1.1 length penalty to `<|turn_eos|>`, take the argmax;
   listen→`<|tts_bos|>` before turn end; force `<|listen|>` on the first 3 units;
   20th token `<|chunk_eos|>`; force `<|chunk_eos|>` when a non-first token would
   bring the unit's text to 28 characters. Stage-0 `repetition_penalty = 1.0`.
   The processor's per-session history is keyed by a session id in
   `extra_args` (vLLM re-adds the request on every streaming update).
4. **Talker handoff** (`spec/minicpmo_live_talker_plan.md`): first-prompt
   top-level `minicpmo_live` marker (not a `model_intermediate_buffer` entry: the
   async-chunk prewarm copies that to Code2Wav); stateless unit extraction in
   `llm2tts` (`cumulative_token_ids` split at stop tokens, the unit's hidden rows
   located through upstream's `latent_input_ids` / `latent_positions` ledger
   rather than the latent's tail, condition `t_2..t_k`); `replace_streaming_prompt` on a turn's first unit;
   Talker Live condition branch and 26-token per-unit budget, rewritten against
   the new Talker; Talker output meta `minicpmo_live`/`live_turn_end`.
5. **Code2Wav**: Live per-unit flush in `tts2code2wav_async_chunk` without
   retiring the request; turn end closes the stream and starts a new codec epoch.
6. **Sampling**: greedy stage 0 (deploy YAML) and greedy Talker
   (`sampling_params_list`: stage-1 `temperature 0`, `top_p 1`, `top_k -1`,
   `min_tokens 0`).
7. **Deploy overlay** `minicpmo_4_5_live.yaml` (`session_mode: turn`, so the
   OmniDuplexSessionHandler stack is not activated, and `live_session_config`).
   No deploy key selects the Live path: `/v1/live/sessions` has a single
   handler, which marks its own requests through request parameters
   (`additional_information` / sampling `extra_args`) wherever the pipeline
   needs to know a request is Live.

Whether `live_duplex_unit` is still needed is decided during implementation;
mute behaviour under the official chunking follows `MiniCPM-o-Demo`.

## 4. Handler (`entrypoints/openai/live/handler.py`)

Native only, rewritten from the old native path:

- Session lifecycle and events: `session.start` / `update` / `close`,
  `input_audio.append`, `input_audio.mute` / `unmute`; every other client event
  (`response.create`, `response.item.create`, `session.instructions.append`,
  delegation) is `unsupported_event`, and `delegation` / tools and input
  transcription in the session config are `unsupported_session_config`.
- Session cap: when `max_sessions` Live sessions are active on this API server,
  a `session.start` gets a `LiveErrorEvent` (`client_event_id` = the
  `session.start` event id, `error.type: "server_error"`,
  `error.code: "session_limit_reached"`), then the WebSocket closes with 1013
  (Try Again Later). No `session.closed` is sent: its `reason` values do not
  cover a refusal and the session never started.
- Audio in → `processor.push_audio` → `next_unit` → driver append, one segment
  at a time.
- Segment out → output transcript (`spoken_text`), audio deltas, usage.
- No input transcripts.
- Context management: no predictive trim. On the engine's
  `context_length_exceeded` terminal output (`finish_reason="error"`), the
  processor trims the history to 50% of the context left after the system
  prompt and reference audio (`max_model_len` minus that prefix; the prefix is
  always kept, the newest units are kept up to the budget), the handler
  resubmits (`render_resume`) and re-sends the dropped unit.
- Session expiry and usage events as before.

## 5. Engine plumbing

- `segment_finished` on outputs (messages, orchestrator forwarding, outputs,
  `omni_base`), re-applied against the new orchestrator.
- `/v1/live/sessions` route and `init_live_app_state` in `api_server.py`.
- Verify end to end that `context_length_exceeded` reaches the API server
  (finish reason and stop reason).

## 6. PersonaPlex

The first implementation fed frames through the stage-0 duplex runtime
(`model_intermediate_buffer["duplex"]`), which is OmniDuplexSessionHandler
stack. Upstream, that runtime is now the `PersonaPlexDuplexPlugin`
(`personaplex/duplex/`) of the unified full-duplex framework.

Decision: a Live-owned stage-0 path for PersonaPlex, written against the model
as MiniCPM-o's is. It may use PersonaPlex's model pipeline code (talker,
temporal/depformer, Mimi, code2wav, embeddings, `pipeline.py`) and
`model_executor/common/duplex/pcm_buffer.py`, but nothing under
`personaplex/duplex/` or `engine/duplex/`. The processor is a fixed 80 ms
window (`unit_ms = 80`).

- Shared prompt helpers (voice-prompt loader, tokenizer loader, prefill-slot
  count, silence / sine / zero-text tokens, `wrap_with_system_tags`) move from
  `personaplex/duplex/` to a new `personaplex/prompting.py`; `duplex/` imports
  them from there.
- No shared runner changes. Stage 0 becomes a final (text) output, so the API
  server receives each frame's sampled text token and sends it back in the next
  frame's update. The talker's `preprocess` for frame t+1 runs the depformer for
  frame t (from the stored hidden state and the fed-back token), emits frame t's
  agent codes to Code2Wav and builds frame t+1's input. Output audio trails the
  input by one extra frame (80 ms).
- Each update carries the frame's raw PCM (1920 samples at 24 kHz); the talker
  Mimi-encodes it.
- No PersonaPlex output transcripts for now.
- Talker and Code2Wav session capacity on a Live deployment comes from the Live
  config, not `duplex_session.max_sessions`.

## 7. Verification (E2E, run-remote)

Recipes (run-remote `recipes/vllm-omni/live-sessions/`), updated for this branch:

- `minicpmo-e2e-test`: LiveKit headless scenarios (`--no-tools --voice default`,
  AEC warmup off), greedy.
- `minicpmo-replay-test`: identical-input replay
  (`vllm-omni-aux/clients/minicpmo-compare/<scenario>_24k.wav`).
- `minicpmo-duplex-reference` (greedy `/v1/realtime`) and a new
  `minicpmo-demo-reference` (MiniCPM-o-Demo, greedy): reference outputs.
- Drop `qwen-e2e-test` (Qwen3-Omni moves to `/v1/realtime`).

Smoke criteria per scenario: no stage errors; every speaking unit's audio
arrives, none for listen units; turn tails complete. Correctness criteria
(spoken audio vs transcript, unit-level decisions vs the Demo reference) are
decided when that testing is needed.

## Order

1. Config, processor protocol, handler, engine plumbing, route (no model yet).
2. MiniCPM-o prompt, talker handoff, Code2Wav flush, sampling (gets audio out).
3. MiniCPM-o input and decode parity, Demo reference recipe.
4. Docs, recipes pages, revisions.
5. PersonaPlex (Live-owned stage-0 path). Nemotron / AURA later.

## Decisions (formerly open questions)

1. PersonaPlex: a Live-owned stage-0 path (6).
2. Context: no predictive trim; trim to 50% of the post-prefix context on
   `context_length_exceeded` (4).
3. Input transcription: dropped for now (4).
4. Encoder cache: optional `max_sessions` per API server, default unbounded (1,
   4).
