# VoxCore – VoxPlayer Windows Client

## Transport and Integration Specification, Version 0.1 (Draft)

**Document ID:** VoxPlayer-TRANSPORT-v0.1  
**Derived from:** VoxPlayer-DI-00; VoxPlayer-FR-v0.1; VoxPlayer-ARCH-v0.1; VoxSpeak-FR-v1.5; VoxSpeak-ARCH-v1.5; VoxSpeak-API-v1.5; VoxSpeak-TRANSPORT-v1.5; VoxSpeak-PROTO-v1.2; verified `vox-speak/` implementation  
**Project:** VoxCore  
**Subsystem:** VoxPlayer (`docs/vox-player`)  
**Status:** Draft

**Revision note:** VoxPlayer-TRANSPORT-v0.1 defines the current direct-to-VoxSpeak integration baseline using only repository-verified VoxSpeak APIs, protobuf schema, transport behavior, tests, and implementation. It separates verified behavior from assumptions and constrains future VoxThink routing to an extension point rather than a present-day contract.

---

## 1. Purpose and Scope

1.1. This document specifies how VoxPlayer shall integrate with remote synthesis services for the current design baseline.

1.2. For this revision, the only verified remote synthesis target is VoxSpeak over gRPC over HTTP/2.

1.3. This document defines:

* verified direct VoxSpeak interaction patterns,
* the adapter boundaries between VoxPlayer and transport-specific code,
* request, stream, session, cancellation, error, retry, and correlation expectations verified from code and tests,
* compatibility expectations for optional or older-server behavior,
* constrained extension points for a future upstream route such as VoxThink.

1.4. This document does **not** define Windows UI behavior, input capture UX, audio-device UX, or application visual design.

1.5. Where repository documentation and implementation differ, this document shall call out the discrepancy explicitly and shall treat the shipped implementation as the current source of truth unless otherwise noted.

---

## 2. Verified Source Baseline

2.1. The verified current remote service is `voxspeak.v1.VoxSpeakService` as defined in `vox-speak/proto/voxspeak/v1/voxspeak.proto` and implemented in `vox-speak/server/voxspeak_server/service.py`.

2.2. The verified current client-side reference behavior is the shipped Python gRPC transport client in `vox-speak/server/voxspeak/transport_grpc.py`.

2.3. The most relevant verification sources for this document are:

* VoxPlayer design/requirements/architecture specifications,
* VoxSpeak API, transport, and protobuf specifications,
* the VoxSpeak server implementation,
* the VoxSpeak gRPC transport client implementation,
* black-box and service-level tests under `vox-speak/server/tests/`.

2.4. VoxPlayer shall treat implementation-verified behavior as normative for integration even when some prose documentation is broader, less specific, or partially stale.

---

## 3. Present-Day Transport Baseline

### 3.1 Protocol and service identity

3.1.1. VoxPlayer shall integrate with VoxSpeak using **gRPC over HTTP/2** only.

3.1.2. The authoritative service name is `voxspeak.v1.VoxSpeakService`.

3.1.3. VoxPlayer shall not assume any verified REST, WebSocket, named-pipe, or custom socket transport for VoxSpeak.

### 3.2 Remote RPC surface relevant to VoxPlayer

3.2.1. The currently verified synthesis/session RPCs are:

* `CreateSynthesisSession`
* `SubscribeSynthesisSession`
* `CancelSynthesisSession`
* `GetSessionStatus`
* `StreamSynthesis` (optional convenience RPC)

3.2.2. The currently verified discovery/preflight RPCs potentially relevant to VoxPlayer are:

* `ListPersonalities`
* `DescribePersonality`
* `ListEngines`
* `EngineCapabilities`
* `GetGlobalSynthesisLimits`
* `ValidateSynthesis`

3.2.3. VoxPlayer shall not require operator-oriented RPCs such as `GetRuntimeMetrics`, `LintPersonalities`, or `ReloadPersonalities` for end-user playback interoperability.

### 3.3 Required interoperability model

3.3.1. The required interoperability baseline is the **session-oriented** flow:

1. send `CreateSynthesisSession`,
2. persist the returned `session_id` and `consumer_token`,
3. send `SubscribeSynthesisSession`,
4. consume `StreamMessage` items until terminal success metadata or transport failure,
5. optionally use `CancelSynthesisSession` and `GetSessionStatus`.

3.3.2. VoxPlayer may use `StreamSynthesis` as an optimization or convenience path when available.

3.3.3. VoxPlayer shall not depend on `StreamSynthesis` existing on every server.

3.3.4. The shipped VoxSpeak Python transport client already follows this pattern by attempting `StreamSynthesis` and falling back to session creation/subscription when the RPC is absent or returns `UNIMPLEMENTED`.

---

## 4. Verified Interaction Patterns

### 4.1 Session-oriented requester/consumer flow

4.1.1. `CreateSynthesisSession` accepts a request envelope containing `RequestHeader`, `SynthesisRequest`, and optional `consumer_platform`.

4.1.2. `CreateSynthesisSessionResponse` returns:

* `header`,
* `session_id`,
* `expires_at`,
* `consumer_token`.

4.1.3. `SubscribeSynthesisSessionRequest` carries:

* `header`,
* `session_id`,
* `consumer_token`,
* optional `desired_audio_format`.

4.1.4. A successful subscription emits zero or more `StreamMessage.audio` payloads, then zero or more `StreamMessage.warning` payloads, then exactly one `StreamMessage.final` payload unless the call terminates with a gRPC error.

4.1.5. In the current implementation, emitted subscription `AudioChunk` messages use `is_last=false`; successful completion is authoritatively indicated by `FinalMetadata`, not by `AudioChunk.is_last`.

4.1.6. `CancelSynthesisSession` is best-effort requester-initiated cancellation.

4.1.7. `GetSessionStatus` provides a snapshot including session state, best-effort progress percentage, and final metadata when available.

### 4.2 Direct streaming convenience flow

4.2.1. `StreamSynthesis` is implemented as a convenience wrapper that internally creates a session and immediately subscribes to it.

4.2.2. `StreamSynthesis` currently rejects malformed request shapes when:

* `request` is missing,
* `output_mode` is not `OUTPUT_MODE_STREAM`,
* `stream_options` is missing,
* `file_output` is supplied.

4.2.3. The current implementation propagates the effective API version from the direct-stream request header into the nested session create/subscribe calls.

4.2.4. Because `StreamSynthesis` internally reuses `CreateSynthesisSession` and `SubscribeSynthesisSession`, VoxPlayer should provide role metadata that is valid for both requester-capable and consumer-capable operations when using the direct-stream route.

4.2.5. **Documented/implemented nuance:** the VoxSpeak transport prose describes `StreamSynthesis` as a consumer convenience path, but the implementation reuses requester-gated session creation logic internally. The safest interoperable assumption for VoxPlayer is therefore to send a combined role such as `both` or `requester_consumer` when role metadata is used.

### 4.3 Discovery and preflight flow

4.3.1. Personality and engine discovery are normal unary gRPC calls and do not require session handling.

4.3.2. `GetGlobalSynthesisLimits` is a distinct preflight surface for server-global admission and planning limits.

4.3.3. The VoxSpeak Python transport client treats `UNIMPLEMENTED` from `GetGlobalSynthesisLimits` as a compatibility case and returns a typed “limits unavailable” result rather than treating it as malformed behavior.

4.3.4. `ValidateSynthesis` is available as a compatibility-safe preflight API. The Python client also contains a local fallback path when the RPC is absent.

---

## 5. Wire Contract VoxPlayer Must Consume

### 5.1 Request header and correlation fields

5.1.1. The wire-level request header is `RequestHeader` with:

* `request_id`,
* `api_version`.

5.1.2. `request_id` is caller-supplied correlation input. If it is omitted, the server may generate one.

5.1.3. The wire-level response header is `ResponseHeader` with:

* `request_id`,
* `api_version`,
* optional `server_id`.

5.1.4. VoxPlayer shall preserve both the locally chosen request identifier and the canonical request identifier returned by VoxSpeak when they differ.

### 5.2 Synthesis request shape

5.2.1. The current protobuf `SynthesisRequest` includes these verified fields:

* `text`
* `personality_id`
* `engine_id`
* `output_mode`
* `audio_format`
* `enable_postprocessing`
* `enable_qc`
* `cache_policy`
* `file_output`
* `stream_options`
* optional `seed`
* `reproducibility_flags`
* optional `verbose_debug_logging`

5.2.2. VoxPlayer shall only populate request fields that exist in the protobuf contract and are supported by the selected workflow.

5.2.3. VoxPlayer shall not invent request-body fields for role signaling, platform identity, queue priority, authorization tokens, retries, or route selection.

5.2.4. `file_output` is required for file mode.

5.2.5. `stream_options` is required for stream mode.

### 5.3 Stream payload shape

5.3.1. `StreamMessage` is a `oneof` envelope containing:

* `audio`,
* `final`,
* `warning`.

5.3.2. `AudioChunk` carries:

* `format`,
* monotonic `index`,
* raw PCM bytes in `pcm`,
* `is_last`.

5.3.3. `AudioFormat` carries:

* `sample_rate_hz`,
* `channels`,
* `sample_format`.

5.3.4. The current wire contract is raw PCM with explicit format metadata; compressed wire codecs are out of scope.

5.3.5. `FinalMetadata` contains `ResponseHeader` and `SynthesisMetadata`.

5.3.6. `SynthesisMetadata` currently includes timing data, cache information, warning codes, processed/original text, resolved personality/engine/model identifiers, synthesis/post-processing structures, QC data, and text-processing trace entries.

---

## 6. Transport Metadata and Session Policy

### 6.1 Role metadata

6.1.1. Role signaling is transport metadata, not part of `SynthesisRequest`.

6.1.2. The current server recognizes these metadata keys for role:

* `x-voxspeak-role`
* `voxspeak-role`
* `role`

6.1.3. Role values are normalized case-insensitively, with hyphens normalized to underscores.

6.1.4. Requester-capable values are:

* `requester`
* `both`
* `requester_consumer`

6.1.5. Consumer-capable values are:

* `consumer`
* `both`
* `requester_consumer`

6.1.6. Current server behavior is permissive when role metadata is omitted entirely, but restrictive when a non-capable role is explicitly supplied for a gated operation.

6.1.7. VoxPlayer shall therefore treat role metadata as deployment/policy-facing configuration and, when emitting it, shall send values appropriate for the chosen call pattern.

### 6.2 Queue-priority metadata

6.2.1. Queue priority is transport metadata, not a protobuf request-body field.

6.2.2. The current server recognizes these metadata keys:

* `x-voxspeak-priority`
* `voxspeak-priority`
* `priority`

6.2.3. Priority values are normalized to bounded integer classes `0..2`.

6.2.4. When queue-priority metadata is missing or invalid, the server defaults to priority class `1`.

6.2.5. VoxPlayer shall expose queue priority, if at all, as an endpoint/policy setting that maps only to the verified bounded class semantics.

### 6.3 Platform metadata and session hints

6.3.1. Consumer/requester platform identity is currently derived from transport metadata.

6.3.2. Recognized consumer-platform metadata keys are:

* `x-voxspeak-platform`
* `voxspeak-platform`
* `platform`

6.3.3. Recognized requester-platform metadata keys are:

* `x-voxspeak-requester-platform`
* `voxspeak-requester-platform`
* `requester-platform`

6.3.4. Session creation also accepts a request-body `consumer_platform` hint for downstream subscription checks.

6.3.5. The implementation canonicalizes Windows aliases such as `win`, `win32`, `win64`, `windows`, and `windows-*` to `windows`.

6.3.6. Subscription policy may prefer consumer metadata keys but can fall back to requester-platform metadata during platform evaluation.

6.3.7. VoxPlayer shall therefore separate:

* **transport platform metadata** used per call, and
* **session consumer platform hint** supplied at session creation when configured.

### 6.4 Consumer token semantics

6.4.1. `consumer_token` is a session-scoped authorization token returned by `CreateSynthesisSessionResponse`.

6.4.2. `SubscribeSynthesisSession` sends `consumer_token` in the request body.

6.4.3. Current default server behavior requires a non-empty consumer token.

6.4.4. Deployments may disable missing-token enforcement, but a supplied mismatched token is still rejected.

6.4.5. VoxPlayer shall treat the token as ephemeral runtime state and shall not treat it as a user-facing credential.

---

## 7. Session Lifecycle, Cancellation, and Completion

### 7.1 Session lifetime

7.1.1. The current server creates sessions with a fixed TTL of 120 seconds.

7.1.2. `CreateSynthesisSessionResponse.expires_at` publishes the session expiration time.

7.1.3. VoxPlayer shall treat the published expiration timestamp as authoritative rather than hard-coding the current 120-second implementation constant.

### 7.2 Session states

7.2.1. The current session store and public status flow expose these states:

* `QUEUED`
* `RUNNING`
* `COMPLETED`
* `CANCELLED`
* `FAILED`
* `EXPIRED`

7.2.2. VoxPlayer shall map these states into route-neutral request and playback states without exposing raw protobuf enums as application-domain types.

### 7.3 Cancellation behavior

7.3.1. Client-side cancellation of a session subscription stream results in cancellation of the active call and a best-effort cancel RPC through the session handle abstraction.

7.3.2. Requester-initiated `CancelSynthesisSession` marks the session cancelled and notifies waiting subscribers.

7.3.3. A cancelled subscription ends with gRPC `CANCELLED` unless a more specific mapped terminal error code is present.

7.3.4. The VoxSpeak Python client reports cancelled streams as `StreamStatus.CANCELLED` and converts transport failures into typed domain errors.

### 7.4 Expiration and failure behavior

7.4.1. Expired sessions terminate subscriptions with gRPC `DEADLINE_EXCEEDED` unless a more specific mapped terminal error code is present.

7.4.2. Failed sessions terminate subscriptions with a mapped gRPC error code or `INTERNAL` when no more specific mapping is available.

7.4.3. VoxPlayer shall treat stream transport termination without terminal metadata as an unsuccessful completion, even if some audio was already received.

---

## 8. Streaming, Fallback, and Playback-Relevant Expectations

### 8.1 Streaming modes

8.1.1. The current stream mode enum supports:

* `STREAM_MODE_REALTIME`
* `STREAM_MODE_FAST_AS_POSSIBLE`

8.1.2. VoxSpeak distinguishes native streaming from pseudo-stream fallback.

8.1.3. VoxPlayer shall not assume that a stream request implies true incremental synthesis from the engine.

### 8.2 Pseudo-stream fallback

8.2.1. Pseudo-stream fallback is a current first-class behavior, not an undefined edge case.

8.2.2. When pseudo-stream fallback is used, VoxSpeak emits machine-readable warning codes both:

* as in-band `StreamWarning` payloads, and
* in terminal `SynthesisMetadata.warnings`.

8.2.3. Verified fallback warning codes include:

* `PSEUDO_STREAMING_BUFFER_THEN_CHUNK`
* `PSEUDO_STREAMING_ENGINE_CAPABILITY_FALLBACK`
* `PSEUDO_STREAMING_RUNTIME_UNAVAILABLE_FALLBACK`
* `PSEUDO_STREAMING_SERVER_FALLBACK`
* `PSEUDO_STREAMING_FILE_OUTPUT_FALLBACK`
* `PSEUDO_STREAMING_POSTPROCESSING_FALLBACK`

8.2.4. VoxPlayer shall surface pseudo-stream fallback as a diagnosable transport/runtime condition and shall not misclassify it as a fatal protocol error.

### 8.3 Required-native rejection behavior

8.3.1. When the requested stream mode requires native streaming and the resolved execution path cannot provide it, the current server rejects the request with `INVALID_ARGUMENT` rather than silently degrading.

8.3.2. VoxPlayer shall distinguish between:

* acceptable pseudo-stream fallback, and
* explicit failure when the selected stream semantics require native streaming.

### 8.4 Audio-format consistency

8.4.1. The server negotiates/records an expected stream chunk format when the session is created.

8.4.2. During subscription, if actual emitted chunks do not match the negotiated format, the server aborts the stream as a contract violation.

8.4.3. VoxPlayer shall treat format changes within a single playback item as invalid unless a future approved contract explicitly allows renegotiation.

---

## 9. Error Mapping and Retry Expectations

### 9.1 Verified server-side gRPC outcomes relevant to VoxPlayer

9.1.1. The current implementation uses at least these gRPC outcomes in player-relevant flows:

* `INVALID_ARGUMENT`
* `FAILED_PRECONDITION`
* `PERMISSION_DENIED`
* `NOT_FOUND`
* `RESOURCE_EXHAUSTED`
* `CANCELLED`
* `DEADLINE_EXCEEDED`
* `INTERNAL`
* `UNIMPLEMENTED`
* `UNAUTHENTICATED` when optional auth interceptors are enabled

9.1.2. VoxPlayer shall preserve the gRPC status, message, and any structured reason details that can be extracted without binding application logic directly to raw gRPC exception types.

### 9.2 Admission/overload behavior

9.2.1. Queue admission failures are reported as `RESOURCE_EXHAUSTED`.

9.2.2. The current implementation includes structured reason codes such as global or engine queue-full conditions in the abort message details payload.

9.2.3. VoxPlayer shall treat `RESOURCE_EXHAUSTED` as a remote admission failure rather than a local playback or parsing error.

### 9.3 Client-side retry behavior verified from VoxSpeak reference client

9.3.1. The shipped Python gRPC transport client only retries **idempotent** operations.

9.3.2. In the current client, retryable gRPC statuses are:

* `UNAVAILABLE`
* `DEADLINE_EXCEEDED`
* `RESOURCE_EXHAUSTED`
* `ABORTED`

9.3.3. Non-idempotent operations such as `CreateSynthesisSession` and `StreamSynthesis` are not retried automatically.

9.3.4. Idempotent setup/control operations such as subscribe setup, status, cancel, validate, and some discovery calls may be retried according to client configuration.

9.3.5. VoxPlayer is not required to copy the Python client implementation exactly, but it shall preserve the same safety principle: do not blindly retry non-idempotent synthesis submission operations.

### 9.4 Timeouts

9.4.1. The VoxSpeak Python gRPC client distinguishes:

* connect timeout,
* unary request timeout,
* stream idle timeout.

9.4.2. VoxPlayer shall preserve separate configuration boundaries for at least connection establishment and active-call timeout behavior, even if final UI/settings exposure is deferred.

---

## 10. Correlation, Diagnostics, and Metadata Expectations

### 10.1 Correlation identifiers

10.1.1. VoxPlayer shall generate or preserve a player-local submission identifier for its own history and diagnostics.

10.1.2. VoxPlayer shall send a transport `request_id` via `RequestHeader` when possible.

10.1.3. VoxPlayer shall record the following independently when available:

* local submission ID,
* remote canonical `request_id`,
* `session_id`,
* route identifier,
* endpoint profile identity.

10.1.4. VoxPlayer shall not assume that local submission ID, caller-supplied request ID, canonical request ID, and session ID are identical.

### 10.2 Final metadata capture

10.2.1. Successful remote completion shall capture terminal `SynthesisMetadata` for diagnostics.

10.2.2. VoxPlayer should preserve, at minimum, the following metadata fields when available:

* resolved `personality_id`,
* resolved `engine_id`,
* resolved `model_id`,
* warning codes,
* timing data,
* cache information,
* processed/original text according to local privacy policy,
* text-processing trace entries when returned.

10.2.3. VoxPlayer shall clearly distinguish remote warning codes from local playback warnings.

### 10.3 API-version policy advertisement

10.3.1. `RequestHeader.api_version` is optional on the wire.

10.3.2. When omitted, the server applies its default API version.

10.3.3. Unsupported versions are rejected with `INVALID_ARGUMENT` and reason details naming the requested version, supported versions, and default version.

10.3.4. The server publishes version policy through gRPC initial metadata using keys including:

* `x-voxspeak-api-default-version`
* `x-voxspeak-api-supported-versions`
* `x-voxspeak-api-policy-mode`
* `x-voxspeak-api-version-policy`

10.3.5. VoxPlayer shall treat response initial metadata, not ad-hoc response DTO fields, as the authoritative source for wire-version policy advertisement.

---

## 11. VoxPlayer Adapter Boundaries

### 11.1 Route-neutral boundary required by VoxPlayer

11.1.1. VoxPlayer shall isolate transport-specific behavior behind a route adapter interface.

11.1.2. The route adapter boundary shall be responsible for:

* endpoint/channel creation,
* auth/TLS/metadata injection,
* protobuf request construction,
* gRPC invocation,
* stream envelope decoding,
* session token/session ID tracking,
* transport error mapping,
* retry policy execution,
* extraction of remote correlation and metadata details.

11.1.3. VoxPlayer application and playback layers shall consume route-neutral DTOs/events rather than protobuf-generated classes.

### 11.2 VoxSpeak adapter responsibilities

11.2.1. The direct VoxSpeak adapter shall own all knowledge of:

* `voxspeak_pb2` and `voxspeak_pb2_grpc`,
* gRPC metadata key names,
* `consumer_token`,
* `session_id`,
* `RequestHeader` / `ResponseHeader`,
* gRPC status mapping,
* version-policy initial metadata.

11.2.2. The adapter shall translate VoxSpeak `StreamMessage` values into route-neutral events such as:

* audio chunk available,
* remote warning received,
* final metadata received,
* transport failed,
* remote status updated,
* request cancelled,
* session expired.

11.2.3. The adapter shall be the only layer allowed to reason directly about whether `StreamSynthesis` is available or whether session fallback was required.

### 11.3 Playback boundary

11.3.1. The playback subsystem shall receive route-neutral PCM chunk data with explicit format metadata.

11.3.2. The playback subsystem shall not parse protobuf messages or inspect gRPC metadata.

11.3.3. The playback subsystem shall rely on the adapter/orchestration layers to declare terminal success, cancellation, expiration, or remote failure.

### 11.4 Diagnostics boundary

11.4.1. Diagnostics collection shall receive normalized transport events and correlation IDs from the adapter layer.

11.4.2. The diagnostics boundary shall preserve structured remote warning and error data without coupling display logic to raw gRPC exceptions.

---

## 12. Compatibility Rules for VoxPlayer

12.1. VoxPlayer shall support servers that lack `StreamSynthesis` by using the session baseline.

12.2. VoxPlayer shall support servers that return `UNIMPLEMENTED` for `GetGlobalSynthesisLimits` by degrading to a typed “limits unavailable” state.

12.3. VoxPlayer shall tolerate deployments where consumer-token omission is either enforced or relaxed, but it shall always treat token mismatch as an authorization failure.

12.4. VoxPlayer shall tolerate deployments where Windows-only consumer enforcement and consumer-platform-hint enforcement are disabled.

12.5. VoxPlayer shall support deployments that require auth metadata such as bearer authorization through gRPC metadata or interceptors, but the exact auth scheme remains endpoint-configuration-specific rather than globally defined by this specification.

---

## 13. Constrained Future Extension Point: VoxThink Routing

13.1. VoxPlayer shall preserve a route abstraction that can support a future upstream route before VoxSpeak.

13.2. No current VoxThink request, response, protobuf, session, cancellation, or streaming contract is verified by the repository materials used for this document.

13.3. Therefore, for this revision, the VoxThink route is constrained to the following extension-point assumptions only:

* VoxPlayer may select a route named for an upstream router.
* That route shall terminate in a route adapter that is independent from the VoxSpeak adapter.
* The future route shall map into the same player-internal route-neutral request, stream, cancellation, and diagnostics contracts.
* VoxPlayer shall not assume VoxThink reuses VoxSpeak `SynthesisRequest`, `StreamMessage`, `consumer_token`, `session_id`, or transport metadata semantics unchanged.

13.4. All VoxThink-specific transport behavior remains **TBD** until a dedicated VoxThink specification and verified implementation exist.

---

## 14. Verified Details vs. TBD Assumptions

### 14.1 Verified in repository

The following are directly verified by VoxSpeak protobuf, implementation, and/or tests:

* gRPC over HTTP/2 is the only approved remote transport baseline.
* Session-based synthesis is the required interoperability contract.
* `StreamSynthesis` is optional and falls back to session flow in the shipped client.
* Raw PCM plus explicit `AudioFormat` is the wire-level audio baseline.
* Session subscription success terminates with `FinalMetadata`.
* `consumer_token` is returned by session creation and used for subscription authorization.
* Role, queue priority, and platform identity are transport metadata concerns.
* Queue priority uses bounded classes `0..2` with default `1`.
* API-version policy is enforced by request header and advertised through initial metadata.
* Pseudo-stream fallback is a supported, warning-signaled behavior.
* Native-stream-required failures produce explicit `INVALID_ARGUMENT` errors.
* Session states, cancellation, and expiration semantics are externally visible.

### 14.2 Verified ambiguity or discrepancy

14.2.1. **Direct-stream role semantics are slightly ambiguous across prose/tests versus implementation detail.**

* Transport prose frames `StreamSynthesis` as a consumer convenience RPC.
* Implementation reuses requester-gated session creation plus consumer-gated subscription.
* Some mocked unit coverage passes `consumer` role to `StreamSynthesis`, but the non-mocked implementation path is safest when supplied with a combined requester+consumer role.

14.2.2. **Subscription message ordering is more specific in code than in transport prose.**

* Prose emphasizes audio then terminal metadata.
* Implementation may emit warning messages after chunk emission and before final metadata.

14.2.3. **Current session TTL is implementation-specific.**

* The present implementation uses 120 seconds.
* VoxPlayer should rely on `expires_at`, not the constant.

### 14.3 TBD / not verified

The following remain intentionally undefined for this revision:

* any VoxThink wire contract,
* any non-gRPC VoxSpeak transport,
* compressed streaming codecs on the wire,
* transport-level resumable playback/session resubscription semantics after client restart,
* bidirectional streaming request updates,
* server-pushed discovery/cache invalidation subscriptions,
* any remote personality create/update/delete API for VoxPlayer.

---

## 15. Summary

15.1. The current VoxPlayer transport baseline is a route-neutral client architecture with a concrete direct-to-VoxSpeak adapter over gRPC/HTTP2.

15.2. The minimum interoperable behavior is session creation plus subscription, with optional use of direct stream when available.

15.3. VoxPlayer shall treat terminal success as `FinalMetadata`, not merely receipt of chunk data.

15.4. VoxPlayer shall preserve transport concerns—roles, queue priority, platform identity, auth metadata, API-version negotiation, session tokens, and gRPC errors—inside the integration adapter boundary.

15.5. VoxPlayer shall treat pseudo-stream fallback, queue admission failures, token enforcement, and version-policy metadata as part of the normal current interoperability contract.

15.6. Future VoxThink routing remains an extension point only and shall not be implemented by guessing that VoxThink shares VoxSpeak transport semantics.

---

## 16. Key Verified Interfaces

16.1. Service: `voxspeak.v1.VoxSpeakService`

16.2. Required baseline RPCs:

* `CreateSynthesisSession`
* `SubscribeSynthesisSession`
* `CancelSynthesisSession`
* `GetSessionStatus`

16.3. Optional convenience RPC:

* `StreamSynthesis`

16.4. Key protobuf message types:

* `RequestHeader`
* `ResponseHeader`
* `SynthesisRequest`
* `CreateSynthesisSessionRequest`
* `CreateSynthesisSessionResponse`
* `SubscribeSynthesisSessionRequest`
* `StreamMessage`
* `AudioChunk`
* `AudioFormat`
* `FinalMetadata`
* `SynthesisMetadata`

16.5. Key transport metadata keys:

* role: `x-voxspeak-role`, `voxspeak-role`, `role`
* priority: `x-voxspeak-priority`, `voxspeak-priority`, `priority`
* consumer platform: `x-voxspeak-platform`, `voxspeak-platform`, `platform`
* requester platform: `x-voxspeak-requester-platform`, `voxspeak-requester-platform`, `requester-platform`
* API version policy publication: `x-voxspeak-api-default-version`, `x-voxspeak-api-supported-versions`, `x-voxspeak-api-policy-mode`, `x-voxspeak-api-version-policy`

---

## 17. Ambiguities and Review Flags

17.1. Confirm during VoxPlayer implementation whether the chosen client adapter should always send combined requester/consumer role metadata on direct streaming calls, even in deployments that presently omit role metadata.

17.2. Confirm whether VoxPlayer should surface individual pseudo-stream reason codes in the user-visible diagnostics timeline or collapse them into a higher-level fallback message while preserving the raw codes in detailed diagnostics.

17.3. Confirm whether VoxPlayer will expose `GetGlobalSynthesisLimits` and `ValidateSynthesis` in the initial product scope or keep them adapter-internal for diagnostics/preflight only.

17.4. Confirm the desired local policy for storing remote original/processed text in history when privacy/redaction settings are enabled.

---

End of Document
