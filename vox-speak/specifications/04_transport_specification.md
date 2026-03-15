# VoxCore – VoxSpeak TTS Subsystem

## Transport and Wire Protocol Specification, Version 1.3 (Approved)

**Document ID:** VoxSpeak-TRANSPORT-v1.3
**Derived from:** VoxSpeak-FR-v1.4, VoxSpeak-ARCH-v1.4, VoxSpeak-API-v1.4
**Project:** VoxCore
**Subsystem:** VoxSpeak (`vox-speak`)
**Status:** Approved

**Revision note:** VoxSpeak-TRANSPORT-v1.3 clarifies control-plane RPCs, version defaulting/advertisement, role/platform enforcement semantics, and queue/admission behavior.

---

## 1. Purpose and Scope

1.1. This document specifies how VoxSpeak APIs are exposed and consumed across process and network boundaries.

1.2. It defines:

* Transport protocols
* Message and streaming semantics
* Wire-level audio representation
* Cancellation and backpressure behavior
* Version and capability negotiation
* Security and operational requirements

1.3. This document maps VoxSpeak-API-v1.4 concepts onto network transports.

---

## 2. Deployment Topology (Authoritative)

2.1. VoxSpeak runs as a service on Ubuntu.

2.2. VoxThink runs on Ubuntu and may submit synthesis requests to VoxSpeak.

2.3. Windows clients are the sole consumers of VoxSpeak audio streams.

2.4. VoxSpeak shall accept text requests originating from:

* VoxThink (Ubuntu), and/or
* Windows clients (direct).

2.5. VoxSpeak shall stream/transport generated audio to Windows clients.

---

## 3. Transport Protocol Selection

3.1. VoxSpeak-TRANSPORT-v1.3 standardizes on **gRPC over HTTP/2**.

---

## 4. Roles and Interaction Patterns

4.1. The transport distinguishes two roles:

* **Requester**: submits a synthesis request (VoxThink or Windows).
* **Consumer**: receives audio and metadata (Windows only).

4.2. Because VoxThink is never a consumer, the transport shall support a split-role workflow where requester and consumer are different clients.

4.3. Role signaling shall be explicit in requests for role-gated operations.

4.4. Consumer platform hints may be provided by requesters and shall be validated or enforced according to configured server policy.

---

## 5. Wire-Level Audio Representation

5.1. Default wire format is raw PCM frames with explicit metadata:

* sample rate
* channels
* sample format (e.g., S16LE)

5.2. Each chunk includes:

* chunk index
* payload bytes
* end-of-stream flag

5.3. Audio compression codecs are out of scope for v1.3.

---

## 6. RPC Model

### 6.1. Introspection

6.1.1. `ListPersonalities`, `DescribePersonality`, `ListEngines`, `EngineCapabilities`.

### 6.2. Session-Based Synthesis (Required)

6.2.1. `CreateSynthesisSession(request) -> CreateSynthesisSessionResponse`

6.2.2. `CreateSynthesisSessionResponse` returns:

* `session_id` (opaque)
* `expires_at`
* optional `consumer_token` (if used for authorization)

6.2.3. `SubscribeSynthesisSession(session_id, consumer_token?) -> stream StreamMessage`

6.2.4. The stream emits:

* zero or more `AudioChunk`
* exactly one terminal `FinalMetadata`

6.2.5. The requester may optionally call:

* `CancelSynthesisSession(session_id)`
* `GetSessionStatus(session_id)`

### 6.3. Convenience: Direct Streaming (Optional)

6.3.1. For cases where the Windows client is both requester and consumer, a direct streaming RPC may be provided:

* `StreamSynthesis(request) -> stream StreamMessage`

6.3.2. If provided, it shall be semantically equivalent to creating a session and immediately subscribing.


### 6.4. Control-Plane Operations (Required)

6.4.1. `ValidateSynthesis(request) -> ValidateSynthesisResponse` shall return preflight diagnostics and resolved configuration preview without emitting audio.

6.4.2. `LintPersonalities(request) -> LintPersonalitiesResponse` shall return structured diagnostics for loaded personality definitions.

6.4.3. `ReloadPersonalities(request) -> ReloadPersonalitiesResponse` shall return reload outcome status and diagnostics.

---

## 7. Streaming Semantics

7.1. Chunk emission respects gRPC backpressure.

7.2. `StreamMode` behavior:

* REALTIME: paced emission
* FAST_AS_POSSIBLE: emit ASAP

7.3. Terminal metadata marks completion.

7.4. Pseudo-streaming (buffer-then-chunk) may be used only when native streaming is unavailable and shall be flagged in metadata warnings.

---

## 8. Cancellation and Lifetime

8.1. Client-side stream cancellation cancels the subscription.

8.2. Requester-initiated cancellation (CancelSynthesisSession) shall terminate synthesis best-effort.

8.3. Sessions shall have a TTL; expiration terminates synthesis/subscription and releases resources.

8.4. Request admission shall apply explicit priority semantics; when priority is omitted, the documented default priority shall be applied.

---

## 9. Versioning and Capability Negotiation

9.1. Requests may include `api_version` (e.g., `1.4`).

9.1.1. If `api_version` is omitted, the server shall apply its default API version and report the effective version in response metadata.

9.2. Server publishes:

* supported versions and policy mode
* default and effective API version metadata
* supported audio formats
* exposed operational limits defined by wire contracts (for example max text length, concurrency)

---

## 10. Error Mapping

10.1. VoxSpeak errors map to gRPC status codes and include structured details.

10.2. Cancellation maps to `CANCELLED`.

10.3. Overload and admission-control rejections shall map to structured resource/capacity errors and shall include retry guidance when available.

---

## 11. Security Requirements

11.1. TLS supported.

11.2. Authentication supported via interceptors (token, mTLS).

11.3. Session subscription authorization may use a scoped `consumer_token` returned at session creation.

---

## 12. Operational Considerations

12.1. Configurable timeouts: connection, request processing, subscription idle.

12.2. Structured logging includes request_id and session_id.

12.3. Metrics shall include request/error counts, latency, active sessions, stream durations, and admission outcomes by priority class.

---

End of Document
