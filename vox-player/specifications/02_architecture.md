# VoxCore – VoxPlayer Windows Client

## Architecture Specification, Version 0.1 (Draft)

**Document ID:** VoxPlayer-ARCH-v0.1  
**Derived from:** VoxPlayer-DI-00; VoxPlayer-FR-v0.1; VoxSpeak-FR-v1.5; VoxSpeak-ARCH-v1.5; VoxSpeak-API-v1.5; VoxSpeak-TRANSPORT-v1.5; VoxSpeak-PROTO-v1.2; verified `vox-speak/` implementation  
**Project:** VoxCore  
**Subsystem:** VoxPlayer (`docs/vox-player`)  
**Status:** Draft

**Revision note:** VoxPlayer-ARCH-v0.1 defines the initial implementation-oriented architecture for a Windows desktop player that integrates with VoxSpeak as it exists in the repository today. The architecture establishes a stable direct-to-VoxSpeak baseline, preserves a clean route abstraction for a future upstream router such as VoxThink, and avoids assuming that a future VoxThink contract will match VoxSpeak transport semantics.

---

## 1. Purpose and Architectural Goals

1.1. This document defines a concrete architecture for VoxPlayer as a single-user Windows desktop application that captures text, submits synthesis requests to a Linux-hosted VoxSpeak service, receives streamed PCM audio, and renders playback locally.

1.2. Primary architectural goals:

* keep the current direct-to-VoxSpeak path implementation-ready,
* isolate VoxSpeak protobuf/gRPC concerns behind adapters,
* keep future route expansion possible without contaminating core player models,
* provide reliable queueing, cancellation, interruption, and playback behavior,
* support diagnosable operation in real desktop environments,
* respect Windows-specific constraints without overcommitting to unverified UI or audio frameworks.

1.3. The architecture shall treat the current VoxSpeak gRPC-over-HTTP/2 contract as the only verified remote synthesis baseline for initial implementation.

1.4. The architecture shall treat future VoxThink-mediated flows as a routing scenario, not as evidence that VoxPlayer may reuse VoxSpeak request or stream types unchanged.

---

## 2. System Context

2.1. VoxPlayer is a Windows desktop client with local responsibilities that VoxSpeak does not provide:

* text capture and submission UX,
* local request queue management,
* local audio buffering and playback device management,
* endpoint/profile selection,
* user-facing diagnostics and request history.

2.2. VoxSpeak remains the current verified remote synthesis/control endpoint. VoxPlayer consumes it as an external service.

2.3. A future VoxThink route is considered an application-level upstream routing target. For this revision, VoxPlayer only requires that route selection be pluggable; it does not require a defined VoxThink transport protocol.

2.4. Present-day distributed topology assumptions:

* VoxPlayer runs on Windows.
* VoxSpeak runs on a Linux distribution, currently Ubuntu 24.04 LTS.
* requester and consumer semantics may collapse into one VoxPlayer process, but the transport contract still distinguishes them.
* deployment policy may require transport metadata for role, platform, queue priority, and version negotiation.

---

## 3. Architectural Overview

3.1. VoxPlayer shall be decomposed into these major subsystems:

* **UI / Presentation subsystem**
* **Application / Orchestration subsystem**
* **Transport / Integration adapters**
* **Playback / Audio subsystem**
* **Persistence / Configuration subsystem**
* **Diagnostics / Telemetry subsystem**

3.2. Dependency direction shall flow inward from UI and external integrations toward application-level policies and internal models.

3.3. The architecture shall separate **VoxPlayer internal models** from **VoxSpeak transport-generated models** so that:

* presentation logic does not depend on protobuf-generated types,
* route-specific transport details do not leak into queue or playback policies,
* a future route can map onto the same application orchestration contracts even if its wire format differs.

3.4. High-level dependency structure:

```text
UI / Presentation
        |
Application / Orchestration
   |        |         |         \
   |        |         |          \
Playback  Persistence  Diagnostics  Route Abstractions
                                 |
                    Transport / Integration Adapters
                                 |
                    VoxSpeak gRPC / future VoxThink adapter
```

---

## 4. Subsystems and Responsibilities

### 4.1. UI / Presentation Subsystem

4.1.1. The UI / presentation subsystem is responsible for:

* text-entry views,
* clipboard-send and screen-capture initiation surfaces,
* route selection,
* endpoint/profile selection,
* request history views,
* playback controls,
* user-visible error and warning presentation,
* diagnostics inspection surfaces.

4.1.2. The UI shall bind only to VoxPlayer view models and application DTOs.

4.1.3. The UI shall not directly construct protobuf request messages, read gRPC metadata, or interpret raw VoxSpeak `StreamMessage` envelopes.

4.1.4. UI-visible concepts shall use user-facing semantics such as “Awaiting audio”, “Receiving audio”, “Playing”, or “Server requires Windows consumer identity”, rather than raw transport-only field names.

### 4.2. Application / Orchestration Subsystem

4.2.1. The application / orchestration subsystem is the architectural center of VoxPlayer.

4.2.2. It is responsible for:

* converting user intent into request workflows,
* validating local submission prerequisites,
* selecting a route for each request,
* coordinating remote submission, stream consumption, buffering, playback, cancellation, and retry policy,
* maintaining submission, connection, and playback state machines,
* preserving queue order and interruption semantics,
* emitting domain events for UI updates and diagnostics.

4.2.3. This subsystem shall own player-local concepts including:

* `SubmissionRecord`,
* `RouteSelection`,
* `EndpointProfile`,
* `ConnectionState`,
* `PlaybackItem`,
* `PlaybackBufferStatus`,
* `RequestFailure`,
* `RequestDiagnosticsSnapshot`.

4.2.4. This subsystem shall depend on abstract route and playback interfaces rather than concrete VoxSpeak or audio-framework implementations.

### 4.3. Transport / Integration Adapters

4.3.1. The transport / integration layer shall encapsulate all remote-contract-specific behavior.

4.3.2. For the direct route, this layer shall provide a **VoxSpeak route adapter** responsible for:

* mapping VoxPlayer request models to VoxSpeak gRPC/protobuf requests,
* applying transport metadata such as role, priority, platform, and optional auth headers,
* selecting `StreamSynthesis` when available and falling back to session flow when absent or `UNIMPLEMENTED`,
* translating gRPC stream envelopes into route-neutral stream events,
* exposing cancellation and status inspection through a route-neutral interface,
* translating gRPC / protobuf / transport failures into VoxPlayer integration errors.

4.3.3. A future **VoxThink route adapter** shall implement the same route-facing orchestration interface, but this architecture does not assume it will use VoxSpeak protobuf DTOs or the same session/stream contract.

4.3.4. The transport layer shall be the only layer that knows about:

* generated `voxspeak_pb2` / `voxspeak_pb2_grpc` types,
* gRPC metadata keys,
* gRPC status codes,
* session `consumer_token`,
* transport-level API version policy advertisement.

### 4.4. Playback / Audio Subsystem

4.4.1. The playback / audio subsystem is responsible for:

* enumerating Windows output devices,
* selecting a target device or following the system default,
* validating and buffering PCM audio,
* converting route-neutral audio events into the audio engine’s ingestion format,
* starting, pausing, resuming, stopping, draining, and skipping playback,
* isolating playback-thread timing from UI-thread timing.

4.4.2. The playback subsystem shall accept an internal audio buffer contract, not protobuf `AudioChunk` objects.

4.4.3. The playback subsystem shall treat explicit format metadata as authoritative and reject incompatible format transitions within one playback item.

4.4.4. The playback subsystem shall support both:

* streamed incremental playback from arriving PCM buffers, and
* already-buffered playback when remote delivery is pseudo-streamed or otherwise delayed.

### 4.5. Persistence / Configuration Subsystem

4.5.1. The persistence / configuration subsystem is responsible for:

* endpoint profiles,
* saved route defaults,
* playback preferences,
* hotkey settings,
* capture-region settings,
* privacy/redaction settings,
* history retention,
* optional last-known discovery cache for personalities/engines.

4.5.2. This subsystem shall distinguish:

* **configuration state**: durable user/application preferences,
* **history state**: request records and diagnostics,
* **ephemeral runtime state**: in-flight buffers, tokens, transient connection state.

4.5.3. Session `consumer_token` values shall be treated as ephemeral request-scoped runtime data and shall not be persisted beyond the lifecycle required for in-flight recovery or diagnostics policy.

### 4.6. Diagnostics / Telemetry Subsystem

4.6.1. The diagnostics / telemetry subsystem is responsible for:

* structured request lifecycle logging,
* correlation of local request IDs with remote request/session IDs when available,
* recording transport warnings and failures,
* recording buffering and playback events,
* providing a user-facing diagnostic timeline per request,
* supporting redaction-aware persistence and display.

4.6.2. It shall consume route-neutral diagnostic events emitted by orchestration and adapter layers.

4.6.3. It may include optional developer-facing detail views that show remote warning codes, gRPC status, negotiated endpoint, resolved route, and retry attempts.

---

## 5. Core Internal Contracts and Boundaries

### 5.1. Internal Models

5.1.1. VoxPlayer shall define route-neutral internal models that represent user workflows rather than a specific remote protocol.

5.1.2. Recommended internal model groups:

* **Submission models**
  * `SubmissionDraft`
  * `SubmissionRequest`
  * `SubmissionSource` (`typed`, `clipboard`, `screen_capture_dynamic`, `screen_capture_fixed`)
  * `RouteSelection`
* **Queue and request models**
  * `QueueItem`
  * `SubmissionRecord`
  * `RequestExecutionHandle`
* **Audio models**
  * `PlayerAudioFormat`
  * `PlayerAudioChunk`
  * `BufferedAudioSegment`
* **State models**
  * `SubmissionState`
  * `ConnectionState`
  * `PlaybackState`
* **Configuration models**
  * `EndpointProfile`
  * `RouteProfile`
  * `PlaybackDevicePreference`
* **Diagnostics models**
  * `RequestWarning`
  * `RequestFailure`
  * `RequestDiagnosticEvent`
  * `RemoteContractSnapshot`

5.1.3. These models shall remain independent from protobuf-generated classes so they can survive transport changes.

### 5.2. VoxSpeak Boundary

5.2.1. The direct VoxSpeak adapter shall define an anti-corruption boundary between VoxPlayer and VoxSpeak.

5.2.2. Adapters/mappers are explicitly required for:

* `SubmissionRequest` -> VoxSpeak `SynthesisRequest`
* endpoint/profile settings -> gRPC client/channel configuration
* route metadata policy -> gRPC metadata headers
* VoxSpeak `StreamMessage` -> route-neutral `RemoteStreamEvent`
* VoxSpeak `AudioFormat` -> `PlayerAudioFormat`
* VoxSpeak `AudioChunk` -> `PlayerAudioChunk`
* VoxSpeak `FinalMetadata` / `SynthesisMetadata` -> internal completion and diagnostics models
* gRPC status / VoxSpeak error payloads -> `RequestFailure`

5.2.3. No other VoxPlayer layer shall depend directly on generated gRPC/protobuf types.

5.2.4. The adapter shall also normalize differences between the two verified VoxSpeak access patterns:

* **direct streaming convenience** via `StreamSynthesis`, and
* **session-oriented flow** via `CreateSynthesisSession` + `SubscribeSynthesisSession`.

### 5.3. Route Provider Interface

5.3.1. The application layer shall depend on a route-neutral provider contract, conceptually similar to:

* submit a text request,
* open or receive a remote audio event stream,
* cancel an in-flight request,
* optionally inspect remote status,
* expose route diagnostics.

5.3.2. The application layer shall not assume that every route implementation supports the same primitives in the same way.

5.3.3. Specifically, the future VoxThink route shall be allowed to:

* have a different session concept,
* have different retry semantics,
* deliver audio indirectly,
* produce a different warning/error taxonomy,
* hide or replace VoxSpeak-level identifiers.

5.3.4. The route-neutral provider contract should therefore express **capabilities** rather than hard-coding VoxSpeak semantics into the application core.

---

## 6. Direct-to-VoxSpeak Integration Architecture

### 6.1. Direct Route Responsibilities

6.1.1. The direct route shall use VoxSpeak’s current gRPC-over-HTTP/2 transport as the only verified direct integration path.

6.1.2. The direct adapter shall support both verified VoxSpeak remote patterns:

* prefer `StreamSynthesis` when the server supports it,
* fall back to `CreateSynthesisSession` followed by `SubscribeSynthesisSession` when `StreamSynthesis` is unavailable or returns `UNIMPLEMENTED`.

6.1.3. Because session-based synthesis is the required interoperability baseline, all orchestration behavior shall be designed so that the session path is sufficient for correctness.

### 6.2. Transport Metadata Policy

6.2.1. The adapter shall apply transport metadata, not protobuf body fields, for:

* requester / consumer role signaling,
* queue priority signaling,
* consumer and requester platform metadata,
* optional auth headers,
* transport-level version-policy interactions.

6.2.2. The architecture shall treat these metadata concerns as endpoint/profile policy, not end-user text-entry fields.

6.2.3. The adapter shall be able to set requester-capable metadata on requester RPCs and consumer-capable metadata on subscription/streaming RPCs.

6.2.4. For the direct-stream convenience path, the adapter shall still behave as if requester and consumer semantics exist, even if a single call hides the session creation from the user.

### 6.3. Session and Token Handling

6.3.1. When using session flow, the adapter shall preserve the returned `session_id`, `consumer_token`, and canonical remote request ID as part of an internal execution handle.

6.3.2. The `consumer_token` shall be supplied only where VoxSpeak requires it: subscription requests.

6.3.3. The application layer shall treat `consumer_token` as opaque and adapter-owned.

6.3.4. The adapter shall be prepared for deployments that reject missing tokens by default, but it shall not rely on relaxed policy behavior.

### 6.4. Audio and Completion Semantics

6.4.1. The adapter shall translate VoxSpeak `StreamMessage` envelopes into route-neutral events:

* `AudioAvailable(chunk)`
* `WarningRaised(code, message, details)`
* `Completed(final_metadata)`
* `Failed(error)`

6.4.2. Successful completion shall be anchored to terminal `FinalMetadata`, not solely to `AudioChunk.is_last`.

6.4.3. The playback pipeline may use `is_last` only as a buffering hint, not as the authoritative completion signal.

6.4.4. The adapter shall preserve warning codes associated with pseudo-streaming fallback and expose them to orchestration and diagnostics.

### 6.5. Discovery and Preflight

6.5.1. Optional discovery/preflight features shall be routed through the direct adapter for:

* `ListPersonalities`
* `DescribePersonality`
* `ListEngines`
* `EngineCapabilities`
* `ValidateSynthesis`
* `GetGlobalSynthesisLimits`

6.5.2. The adapter shall degrade gracefully when a connected older server returns `UNIMPLEMENTED` for `GetGlobalSynthesisLimits`.

6.5.3. Runtime metrics, lint, and reload RPCs may be useful for diagnostics or developer tools, but they are not required for the baseline end-user playback path.

---

## 7. Future Via-VoxThink Architecture

7.1. VoxPlayer shall model “via VoxThink” as a separate route provider behind the same application-level route selection mechanism.

7.2. This architecture intentionally does **not** assume that VoxThink will:

* expose VoxSpeak protobuf messages,
* expose session `consumer_token` semantics,
* use the same `StreamMessage` envelope shape,
* preserve VoxSpeak request IDs or warning codes,
* expose a direct playback stream at all.

7.3. Therefore, VoxPlayer shall not push VoxSpeak-specific types or lifecycle assumptions into the route-neutral application core.

7.4. The future route shall be allowed to implement one of several broad patterns, including:

* forwarding text to VoxThink and receiving a stream directly from VoxThink,
* forwarding text to VoxThink and letting VoxThink orchestrate VoxSpeak while VoxPlayer still receives audio from VoxThink,
* forwarding text to VoxThink and receiving a routed endpoint/session description that results in a subsequent audio stream from VoxSpeak.

7.5. For this revision, the only architectural commitment is that route selection, request history, diagnostics, and playback state are expressed using VoxPlayer-native models so the future route can plug in without redesigning the entire client.

---

## 8. State Models

### 8.1. Submission State Model

8.1.1. Each request shall have a submission lifecycle independent from playback.

8.1.2. Recommended submission states:

* `draft`
* `validating_local`
* `queued_local`
* `submitting_remote`
* `awaiting_remote_acceptance`
* `accepted_remote`
* `awaiting_remote_stream`
* `receiving_remote_audio`
* `terminal_completed`
* `terminal_cancelled`
* `terminal_failed`

8.1.3. State notes:

* `queued_local` means not yet actively submitted because another item owns the active slot.
* `accepted_remote` means the route accepted the request, but audio has not yet been observed.
* `receiving_remote_audio` begins on first accepted audio event.
* terminal submission success requires remote finalization, not merely local playback start.

### 8.2. Connection State Model

8.2.1. Connection state is endpoint-scoped, not request-scoped, though requests may observe it.

8.2.2. Recommended connection states:

* `uninitialized`
* `resolving_profile`
* `connecting`
* `ready`
* `degraded`
* `retry_wait`
* `disconnected`
* `authentication_failed`
* `version_incompatible`

8.2.3. The application shall distinguish endpoint connectivity problems from request validation failures.

8.2.4. The direct adapter may infer `degraded` when:

* control-plane compatibility is reduced but streaming remains usable,
* direct stream RPC is unavailable but session fallback still works,
* limits/discovery surfaces are unavailable while baseline synthesis remains usable.

### 8.3. Playback State Model

8.3.1. Playback state is local-device oriented and may lag behind submission state.

8.3.2. Recommended playback states:

* `idle`
* `buffering`
* `ready_to_play`
* `playing`
* `paused`
* `stopping`
* `draining`
* `completed`
* `cancelled`
* `failed`
* `device_unavailable`

8.3.3. State notes:

* `buffering` means insufficient local audio to begin or continue smooth playback.
* `draining` means remote audio is complete and remaining buffered audio is still being rendered.
* `completed` requires both remote success and local drain completion.

### 8.4. Composite Request View

8.4.1. A user-visible queue item should expose a composite summary derived from the three state models.

8.4.2. Example mappings:

* submission `awaiting_remote_stream` + playback `idle` -> “Waiting for audio”
* submission `receiving_remote_audio` + playback `buffering` -> “Receiving audio”
* submission `receiving_remote_audio` + playback `playing` -> “Playing”
* submission `terminal_completed` + playback `draining` -> “Finishing playback”
* submission `terminal_failed` -> “Failed”

---

## 9. Queueing, Buffering, Cancellation, and Interruption

### 9.1. Player-Local Queueing Model

9.1.1. VoxPlayer shall maintain a player-local queue that exists independently from any remote queue.

9.1.2. The local queue is responsible for:

* preserving submission order,
* allowing future reordering/replacement policy,
* preventing UI actions from directly coupling to transport concurrency,
* serializing playback decisions even if remote services support more concurrency.

9.1.3. Initial implementation guidance:

* one active playback item,
* zero or one actively connecting/submitting item per chosen policy,
* remaining items retained as local queued items.

9.1.4. The architecture may later support speculative prefetch or parallel remote submission, but the baseline shall not require it.

### 9.2. Buffering Model

9.2.1. The playback subsystem shall use at least two conceptual buffers:

* **ingest buffer**: receives route-neutral audio chunks,
* **render buffer**: audio engine-consumable queue for device playback.

9.2.2. Buffering policy shall be configurable enough to support:

* low-latency real-time playback attempts,
* more conservative startup buffering for unstable connections,
* pseudo-streamed responses that arrive after significant remote buffering.

9.2.3. Buffer thresholds shall remain a local playback concern and shall not alter the remote wire contract.

### 9.3. Cancellation Model

9.3.1. VoxPlayer shall distinguish these actions:

* **pause playback**: local only,
* **stop playback**: local rendering stop, may or may not imply remote cancel based on user action or policy,
* **cancel request**: explicit remote cancellation intent,
* **skip item**: local queue/navigation action that may trigger stop and optionally remote cancel.

9.3.2. Remote cancellation is route-specific and shall be implemented by the active route provider.

9.3.3. For the direct VoxSpeak route:

* if session flow is active, cancellation should call `CancelSynthesisSession` where still meaningful,
* if direct-stream convenience is active, the adapter should still expose a unified cancellation operation and translate it to the best supported behavior, recognizing that the server internally maps direct-stream cancellation to session cancellation behavior.

9.3.4. Cancellation results shall be modeled as best-effort and observable. A request can move through states such as “cancelling” and then “cancelled” or “failed to cancel”.

### 9.4. Interruption Model

9.4.1. Interruption means a higher-priority local UX event supersedes the active item, such as:

* user manually plays another item,
* user triggers a “speak clipboard now” hotkey while current playback is active,
* output device failure forces local playback stop.

9.4.2. The orchestration subsystem shall own interruption policy.

9.4.3. Recommended baseline policy:

* explicit user interruption stops local playback immediately,
* if the interrupted request is still remotely active, the user action or policy determines whether remote cancellation is issued,
* interrupted items may either return to queued state or move to cancelled/failed state depending on the action semantics.

---

## 10. Endpoint and Profile Configuration Model

### 10.1. Endpoint Profiles

10.1.1. VoxPlayer shall persist endpoint profiles as first-class configuration entities.

10.1.2. An endpoint profile should include, where applicable:

* profile name,
* route kind (`direct_voxspeak`, future `via_voxthink`, etc.),
* network endpoint / authority,
* TLS mode and certificate settings,
* optional auth header or token reference,
* API-version preference,
* requester role metadata policy,
* consumer role metadata policy,
* queue priority class,
* consumer-platform identity policy,
* discovery/preflight behavior toggles,
* retry policy settings,
* diagnostic verbosity preferences.

10.1.3. The configuration model shall not require that all fields apply to all route kinds.

### 10.2. Route Profiles vs Endpoint Profiles

10.2.1. The architecture shall distinguish:

* **route profiles**: user-intent selection such as direct-to-VoxSpeak vs future via-VoxThink,
* **endpoint profiles**: concrete connection and transport policy settings for one route implementation.

10.2.2. This separation allows multiple environments per route, such as local development, LAN server, staging, or production.

### 10.3. Playback and Capture Preferences

10.3.1. Playback preferences shall be persisted separately from endpoint profiles so audio-device selection changes do not mutate remote integration settings.

10.3.2. Capture and hotkey preferences shall be persisted separately from route profiles so alternative input workflows can share the same route definitions.

---

## 11. Error Propagation and Observability Design

### 11.1. Error Taxonomy Layers

11.1.1. Errors should be normalized across these layers:

* **local validation errors**
* **configuration/profile errors**
* **route integration errors**
* **transport compatibility/authorization errors**
* **remote synthesis errors**
* **playback/device errors**
* **history/persistence errors**

11.1.2. The application layer shall preserve both:

* a user-facing message, and
* structured machine-readable details for diagnostics.

### 11.2. Direct VoxSpeak Error Mapping

11.2.1. The direct adapter shall translate gRPC/protobuf outcomes into route-neutral categories such as:

* `not_found`
* `permission_denied`
* `cancelled`
* `invalid_request`
* `resource_exhausted`
* `version_incompatible`
* `transport_unavailable`
* `deadline_or_timeout`
* `contract_violation`
* `remote_internal_failure`

11.2.2. The adapter shall preserve original details where helpful, including:

* gRPC status code,
* remote request ID,
* remote session ID,
* warning codes,
* whether `StreamSynthesis` fell back to session flow,
* whether the failure occurred during create, subscribe, stream, cancel, or status phases.

### 11.3. Observability Events

11.3.1. The architecture should emit structured events such as:

* `submission.created`
* `submission.validated`
* `route.selected`
* `remote.submit.started`
* `remote.submit.accepted`
* `remote.audio.first_chunk`
* `remote.warning`
* `remote.completed`
* `remote.cancel.requested`
* `remote.cancel.confirmed`
* `playback.buffering_started`
* `playback.started`
* `playback.paused`
* `playback.device_failed`
* `request.failed`

11.3.2. These events should carry correlation fields such as:

* local request ID,
* route kind,
* endpoint profile ID,
* remote request ID when available,
* remote session ID when available,
* playback device ID when relevant.

### 11.4. User-Facing Diagnostics

11.4.1. VoxPlayer should provide a per-request diagnostic timeline.

11.4.2. That timeline should be able to show, when available:

* chosen route,
* endpoint profile,
* selected personality/engine,
* whether direct stream or session fallback was used,
* whether pseudo-streaming fallback occurred,
* warning codes returned by VoxSpeak,
* whether the failure was local, transport-level, remote, or playback-local.

11.4.3. Privacy settings shall determine whether full submitted text, redacted text, or a hash/reference is retained in history and diagnostics.

---

## 12. Windows-Specific Architectural Considerations

12.1. The architecture shall assume a Windows desktop environment for the client and a Linux-hosted VoxSpeak service environment for the remote synthesis system, but shall not bind this specification to an unverified UI toolkit, OCR library, hotkey library, audio backend, or Linux service-packaging mechanism.

12.2. Windows-specific concerns the architecture must accommodate include:

* global hotkeys that may fire while another application has focus,
* clipboard reads without UI focus transfer,
* screen-region capture across multi-monitor and mixed-DPI environments,
* output-device enumeration and default-device tracking,
* device-loss or device-switch events during playback,
* thread-affinity constraints common in desktop UI frameworks.

12.3. Architectural implications:

* UI-thread work and audio/render work shall be separated.
* screen-coordinate normalization shall be abstracted behind a capture service so the core application does not encode monitor-scaling logic.
* output-device handling shall be abstracted behind a playback-device provider so the rest of the client does not depend on one Windows audio API choice.
* the application shall assume transient device disappearance is possible and recoverable.

12.3.1. Linux-hosted VoxSpeak implications:

* VoxPlayer shall treat VoxSpeak as a remote network service, not as a co-located Windows library dependency.
* endpoint/profile configuration shall assume cross-OS deployment concerns such as TLS, certificates, hostnames, timeouts, and remote service availability.
* VoxPlayer shall not assume access to Linux-local filesystem artifacts, device resources, or process controls on the VoxSpeak host.
* diagnostic messaging should make the cross-OS boundary visible where useful, for example by distinguishing Windows playback/device failures from Linux-hosted VoxSpeak transport or synthesis failures.

12.4. This specification intentionally does not commit to WinUI, WPF, Qt, WASAPI, XAudio2, NAudio, or any other specific implementation technology.

---

## 13. Sequence Flows

### 13.1. Direct text -> VoxSpeak -> audio -> VoxPlayer -> playback

```text
User
  -> UI: enter text and press Submit
UI
  -> Application: create SubmissionRequest(route=DirectToVoxSpeak)
Application
  -> Validation: validate local inputs/profile/device
Validation
  -> Application: valid
Application
  -> Queue Manager: enqueue item / activate item
Queue Manager
  -> Direct VoxSpeak Adapter: submit stream request
Direct VoxSpeak Adapter
  -> VoxSpeak: StreamSynthesis(request, metadata)
VoxSpeak
  -> Direct VoxSpeak Adapter: stream messages
[If StreamSynthesis unavailable]
Direct VoxSpeak Adapter
  -> VoxSpeak: CreateSynthesisSession
VoxSpeak
  -> Direct VoxSpeak Adapter: session_id + consumer_token
Direct VoxSpeak Adapter
  -> VoxSpeak: SubscribeSynthesisSession(session_id, consumer_token)
VoxSpeak
  -> Direct VoxSpeak Adapter: StreamWarning? + AudioChunk* + FinalMetadata
Direct VoxSpeak Adapter
  -> Application: RemoteStreamEvent(s)
Application
  -> Playback Buffer: append PCM chunks
Playback Buffer
  -> Audio Engine: render to selected Windows output device
Audio Engine
  -> Application: playback started / drained / completed
Application
  -> UI: update states, history, diagnostics
```

### 13.2. Future text -> VoxThink -> VoxSpeak -> audio -> VoxPlayer -> playback

```text
User
  -> UI: enter text and select route=ViaVoxThink
UI
  -> Application: create SubmissionRequest(route=ViaVoxThink)
Application
  -> VoxThink Route Adapter: submit route-neutral request
VoxThink Route Adapter
  -> VoxThink: route-specific submit call (TBD contract)
VoxThink
  -> VoxSpeak: internal/orchestrated synthesis call (implementation outside VoxPlayer scope)
VoxThink and/or VoxSpeak
  -> VoxThink Route Adapter: route-specific audio/completion events
VoxThink Route Adapter
  -> Application: route-neutral RemoteStreamEvent(s)
Application
  -> Playback Buffer: append PCM/audio frames
Playback Buffer
  -> Audio Engine: render to Windows output device
Application
  -> UI: update route-neutral submission/playback state
```

**Architectural note:** the only guaranteed boundary here is between the application layer and a route adapter that emits route-neutral events. The internal VoxThink-to-VoxSpeak protocol remains unspecified and is intentionally not modeled in detail.

### 13.3. Cancellation / interruption flow

```text
User
  -> UI: click Cancel or trigger interrupting action
UI
  -> Application: cancel active item
Application
  -> Playback Subsystem: stop or pause local rendering immediately per policy
Application
  -> Active Route Adapter: request remote cancellation
[Direct VoxSpeak session path]
Active Route Adapter
  -> VoxSpeak: CancelSynthesisSession(session_id)
VoxSpeak
  -> Active Route Adapter: cancelled acknowledgement / terminal stream cancellation
Active Route Adapter
  -> Application: cancellation outcome event
Application
  -> Queue Manager: move item to cancelled or interrupted terminal state
Application
  -> UI: update controls/history/diagnostics
```

### 13.4. Connection failure / retry flow

```text
Application
  -> Route Adapter: open connection / submit request
Route Adapter
  -> Endpoint: connection attempt fails (UNAVAILABLE, timeout, TLS mismatch, etc.)
Route Adapter
  -> Application: integration failure with category + retryability
Application
  -> Connection State Store: set degraded or retry_wait
Application
  -> Diagnostics: record failure with endpoint/profile context
[If retry policy allows]
Application
  -> Retry Scheduler: wait backoff interval
Retry Scheduler
  -> Route Adapter: retry submit/connect
[If retry succeeds]
Route Adapter
  -> Application: ready / accepted_remote
[If retry exhausts]
Application
  -> Queue/Request State: terminal_failed
Application
  -> UI: surface exact failure class and retry result
```

---

## 14. Testability and Mock/Fake Integration Strategy

### 14.1. Architectural Testability Goals

14.1.1. The architecture shall make most VoxPlayer behavior testable without a real Windows UI shell or a real VoxSpeak server.

14.1.2. The core application/orchestration layer shall be testable against abstractions for:

* route adapters,
* playback engine,
* clock/timers,
* persistence store,
* diagnostics sink.

### 14.2. Fake Route Providers

14.2.1. VoxPlayer shall support fake route providers that emit route-neutral events such as:

* immediate success with buffered audio,
* streaming chunk-by-chunk success,
* pseudo-streaming delayed first audio,
* warning-bearing success,
* cancellation mid-stream,
* transport failure before first audio,
* contract violation such as format mismatch.

14.2.2. These fakes shall let tests validate queueing, UI state transitions, buffering policy, and error handling without gRPC dependencies.

### 14.3. VoxSpeak-Specific Integration Tests

14.3.1. The direct VoxSpeak adapter should additionally be tested against:

* mocked gRPC stubs for mapper and error translation logic,
* repository-provided live-service black-box tests where available,
* compatibility cases such as `StreamSynthesis` -> session fallback, global-limits `UNIMPLEMENTED`, token enforcement, and pseudo-stream warnings.

14.3.2. This split preserves a clean boundary:

* most player tests operate on route-neutral fakes,
* a smaller set of adapter integration tests validate the real VoxSpeak contract.

### 14.4. Playback Fakes

14.4.1. The playback subsystem shall expose a fake audio sink that records:

* chunks received,
* playback start/stop/pause calls,
* drain completion,
* simulated device failures.

14.4.2. This allows deterministic tests for interruption, device loss, and completion semantics without requiring real Windows audio output.

---

## 15. Architectural Risks and Tradeoffs

### 15.1. Route Neutrality vs Simplicity

15.1.1. Keeping VoxSpeak details behind an anti-corruption layer adds design overhead now.

15.1.2. The tradeoff is justified because the requirements explicitly include a future upstream route whose transport contract is not yet verified.

### 15.2. Local Queue vs Remote Queue Awareness

15.2.1. A strong local queue simplifies UX and playback control.

15.2.2. The tradeoff is that local queued state may not match remote admission/queue state exactly.

15.2.3. The architecture therefore treats local queue state and remote session state as related but distinct concepts.

### 15.3. Completion Anchored to Final Metadata

15.3.1. Waiting for terminal metadata before declaring success aligns with verified VoxSpeak semantics.

15.3.2. The tradeoff is that local playback may finish audibly before remote success can be confirmed.

15.3.3. The architecture resolves this by distinguishing remote completion from local drain completion and only showing full success when both are satisfied.

### 15.4. Windows Device Robustness vs Latency

15.4.1. Conservative buffering and device-loss handling improve robustness.

15.4.2. The tradeoff is higher startup latency for some playback scenarios.

15.4.3. The architecture therefore keeps buffering policy configurable and local.

### 15.5. Transport Metadata in Profiles

15.5.1. Storing route metadata policy in endpoint profiles matches the verified VoxSpeak deployment model.

15.5.2. The tradeoff is that advanced settings become more complex for users.

15.5.3. The architecture should therefore allow simple UI defaults with an advanced profile editor rather than surfacing raw metadata concepts everywhere.

---

## 16. Summary of Key Architectural Decisions

16.1. VoxPlayer shall use a layered architecture that cleanly separates UI, orchestration, transport adapters, playback, persistence, and diagnostics.

16.2. VoxSpeak integration shall be isolated behind a direct route adapter that owns all gRPC/protobuf concerns and explicitly maps to/from VoxPlayer-native models.

16.3. Session-based VoxSpeak synthesis is the required correctness baseline; `StreamSynthesis` is an optimization that the adapter may use opportunistically.

16.4. Successful completion shall require both VoxSpeak terminal metadata and local playback drain completion.

16.5. Queueing, buffering, cancellation, and interruption shall be modeled as player-local concerns coordinated with route-specific remote operations.

16.6. Endpoint/profile configuration shall store transport and policy settings separately from playback and capture preferences.

16.7. Future via-VoxThink support shall be introduced as a separate route adapter without assuming VoxThink shares VoxSpeak’s transport contract.

---

## 17. Most Important Integration Boundaries

17.1. **UI <-> Application boundary:** UI binds to view models and commands only; no protobuf or gRPC types cross this boundary.

17.2. **Application <-> Route Adapter boundary:** route-neutral requests, stream events, cancellation, status, and diagnostics only.

17.3. **Route Adapter <-> VoxSpeak boundary:** all generated protobuf types, gRPC metadata, session tokens, and gRPC status mapping remain here.

17.4. **Application <-> Playback boundary:** internal PCM/audio buffer models only; playback never consumes protobuf chunk types directly.

17.5. **Application <-> Persistence/Diagnostics boundary:** durable settings and history are stored in VoxPlayer-native schemas with redaction-aware policy.

---

## 18. Remaining Open Design Points

18.1. Exact Windows implementation technologies for UI, hotkeys, OCR/text capture, and audio output remain intentionally open.

18.2. The precise local queue concurrency policy beyond the initial single-active-playback baseline remains open.

18.3. The shape of discovery caching and offline personality/engine presentation remains open.

18.4. The detailed retry policy matrix by error class and route remains open.

18.5. Any concrete via-VoxThink submission, streaming, cancellation, or status protocol remains open until a verified VoxThink contract exists.

---

End of Document
