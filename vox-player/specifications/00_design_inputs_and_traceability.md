# VoxPlayer Design Inputs and Traceability

## Document Status

- **Document ID:** VoxPlayer-DI-00
- **Status:** Draft groundwork
- **Purpose:** Capture the current VoxSpeak interfaces and constraints that a future VoxPlayer design must treat as the verified baseline.
- **Scope boundary:** This document does **not** define VoxPlayer requirements or architecture. It records only design inputs, verified constraints, and open TBDs.

---

## 1. VoxSpeak Surface VoxPlayer Must Consume

### 1.1 Summary

VoxSpeak currently exposes a gRPC-over-HTTP/2 transport whose required synthesis baseline is **session-based**:

1. A requester creates a synthesis session with `CreateSynthesisSession`.
2. A consumer subscribes with `SubscribeSynthesisSession`.
3. The consumer receives a stream of `StreamMessage` envelopes containing zero or more `AudioChunk` messages, optional `StreamWarning` messages, and a terminal `FinalMetadata` message unless the stream ends with a transport error.
4. A requester may additionally use `CancelSynthesisSession` and `GetSessionStatus`.

The protobuf surface also defines an optional convenience RPC, `StreamSynthesis`, for callers that act as both requester and consumer. The shipped Python gRPC transport client prefers `StreamSynthesis` when available and falls back to the session flow when the RPC is absent or returns `UNIMPLEMENTED`.

For VoxPlayer, that means the **minimum verified remote integration contract** is:

- submit text as a `SynthesisRequest`,
- support streaming PCM audio playback from `StreamMessage.audio`,
- consume terminal `FinalMetadata`,
- handle pseudo-streaming warnings and transport errors,
- retain the ability to cancel or poll session state when using the session flow,
- send requester/consumer role and platform hints through gRPC metadata when deployment policy requires them.

### 1.2 Verified remote synthesis/control surface

The current protobuf/service baseline exposes these RPC groups relevant to a player application:

- **Synthesis/session lifecycle**
  - `CreateSynthesisSession`
  - `SubscribeSynthesisSession`
  - `CancelSynthesisSession`
  - `GetSessionStatus`
  - `StreamSynthesis` (optional convenience)
- **Discovery / preflight / control plane**
  - `ListPersonalities`
  - `DescribePersonality`
  - `ListEngines`
  - `EngineCapabilities`
  - `GetGlobalSynthesisLimits`
  - `ValidateSynthesis`
- **Operator-focused but implementation-visible**
  - `GetRuntimeMetrics`
  - `LintPersonalities`
  - `ReloadPersonalities`

### 1.3 Audio and metadata shape VoxPlayer must be prepared for

Confirmed transport-visible data:

- audio is delivered as raw PCM chunks, not compressed wire codecs,
- each chunk includes explicit `AudioFormat`, monotonic `index`, PCM bytes, and `is_last`,
- stream termination is represented by `FinalMetadata` on successful completion,
- synthesis metadata includes resolved `personality_id`, `engine_id`, `model_id`, timing data, cache information, warnings, and text-processing trace entries,
- pseudo-stream fallback is surfaced with explicit warning codes both in-band (`StreamWarning`) and in terminal metadata warnings.

---

## 2. Verified Integration Constraints for VoxPlayer

The items below are verified against repository docs, implementation, or both.

### 2.1 Transport and session model

1. **gRPC over HTTP/2 is the only approved remote transport baseline.**
   - Source basis: docs + proto/service.
   - VoxPlayer should be designed around the gRPC contract, not a guessed REST/WebSocket surface.

2. **Session-based synthesis is the required baseline; direct streaming is optional.**
   - Source basis: docs + proto + service + Python client.
   - VoxPlayer must not require `StreamSynthesis` to exist on every server.

3. **A successful session subscription delivers zero or more audio chunks followed by one terminal metadata message.**
   - Source basis: docs + proto + service.
   - VoxPlayer playback/state handling must keep consuming until terminal metadata or transport failure.

4. **Subscription, cancellation, and status are role-gated at the transport boundary.**
   - Source basis: docs + implementation.
   - Requester-capable callers create/status/cancel sessions; consumer-capable callers subscribe.

5. **Role metadata is carried in gRPC metadata, not in the protobuf request body.**
   - Source basis: docs + implementation.
   - Approved role keys are `x-voxspeak-role`, `voxspeak-role`, and `role`.

6. **Queue priority is also transport metadata, not a request-body field.**
   - Source basis: docs + implementation.
   - Approved keys are `x-voxspeak-priority`, `voxspeak-priority`, and `priority`; values are normalized to integer classes `0..2`, with default `1` on omission/invalid input.

### 2.2 Authorization and consumer identity

7. **Session subscription uses a session-scoped `consumer_token`.**
   - Source basis: docs + proto + implementation.
   - The token is returned by `CreateSynthesisSessionResponse` and sent on `SubscribeSynthesisSessionRequest`.

8. **Missing consumer tokens are rejected by default, but deployments can relax the requirement.**
   - Source basis: docs + implementation.
   - Even when missing-token enforcement is disabled, supplied mismatched tokens are still rejected.

9. **Consumer platform enforcement is deployment policy, not universal baseline behavior.**
   - Source basis: docs + implementation.
   - The service can enforce Windows-only consumers and/or require that the subscriber platform match the session's `consumer_platform` hint.

10. **Platform identity is derived from transport metadata, with canonical Windows aliases.**
    - Source basis: implementation + tests.
    - Consumer keys are preferred, but the implementation can fall back to requester-platform metadata during subscription checks.

### 2.3 Request-shape and playback constraints

11. **`SynthesisRequest` output shapes are mode-sensitive.**
    - Source basis: docs + proto + public Python API.
    - `file_output` is required for file mode; `stream_options` is required for stream mode.

12. **`StreamSynthesis` only accepts streaming-shaped requests.**
    - Source basis: docs + implementation.
    - The server rejects requests whose `output_mode` is not `OUTPUT_MODE_STREAM`, requests missing `stream_options`, and requests that include `file_output`.

13. **Realtime streaming is not guaranteed.**
    - Source basis: docs + implementation + tests.
    - If a request requires native streaming and the server cannot provide it, the call fails with `INVALID_ARGUMENT` instead of silently degrading.

14. **Pseudo-streaming fallback is a first-class current behavior.**
    - Source basis: docs + implementation + tests.
    - VoxPlayer must treat fallback warnings as part of normal interoperability, not as undefined server behavior.

15. **The wire audio contract is explicit PCM format metadata plus raw bytes.**
    - Source basis: docs + proto + implementation.
    - VoxPlayer must not assume compressed containers on the wire.

16. **Chunk format mismatches are treated as contract violations.**
    - Source basis: implementation.
    - The server aborts the subscription when emitted stream chunks do not match the negotiated/declared format.

17. **Successful playback completion should be anchored to terminal metadata, not only `AudioChunk.is_last`.**
    - Source basis: proto + service.
    - The server constructs chunk messages with `is_last=false` during subscription and uses `FinalMetadata` as the authoritative success terminator.

### 2.4 Compatibility and preflight constraints

18. **API version negotiation is header-based and enforced by the server.**
    - Source basis: docs + implementation.
    - `RequestHeader.api_version` is optional; omission selects the server default. Unsupported versions are rejected with `INVALID_ARGUMENT`.

19. **Version-policy advertisement is published in gRPC initial metadata.**
    - Source basis: docs + implementation.
    - VoxPlayer should not require response-body DTO fields for version policy.

20. **Global synthesis limits are a distinct preflight surface.**
    - Source basis: docs + proto + implementation.
    - Newer clients must degrade gracefully when an older server returns `UNIMPLEMENTED` for `GetGlobalSynthesisLimits`.

21. **Engine capability discovery and global limits are separate concerns.**
    - Source basis: docs + implementation + tests.
    - Engine capability responses can report adapter/server streaming capability details, but admission limits come from `GetGlobalSynthesisLimits`.

---

## 3. Protocol / API / Transport Details Confirmed by Code

This section records concrete details verified directly from the shipped implementation and protobuf schema.

### 3.1 RPC and message details confirmed by proto/service code

- Service name: `voxcore.speech.v1.SpeechService`.
- Required synthesis baseline RPCs:
  - `CreateSynthesisSession`
  - `SubscribeSynthesisSession`
  - `CancelSynthesisSession`
  - `GetSessionStatus`
- Optional convenience RPC:
  - `StreamSynthesis`
- `CreateSynthesisSessionResponse` includes:
  - `session_id`
  - `expires_at`
  - `consumer_token`
  - optional `early_metadata`
- `SubscribeSynthesisSessionRequest` includes:
  - `session_id`
  - `consumer_token`
  - optional `desired_audio_format`
- `GetSessionStatusResponse` includes:
  - `session_id`
  - `state`
  - best-effort `progress_percent`
  - optional `metadata`
- `CreateSynthesisSessionRequest` includes optional `consumer_platform` for downstream subscription policy.

### 3.2 Request and stream payload details confirmed by code

- `SynthesisRequest` on the wire currently contains:
  - `text`
  - `personality_id`
  - optional `engine_id`
  - `output_mode`
  - optional `audio_format`
  - `enable_postprocessing`
  - `enable_qc`
  - optional `cache_policy`
  - optional `file_output`
  - optional `stream_options`
  - optional `seed`
  - optional `reproducibility_flags`
  - optional `verbose_debug_logging`
- `StreamOptions.mode` supports:
  - `STREAM_MODE_REALTIME`
  - `STREAM_MODE_FAST_AS_POSSIBLE`
- `AudioChunk` carries:
  - `format.sample_rate_hz`
  - `format.channels`
  - `format.sample_format`
  - `index`
  - `pcm`
  - `is_last`
- `SynthesisMetadata` currently includes:
  - original and processed text,
  - resolved personality / engine / model identifiers,
  - flexible synthesis and post-processing structures,
  - QC results,
  - cache information,
  - timings,
  - warning codes,
  - text-processing trace entries.

### 3.3 Warning, fallback, and compatibility details confirmed by code

- The implementation uses explicit pseudo-stream warning codes, including:
  - `PSEUDO_STREAMING_BUFFER_THEN_CHUNK`
  - `PSEUDO_STREAMING_ENGINE_CAPABILITY_FALLBACK`
  - `PSEUDO_STREAMING_RUNTIME_UNAVAILABLE_FALLBACK`
  - `PSEUDO_STREAMING_SERVER_FALLBACK`
  - `PSEUDO_STREAMING_FILE_OUTPUT_FALLBACK`
  - `PSEUDO_STREAMING_POSTPROCESSING_FALLBACK`
- The implementation also records lower-level stream reason codes inside resolved metadata, such as:
  - `ENGINE_CAPABILITY_SUPPORTS_STREAMING_FALSE`
  - `ENGINE_RUNTIME_NATIVE_STREAMING_UNAVAILABLE`
  - `SERVER_NATIVE_STREAMING_NOT_IMPLEMENTED`
  - `STREAM_OUTPUT_USES_FILE_PERSISTENCE_FALLBACK`
  - `STREAM_POSTPROCESSING_REQUIRES_BUFFERED_FALLBACK`
  - `STREAM_MODE_REQUIRES_NATIVE_STREAMING`
- The service publishes API-version policy via initial metadata keys:
  - `x-voxspeak-api-default-version`
  - `x-voxspeak-api-supported-versions`
  - `x-voxspeak-api-policy-mode`
  - `x-voxspeak-api-version-policy`
  - and, when applicable, `x-voxspeak-api-effective-version` / `x-voxspeak-api-requested-version`
- The current server implementation advertises default wire version `1.5` and supported versions `1.3` and `1.5`.

### 3.4 Client-facing implementation details relevant to VoxPlayer

- The shipped Python transport client:
  - treats `StreamSynthesis` as preferred but optional,
  - falls back to session create + subscribe when `StreamSynthesis` is unavailable,
  - exposes `get_global_synthesis_limits()` with a typed unavailable fallback when the server returns `UNIMPLEMENTED`,
  - returns caller-supplied `request_id` for streaming results and server-canonical request IDs for non-stream results.
- The public Python package validates only the currently documented in-process DTO fields and does **not** expose `seed` or `reproducibility_flags` on `voxspeak.types.SynthesisRequest`, even though those fields exist on the wire protobuf.

### 3.5 Doc / implementation discrepancies to carry forward explicitly

1. **Wire `SynthesisRequest` vs public Python `SynthesisRequest`**
   - The protobuf schema includes `seed` and `reproducibility_flags`.
   - The public Python DTO and top-level convenience API do not currently expose those fields.
   - For VoxPlayer groundwork, the transport/proto is the safer source of truth for remote interoperability, while the Python API is the source of truth for current in-process convenience functions.

2. **Operator metrics availability vs player needs**
   - `GetRuntimeMetrics` is implemented and documented in the transport/proto baseline.
   - The public package surface intentionally does not require a top-level runtime-metrics function.
   - VoxPlayer should therefore treat runtime metrics as optional operator/deployment integration, not as a required player dependency.

---

## 4. Design Assumptions That Remain TBD

The following items are **not** confirmed for VoxPlayer and must remain TBD until a dedicated VoxPlayer spec resolves them.

1. **Whether VoxPlayer will always use session-based RPCs, always use direct streaming when available, or negotiate between them at runtime.**
2. **Whether VoxPlayer itself will send `CreateSynthesisSession` and `SubscribeSynthesisSession`, or whether another local component/process will own one of those roles.**
3. **Whether VoxPlayer will expose personality and engine selection directly to users, cache them locally, or fetch them on-demand.**
4. **Whether VoxPlayer will surface validation/preflight operations (`ValidateSynthesis`, `EngineCapabilities`, `GetGlobalSynthesisLimits`) in the UI or use them only internally.**
5. **What local audio buffering, jitter tolerance, and playback scheduling policies VoxPlayer needs for realtime vs fast-as-possible streams.**
6. **Whether VoxPlayer will persist received audio locally, and if so, what file lifecycle and metadata persistence rules should apply on the Windows side.**
7. **Whether VoxPlayer will surface raw warning codes/messages directly to users, map them into UX states, or log them only diagnostically.**
8. **How VoxPlayer should authenticate to VoxSpeak in production deployments (for example TLS, bearer auth, or mTLS interceptors).** The repository shows those capabilities on the server side, but no VoxPlayer authentication profile is yet specified.
9. **What recovery/retry policy VoxPlayer should apply after `UNAVAILABLE`, `RESOURCE_EXHAUSTED`, `DEADLINE_EXCEEDED`, `CANCELLED`, or version negotiation failures.**
10. **Whether VoxPlayer will use protobuf-generated client code directly in .NET/Windows or an intermediate local bridge.**
11. **Whether VoxPlayer should depend on optional fields such as `desired_audio_format`, `early_metadata`, or detailed text-processing trace content for core UX.**
12. **Any assumption about deterministic replay or reproducibility semantics beyond what VoxSpeak already discloses.**

---

## 5. First-Pass Traceability from VoxPlayer Concerns to VoxSpeak Specs

| VoxPlayer concern | VoxSpeak FR | VoxSpeak ARCH | VoxSpeak API | VoxSpeak TRANSPORT | VoxSpeak PROTO | Notes |
|---|---|---|---|---|---|---|
| Submit user text for synthesis | FR §4.1, §8.2 | ARCH §4.1, §5.1 | API §3.1, §6.1 | TRANSPORT §6.2, §6.3 | `SynthesisRequest`, `CreateSynthesisSession`, `StreamSynthesis` | Core text-to-speech request path. |
| Stream audio for playback | FR §11.1, §12.1 | ARCH §5.2, §6 | API §6.2 | TRANSPORT §5, §6.2, §6.3, §7 | `AudioChunk`, `StreamMessage`, `FinalMetadata` | Use transport stream as playback source of truth. |
| Handle pseudo-stream fallback safely | FR §5.6, §11.1, §12.* | ARCH §5.2.2-.4, §6.4 | API §6.2, §8 | TRANSPORT §7.5-.9 | `StreamWarning`, metadata warnings | Must not assume all streams are native realtime streams. |
| Cancel playback / request state | FR §8.2, §9.4 | ARCH §4.7, §6.2 | API streaming/status abstractions | TRANSPORT §6.2.5, §8 | `CancelSynthesisSession`, `GetSessionStatus`, `SessionState` | Relevant if VoxPlayer owns session lifecycle. |
| Select personalities / inspect engines | FR §3.5-.6, §8.2 | ARCH §4.1, §4.3, §4.4, §7 | API §3.1, §6.6-.7 | TRANSPORT §6.1 | list/describe/introspection RPCs | Use discovery instead of hard-coding personalities/engines. |
| Respect server limits / admission | FR §8.4 | ARCH §4.7.2-.3, §7.2-.3 | API §6.6.4-.5 | TRANSPORT §6.1, §9 | `GetGlobalSynthesisLimits`, gRPC errors | Important for preflight and overload UX. |
| Handle requester/consumer roles and policy metadata | FR §10.4 | ARCH §4.2.2, §10 | API §9 | TRANSPORT §4, §12 | transport metadata + `consumer_token` | Transport-only, not in public in-process DTOs. |
| Track compatibility / server version policy | FR §8.4 | ARCH §4.2.3-.4, §7.3-.4 | API §8.4-.5 | TRANSPORT §10 | `RequestHeader.api_version`, initial metadata | VoxPlayer should treat metadata as version-policy source. |
| Future upstream text routing via VoxThink | N/A in VoxSpeak FR | TRANSPORT §2.2 mentions VoxThink/equivalent requester services | N/A | TRANSPORT §2.2 | N/A | Only a topology hint today; no VoxThink API is defined. |

---

## 6. Do Not Assume

This section is intentionally strict because VoxThink does not yet exist as a specified module.

1. **Do not assume a VoxThink RPC, protobuf package, or local IPC interface.**
2. **Do not assume VoxPlayer will always talk directly to VoxSpeak.** The future architecture may allow direct-to-VoxSpeak and via-VoxThink modes.
3. **Do not assume how text is transformed before VoxSpeak receives it when VoxThink is introduced.**
4. **Do not assume VoxThink owns personality selection, engine selection, request IDs, session IDs, queue priorities, or consumer tokens.**
5. **Do not assume VoxThink terminates or proxies the VoxSpeak audio stream.** That behavior is currently unspecified.
6. **Do not assume a combined VoxThink+VoxSpeak session model.** Only VoxSpeak's current session model is verified.
7. **Do not assume conversational, planning, or dialog-state semantics upstream of VoxSpeak.** VoxSpeak currently exposes TTS synthesis semantics, not dialog orchestration.
8. **Do not assume any additional metadata contract between VoxPlayer and VoxThink for Windows focus state, playback acknowledgements, user typing cadence, or interruption handling.**
9. **Do not assume that VoxThink will preserve, replace, or synthesize VoxSpeak request IDs and warnings.**
10. **Do not assume that future VoxThink routing will remove the need for VoxPlayer to understand current VoxSpeak transport errors, fallback warnings, and terminal metadata.**

---

## 7. Initial Design Guidance for the Next VoxPlayer Spec Phase

Given the verified baseline above, the next VoxPlayer documents should be written as if:

- VoxSpeak is a gRPC dependency with session-based streaming as the mandatory interoperable baseline.
- Audio playback must operate on PCM chunk streams and terminal metadata, not guessed file-download semantics.
- Warning-aware streaming behavior is part of the contract.
- Role, priority, platform, token, and API-version concerns belong to transport integration code, not to generic text-entry or playback domain models.
- Any future VoxThink integration must be represented as an upstream abstraction point with explicit TBD boundaries.

