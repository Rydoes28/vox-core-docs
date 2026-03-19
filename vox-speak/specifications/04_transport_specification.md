# VoxCore – VoxSpeak TTS Subsystem

## Transport and Wire Protocol Specification, Version 1.5 (Approved)

**Document ID:** VoxSpeak-TRANSPORT-v1.5
**Derived from:** VoxSpeak-FR-v1.5, VoxSpeak-ARCH-v1.5, VoxSpeak-API-v1.5
**Project:** VoxCore
**Subsystem:** VoxSpeak (`vox-speak`)
**Status:** Approved

**Revision note:** VoxSpeak-TRANSPORT-v1.5 aligns the transport contract with metadata-based role signaling, required global-limits publication semantics, machine-readable pseudo-stream fallback warnings, and deploy-time authorization policy controls.

---

## 1. Purpose and Scope

1.1. This document specifies how VoxSpeak APIs shall be exposed and consumed across process and network boundaries.

1.2. It defines:

* transport protocol selection
* remote request and streaming semantics
* wire-level audio representation
* cancellation and backpressure behavior
* API-version compatibility signaling
* role, platform, and authorization policy behavior
* operational limit publication

1.3. This document maps VoxSpeak-API-v1.5 concepts onto network transports.

---

## 2. Deployment Topology Baseline

2.1. VoxSpeak shall support deployment as a service on Ubuntu-class hosts.

2.2. VoxThink or equivalent requester services may run separately from VoxSpeak and submit synthesis requests remotely.

2.3. Consumer clients may be distinct from requester clients.

2.4. Windows consumers shall be supported.

2.5. Windows-only consumption shall be a deploy-time enforcement option rather than an unconditional baseline requirement.

---

## 3. Transport Protocol Selection

3.1. VoxSpeak-TRANSPORT-v1.5 shall standardize on gRPC over HTTP/2.

3.2. Other transports shall not be considered conformant unless approved by a later transport revision.

---

## 4. Roles, Metadata, and Interaction Patterns

4.1. The transport shall distinguish two actor roles:

* **Requester**: submits synthesis requests and may query status or cancel sessions.
* **Consumer**: receives streamed audio and terminal metadata.

4.2. The transport shall support split-role workflows in which requester and consumer are different clients.

4.3. Role signaling for role-gated operations shall be carried in transport metadata rather than in the protobuf request body.

4.4. Conformant servers shall recognize requester or consumer role metadata using the approved metadata key set:

* `x-voxspeak-role`
* `voxspeak-role`
* `role`

4.5. Role metadata values shall be normalized case-insensitively and shall support requester-capable values and consumer-capable values.

4.6. Queue-priority input for remote calls shall be carried in transport metadata using the approved key set:

* `x-voxspeak-priority`
* `voxspeak-priority`
* `priority`

4.7. Queue-priority values shall be interpreted as bounded integer classes in the inclusive range `0..2`.

4.8. When queue-priority metadata is omitted or invalid, the server shall apply default priority class `1`.

4.9. Consumer-platform metadata may be carried using approved transport metadata keys, and requester-platform metadata may be carried independently for auditing or policy evaluation.

4.10. Session-creation requests may include a consumer-platform hint in the request body for downstream subscription policy.

---

## 5. Wire-Level Audio Representation

5.1. The default wire format shall be raw PCM frames with explicit metadata for sample rate, channel count, and sample format.

5.2. Each emitted chunk shall include:

* chunk index
* payload bytes
* end-of-stream flag

5.3. Wire-compressed audio codecs shall remain out of scope for this baseline.

---

## 6. RPC Model

### 6.1. Introspection and Control Plane

6.1.1. The transport baseline shall include these unary RPCs:

* `ListPersonalities`
* `DescribePersonality`
* `ListEngines`
* `EngineCapabilities`
* `GetGlobalSynthesisLimits`
* `ValidateSynthesis`
* `LintPersonalities`
* `ReloadPersonalities`

6.1.2. `GetGlobalSynthesisLimits` shall publish server-global admission and planning limits.

6.1.3. Servers that implement the approved baseline shall provide `GetGlobalSynthesisLimits`.

6.1.4. Clients communicating with older servers that return `UNIMPLEMENTED` for `GetGlobalSynthesisLimits` shall treat that outcome as a typed limits-unavailable compatibility state.

### 6.2. Session-Based Synthesis

6.2.1. `CreateSynthesisSession` shall create a session and return an opaque `session_id`.

6.2.2. `CreateSynthesisSessionResponse` shall return:

* `session_id`
* `expires_at`
* `consumer_token`
* optional early metadata when available

6.2.3. `SubscribeSynthesisSession` shall return a stream of `StreamMessage` values for the identified session.

6.2.4. The stream shall emit zero or more `AudioChunk` messages followed by exactly one terminal `FinalMetadata` message, unless the stream terminates with a transport error.

6.2.5. Requester clients shall be able to invoke `CancelSynthesisSession` and `GetSessionStatus`.

### 6.3. Direct Streaming Convenience

6.3.1. A server may provide `StreamSynthesis` for consumers that are both requester and consumer.

6.3.2. When provided, `StreamSynthesis` shall be semantically equivalent to creating a session and immediately subscribing to it.

6.3.3. `StreamSynthesis` shall require streaming output semantics and shall reject file-output-only request shapes.

---

## 7. Streaming Semantics

7.1. Chunk emission shall respect gRPC backpressure.

7.2. `REALTIME` mode shall emit paced chunks.

7.3. `FAST_AS_POSSIBLE` mode shall emit chunks as quickly as practical.

7.4. Terminal metadata shall mark stream completion.

7.5. Pseudo-stream fallback may be used whenever native streaming cannot be used or when the effective processing path requires buffering before chunk emission.

7.6. Pseudo-stream fallback shall be signaled using in-band `StreamWarning` messages and duplicated warning codes in terminal metadata.

7.7. Pseudo-stream fallback signaling shall include the general warning code `PSEUDO_STREAMING_BUFFER_THEN_CHUNK`.

7.8. Pseudo-stream fallback signaling shall also include reason-specific warning codes when applicable, including:

* `PSEUDO_STREAMING_ENGINE_CAPABILITY_FALLBACK`
* `PSEUDO_STREAMING_RUNTIME_UNAVAILABLE_FALLBACK`
* `PSEUDO_STREAMING_SERVER_FALLBACK`
* `PSEUDO_STREAMING_FILE_OUTPUT_FALLBACK`
* `PSEUDO_STREAMING_POSTPROCESSING_FALLBACK`

7.9. When a requested stream mode requires native streaming and the server cannot satisfy that requirement, the call shall fail with a structured argument or validation error instead of silently using pseudo-streaming.

---

## 8. Cancellation, Session Lifetime, and Status

8.1. Client-side subscription cancellation shall cancel the active subscription.

8.2. Requester-initiated cancellation shall terminate synthesis on a best-effort basis.

8.3. Sessions shall have a time-to-live and shall expire when that lifetime is exceeded.

8.4. `GetSessionStatus` shall report at least session state and best-effort progress percentage.

8.5. When final metadata is available, `GetSessionStatus` may return it.

---

## 9. Admission Control and Published Limits

9.1. Admission control shall apply explicit queue-priority semantics.

9.2. Overload rejections shall map to `RESOURCE_EXHAUSTED` and shall include structured reason details when available.

9.3. The published global-limits contract shall include, at minimum:

* maximum synthesis duration in milliseconds
* global maximum concurrent jobs
* global maximum queued jobs
* per-engine maximum concurrent jobs
* per-engine maximum queued jobs

9.4. Published limit values shall represent effective server policy values for the connected server instance.

9.5. A maximum synthesis duration value of `0` shall mean that no explicit duration cap is published.

---

## 10. API-Version Compatibility Signaling

10.1. Requests may include `header.api_version`.

10.2. When `header.api_version` is omitted, the server shall apply its default wire version.

10.3. Unsupported request versions shall be rejected with `INVALID_ARGUMENT` and structured details identifying the requested version, the supported versions, and the default version.

10.4. Version-policy advertisement shall be published through gRPC initial metadata.

10.5. Conformant servers shall publish, at minimum, the following metadata keys when initial metadata is available:

* `x-voxspeak-api-default-version`
* `x-voxspeak-api-supported-versions`
* `x-voxspeak-api-policy-mode`
* `x-voxspeak-api-version-policy`

10.6. When applicable, servers shall also publish:

* `x-voxspeak-api-effective-version`
* `x-voxspeak-api-requested-version`

10.7. `EngineCapabilities` responses may include an advisory copy of API-version policy metadata inside the flexible capability payload.

---

## 11. Error Mapping

11.1. VoxSpeak domain errors shall map to gRPC status codes and may include structured details.

11.2. Cancellation shall map to `CANCELLED`.

11.3. Unknown sessions shall map to `NOT_FOUND`.

11.4. Role, token, and platform authorization failures shall map to `PERMISSION_DENIED`.

11.5. Invalid request shapes or unsupported version requests shall map to `INVALID_ARGUMENT` unless a more specific status is required.

---

## 12. Security and Authorization Requirements

12.1. TLS shall be supported.

12.2. Authentication may be layered through interceptors, including token- or mTLS-based approaches.

12.3. Session subscription authorization shall use a session-scoped `consumer_token`.

12.4. Missing `consumer_token` input shall be rejected by default for subscription calls.

12.5. Deployments may permit subscription without a token as an explicit policy override.

12.6. When a token is provided and does not match the session token, the subscription shall be rejected.

12.7. Consumer-platform enforcement shall be optional and policy-driven.

12.8. When consumer-platform enforcement is enabled for a session and the subscriber platform is absent or mismatched, the subscription shall be rejected.

---

## 13. Operational Considerations

13.1. Transport configuration shall support connection, unary-request, and stream-idle timeouts.

13.2. Structured logging shall include request identifiers and session identifiers when available.

13.3. Metrics shall include request counts, error counts, latency, active sessions, stream durations, and admission outcomes by queue-priority class.

---

End of Document
