Open Questions
1. Go with (a), writing your own stage-0 path for the model
2. No predictive left-trim. On context_length_exceeded, trim to a fixed 50% context length
3. Drop input transcription in your initial implementation
4. Create an optional config to bound max concurrent sessions. Default of None means no bound

Other Gaps
- Whether `live_duplex_unit` is needed will be for you to decide as you implement
- The author was likely trying to prevent the OmniDuplexSessionHandler stack from being activated (which is configured with "session_mode: duplex"). If you need to enforce the /v1/live/sessions path at the model pipeline level, use query parameters in the /v1/live/sessions handler
- Keep openai>=3.18 pin
- This is a design detail you will have to resolve via consulting ./MiniCPM-o-Demo (the implementation from the developers of the official MiniCPM-o)
- Drop predictive left-trim. Only trim during context_length_exceeded handling
- Defer AURA Omni and Nemotron VoiceChat support for later. Your initial implementation will just have MiniCPM-o and PersonaPlex
- We'll decide how to test correctness once that's necessary
- Only include if you need it.

Additional questions
1. Model pipeline code and pcm_buffer look good to share. If there are any additional components you'd like to share ask me
2. system prompt and reference audio must always be kept. It should be 50% of the context not including those parts (which should be relatively small)
3. Yes
4. Per API server. Client gets a simple Error message and graceful shutdown (consult the OpenAI OpenAPI) when they send a session.start to a socket on this endpoint that is over it's allowed number of sessions.
5. It is correct for /v1/live/sessions to use request parameters. The deploy config is to disambiguate between the multiple possible /v1/realtime handlers. Since our handler is the only one serving /v1/live/sessions, it does not make sense to expose a config for this
