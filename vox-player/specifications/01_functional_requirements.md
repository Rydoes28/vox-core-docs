# VoxCore – VoxPlayer Windows Client

## Functional Requirements, Version 0.1 (Draft)

**Document ID:** VoxPlayer-FR-v0.1  
**Project:** VoxCore  
**Subsystem:** VoxPlayer (`docs/vox-player`)  
**Status:** Draft  
**Derived from:** VoxPlayer-DI-00; VoxSpeak-FR-v1.5; VoxSpeak-ARCH-v1.5; VoxSpeak-API-v1.5; VoxSpeak-TRANSPORT-v1.5; VoxSpeak-PROTO-v1.2  
**Change policy:** Breaking requirement changes require VoxPlayer-FR-v1; non-breaking clarifications may be issued as VoxPlayer-FR-v0.x.

**Revision note:** VoxPlayer-FR-v0.1 defines the initial functional baseline for the Windows playback client, grounded in the currently verified VoxSpeak transport, protobuf, and implementation constraints. Future upstream routing through VoxThink remains explicitly assumption-constrained.

---

## 1. Scope and Objectives

1.1. The system shall provide a Windows desktop application that accepts user text input, submits that text for speech synthesis, and plays returned audio to a locally selected output device.

1.2. The system shall support a present-day direct integration path from VoxPlayer to VoxSpeak using the currently verified gRPC-over-HTTP/2 contract. Trace: DI §1.1-§1.3, §2.1, §3.1-§3.4.

1.3. The system shall provide a route-selection abstraction that supports both:

* direct submission to VoxSpeak, and
* a future optional upstream text-routing stage before VoxSpeak.

1.4. The system shall expose user-visible playback, routing, connection, status, and diagnostics behavior without requiring users to understand transport-only implementation details.

1.5. The system shall define present-day requirements only for verified VoxSpeak interactions and shall mark future upstream-routing behavior as TBD or assumption-constrained where a contract does not yet exist. Trace: DI §6.

1.6. The system shall remain implementable as a single-user Windows desktop client and shall not require headless or multi-tenant deployment semantics.

---

## 2. Functional Baseline and Terminology

2.1. For this version, the mandatory remote synthesis baseline shall be the VoxSpeak session-oriented gRPC flow consisting of:

1. `CreateSynthesisSession`
2. `SubscribeSynthesisSession`
3. optional `CancelSynthesisSession`
4. optional `GetSessionStatus`

Trace: DI §1.1, §2.1, §3.1, §5.

2.2. VoxPlayer may use `StreamSynthesis` when the connected VoxSpeak server implements it, but VoxPlayer shall not require that RPC to exist for interoperable operation. Trace: DI §1.1, §2.1.2, §3.4, §4.1.

2.3. VoxPlayer shall treat streamed `AudioChunk` messages and terminal `FinalMetadata` as the authoritative success-path output contract for remote synthesis playback. Trace: DI §1.1, §2.1.3, §2.3.15-§2.3.17.

2.4. VoxPlayer shall treat role metadata, queue priority metadata, platform metadata, session authorization tokens, and API-version negotiation as transport integration concerns rather than as end-user text-entry fields. Trace: DI §2.1.4-§2.1.6, §2.2, §2.4.18-§2.4.21, §7.

2.5. VoxPlayer shall treat future upstream routing as an application-level route choice, not as a modification of the verified VoxSpeak contract.

2.6. Unless a requirement is explicitly marked as future-facing, the requirement shall apply to the initial direct-to-VoxSpeak design baseline.

2.7. Requirements marked **[VOXSPEAK UPDATE MAY BE REQUIRED]** identify behaviors whose full implementation may require a future VoxSpeak contract, API, or approved management-workflow change beyond the currently verified baseline.

---

## 3. Present-Day User Text Submission Requirements

3.1. VoxPlayer shall provide a user-editable text-entry surface for synthesis input.

3.2. VoxPlayer shall allow a user to submit a text request from the current text-entry surface without requiring file-based intermediate artifacts.

3.3. VoxPlayer shall support a user action that submits the current textual content of the Windows clipboard directly for synthesis without first requiring the user to paste that content into the text-entry surface.

3.4. Clipboard-send shall be invocable while another Windows application has focus and shall not require the user to switch focus to VoxPlayer before submission.

3.5. When clipboard-send is used, VoxPlayer shall capture the clipboard text snapshot used for the request at the time of submission and shall treat later clipboard changes as irrelevant to that in-flight request.

3.6. VoxPlayer shall support a screen-region text-capture input mode that allows the user either to:

* dynamically mark a region of the current desktop for text extraction, or
* select a pre-defined fixed capture region.

3.7. VoxPlayer shall provide two separately configurable hotkeys for screen-region text capture: one for dynamic region marking and one for invoking capture from a pre-defined fixed region.

3.8. Screen-region capture shall be invocable while another Windows application has focus and shall not require the user to switch focus to VoxPlayer before initiating capture.

3.9. After either screen-region hotkey is triggered, VoxPlayer shall not require additional interaction with the VoxPlayer application to complete capture and create the submission input.

3.10. Screen-region text capture shall support Windows multi-monitor environments, including monitors with differing resolutions and scaling characteristics, without assuming a single-display coordinate space.

3.11. VoxPlayer shall treat the text-extraction mechanism for screen-region capture as implementation-TBD, but the functional submission contract shall require that any extracted text be reviewable or directly submittable as request input according to the selected user workflow.

3.12. Each submission shall produce a player-visible request record containing, at minimum:

* a locally unique request/session record identifier,
* submission timestamp,
* selected route,
* submitted text or a redacted representation when local privacy settings require redaction,
* input-source type,
* connection target identity,
* terminal playback/request state when known.

3.13. For direct-to-VoxSpeak submission, VoxPlayer shall construct a `SynthesisRequest` compatible with the current VoxSpeak wire contract and shall populate only fields that are supported by the selected user workflow. Trace: DI §3.2, §3.5.1.

3.14. VoxPlayer shall not invent remote request-body fields for role, platform, queue priority, or consumer authorization token signaling. Those concerns shall remain in transport metadata or the session subscription request where the VoxSpeak contract requires them. Trace: DI §2.1.5-§2.1.6, §2.2.7, §5.

3.15. VoxPlayer shall support selecting stream-shaped request output for live playback. Trace: DI §2.3.11-§2.3.12.

3.16. VoxPlayer shall validate locally available required inputs before submission, including at minimum:

* non-empty text,
* a valid route selection,
* a resolvable remote endpoint configuration for routes that require VoxSpeak,
* a playable local output-device selection or an explicit no-playback test mode if such a mode is later introduced.

3.17. When a required local input is missing, VoxPlayer shall reject the submission before remote transport invocation and shall present a user-visible validation error.

3.18. For clipboard-send or screen-region capture submission, VoxPlayer shall reject the submission locally when no textual content can be obtained from the selected source.

3.19. When VoxPlayer exposes optional personality or engine selection, it shall submit those values using the verified VoxSpeak request fields rather than local-only aliases. Trace: DI §5, “Select personalities / inspect engines”.

3.20. VoxPlayer shall preserve the submitted text record long enough to support in-app request history and diagnostics, subject to user-configurable local retention settings.

---

## 4. Route Selection and Upstream Routing Requirements

4.1. VoxPlayer shall expose a route-selection state for each submission.

4.2. The initial supported route set shall include **Direct to VoxSpeak**.

4.3. VoxPlayer shall support configuration of a future alternate route representing **Upstream router before VoxSpeak**.

4.4. When the direct route is selected, VoxPlayer shall send user text to VoxSpeak according to the current verified VoxSpeak transport contract. Trace: DI §1-§3, §5.

4.5. When the future upstream-router route is selected, VoxPlayer shall preserve the same user-observable submission lifecycle states defined for direct routing unless and until a future approved specification defines route-specific differences.

4.6. VoxPlayer shall not require the future upstream-router route for basic operation.

4.7. Any requirement whose behavior depends on an undefined VoxThink request, response, session, or streaming contract shall be marked TBD / assumption-constrained and shall not be treated as an initial implementation blocker for the direct route. Trace: DI §6.

4.8. VoxPlayer shall persist the user’s default route preference independently from per-request route overrides.

4.9. VoxPlayer shall record, in request history and diagnostics, which route was selected for each submission.

---

## 5. Direct VoxSpeak Integration Requirements

5.1. VoxPlayer shall integrate with VoxSpeak over gRPC over HTTP/2. Trace: DI §2.1.1.

5.2. VoxPlayer shall support the VoxSpeak session-based synthesis flow as the mandatory interoperability baseline. Trace: DI §2.1.2, §3.1.

5.3. VoxPlayer may use `StreamSynthesis` as an optimization or convenience path when supported by the connected server, but shall provide a working fallback to `CreateSynthesisSession` plus `SubscribeSynthesisSession` when `StreamSynthesis` is absent or returns `UNIMPLEMENTED`. Trace: DI §1.1, §3.4.

5.4. VoxPlayer shall carry requester- and consumer-capable role metadata when invoking role-gated VoxSpeak operations. Trace: DI §2.1.4-§2.1.5.

5.5. VoxPlayer shall support queue-priority metadata configuration only within the bounded integer-class semantics verified by VoxSpeak, and shall tolerate server defaulting when no explicit priority is configured. Trace: DI §2.1.6.

5.6. VoxPlayer shall preserve and use the `consumer_token` returned by `CreateSynthesisSessionResponse` when subscribing to a session stream. Trace: DI §2.2.7, §3.1.

5.7. VoxPlayer shall tolerate deployments in which missing consumer tokens are either rejected or relaxed according to server policy, but it shall treat a supplied-token mismatch as an authorization failure. Trace: DI §2.2.8.

5.8. VoxPlayer shall provide a configurable consumer-platform identity hint when deployment policy requires Windows-only or platform-matching consumption, and shall not assume such policy is universally enabled. Trace: DI §2.2.9-§2.2.10.

5.9. VoxPlayer shall support optional retrieval of VoxSpeak personality and engine discovery data through the verified control-plane RPCs when that data is needed for user selection or diagnostics. Trace: DI §1.2, §5.

5.10. VoxPlayer should support `ValidateSynthesis` and `GetGlobalSynthesisLimits` as preflight surfaces when connection policy or UX design requires them, but the player shall degrade gracefully when an older server returns `UNIMPLEMENTED` for global-limits publication. Trace: DI §2.4.20-§2.4.21.

5.11. VoxPlayer shall not require operator-focused RPCs such as runtime metrics, lint, or reload for basic end-user playback behavior. Trace: DI §1.2, §3.5.2.

5.12. VoxPlayer shall support VoxSpeak request-header API-version signaling and shall treat gRPC initial metadata as the authoritative source for published version-policy advertisement. Trace: DI §2.4.18-§2.4.19, §3.3.

5.13. When VoxPlayer exposes personality tuning or editing, it shall load the selected personality into a non-persistent editable working copy so the user can iteratively adjust it without immediately modifying the stored personality definition. **[VOXSPEAK UPDATE MAY BE REQUIRED]** Trace: DI §1.2, §5 "Select personalities / inspect engines".

5.14. VoxPlayer shall not persistently modify or overwrite a stored personality unless the user performs an explicit save, apply, overwrite, or equivalent persist action. **[VOXSPEAK UPDATE MAY BE REQUIRED]**

5.15. VoxPlayer shall distinguish clearly between a stored personality and an unsaved working copy in the user workflow and diagnostics.

5.16. VoxPlayer shall support duplicating an existing personality into a new editable working copy so the duplicated personality can be used as a template when creating a new personality. **[VOXSPEAK UPDATE MAY BE REQUIRED]**

5.17. Any workflow that creates or persistently saves a personality shall not assume the existence of a current VoxSpeak create/update personality RPC and shall therefore use only an explicitly approved management workflow for persistence. **[VOXSPEAK UPDATE MAY BE REQUIRED]** Trace: DI §1.2, §3.5.2, §4.3.

---

## 6. Playback Queue and Control Requirements

6.1. VoxPlayer shall maintain a player-local playback queue abstraction for submitted requests.

6.2. VoxPlayer shall support, at minimum, the following user-visible request/playback states:

* draft
* submitting
* queued locally
* awaiting remote stream
* receiving audio
* playing
* completed
* cancelled
* failed

6.3. VoxPlayer shall preserve request ordering in the local playback queue according to the order in which the user submits requests, unless the user explicitly reorders or replaces queued items.

6.4. VoxPlayer shall support these playback controls for the active item:

* play or resume when audio is available,
* pause local playback,
* stop playback,
* skip to the next queued item,
* cancel the active remote request when it is still in progress.

6.5. VoxPlayer shall distinguish **stop local playback** from **cancel remote synthesis** when the remote request is still active, and the UI shall make that distinction observable.

6.6. VoxPlayer shall not treat receipt of a chunk with `is_last=true` as sufficient evidence of successful completion unless terminal `FinalMetadata` is also received or an explicitly defined local fallback policy is invoked after transport failure. Trace: DI §2.3.17.

6.7. VoxPlayer shall not mark a request as successfully completed until all buffered audio intended for playback has been rendered locally and the corresponding terminal metadata has been processed.

6.8. VoxPlayer may support queue-clearing controls, but clearing queued items that have not yet been submitted remotely shall not require remote cancellation.

6.9. VoxPlayer shall retain enough local queue state to recover user-visible history after playback completes or fails.

---

## 7. Streaming and Non-Streaming Response Handling Requirements

7.1. For direct VoxSpeak integration, VoxPlayer shall support playback from streamed PCM `AudioChunk` messages with explicit `AudioFormat` metadata. Trace: DI §1.3, §2.3.15, §3.2.

7.2. VoxPlayer shall validate that successive audio chunks within a single playback stream remain compatible with the format expected by the playback pipeline; incompatible stream data shall be treated as a request failure or contract violation and shall not be silently mixed into playback. Trace: DI §2.3.16.

7.3. VoxPlayer shall consume and surface `StreamWarning` messages when present during streaming playback. Trace: DI §1.1, §2.3.14, §3.3.

7.4. VoxPlayer shall process terminal `FinalMetadata` as the authoritative completion payload for successful remote streaming requests. Trace: DI §1.1, §2.3.17.

7.5. VoxPlayer shall support both presently verified VoxSpeak streaming modes when exposed in the UI or configuration:

* real-time / paced streaming,
* fast-as-possible streaming.

Trace: DI §3.2, §5 “Stream audio for playback”.

7.6. VoxPlayer shall treat pseudo-streaming fallback as a normal interoperability case and shall not misclassify it as an unspecified server failure. Trace: DI §2.3.14, §3.3, §5.

7.7. When pseudo-streaming fallback occurs, VoxPlayer shall preserve the machine-readable warning codes in local diagnostics and should expose a user-visible indication that playback may begin only after server-side buffering. Trace: DI §2.3.14, §3.3.

7.8. When a requested streaming mode requires native streaming and VoxSpeak returns an argument/validation failure instead of degrading, VoxPlayer shall present a user-visible failure reason and shall not transparently reissue the request with weaker semantics unless an explicit user or policy setting authorizes that retry behavior. Trace: DI §2.3.13, §3.3.

7.9. VoxPlayer shall support a buffered playback path for already-complete audio data that becomes available locally, but the initial direct VoxSpeak remote contract shall not assume the existence of a non-streaming audio-return RPC. Trace: DI §4.1, §6.1, §6.5.

7.10. Any future non-streaming upstream-router response mode is TBD / assumption-constrained until that route’s contract is specified.

---

## 8. Device and Output Management Requirements

8.1. VoxPlayer shall enumerate available Windows audio output devices and shall expose selection of the active playback device.

8.2. VoxPlayer shall provide a default output-device mode that follows the Windows system default audio output.

8.3. VoxPlayer shall support switching the preferred playback device between requests.

8.4. When the active output device becomes unavailable during playback, VoxPlayer shall:

* stop or pause local playback safely,
* preserve the request record and diagnostic state,
* present a user-visible device error,
* allow the user to retry playback when a valid device becomes available.

8.5. VoxPlayer shall not require that the selected output device influence the `AudioFormat` negotiated with VoxSpeak, though the player may perform local format conversion as required by the Windows audio stack.

8.6. VoxPlayer shall support playback of the currently verified VoxSpeak wire-level PCM contract without requiring compressed transport codecs. Trace: DI §1.3, §2.3.15.

8.7. VoxPlayer may support local volume control independent of synthesis parameters.

8.8. VoxPlayer shall distinguish local device/rendering failures from remote synthesis failures in diagnostics and user-visible status.

---

## 9. Cancellation and Interruption Requirements

9.1. VoxPlayer shall support user-initiated cancellation of an active synthesis request.

9.2. When VoxPlayer owns the direct VoxSpeak session lifecycle, user cancellation of an in-progress request shall invoke `CancelSynthesisSession` on a best-effort basis when a session has already been created. Trace: DI §1.1, §2.3 “Cancel playback / request state”.

9.3. VoxPlayer shall also terminate or dispose of the active subscription stream when local cancellation occurs.

9.4. VoxPlayer shall support interruption by a newer submission according to a configurable policy with at least these choices:

* do not interrupt active playback,
* interrupt playback only,
* interrupt playback and cancel remote synthesis.

9.5. When interruption affects only local playback and not remote synthesis, VoxPlayer shall continue to track the remote request until a terminal state is known or the user explicitly discards it.

9.6. VoxPlayer shall surface `CANCELLED` outcomes distinctly from transport unavailability, validation failure, permission failure, and successful completion. Trace: DI §4.1.9, §5 “Cancel playback / request state”.

9.7. VoxPlayer should use `GetSessionStatus` when needed to reconcile uncertain terminal state after cancellation, transport loss, or UI restart, provided the session identifier remains available. Trace: DI §1.1, §3.1.

9.8. Any interruption semantics that require upstream-router conversational state or dialogue management are TBD / assumption-constrained until VoxThink behavior is specified. Trace: DI §6.7-§6.10.

---

## 10. Connection, Endpoint, and Compatibility Handling Requirements

10.1. VoxPlayer shall support configuration of one or more VoxSpeak service endpoints.

10.2. VoxPlayer shall allow the user or deployment policy to select an active endpoint.

10.3. VoxPlayer shall persist endpoint configuration separately from transient request history.

10.4. VoxPlayer shall expose connection status sufficient to distinguish at minimum:

* endpoint configured but not yet tested,
* reachable and compatible,
* unreachable,
* reachable but incompatible,
* reachable but authorization denied.

10.5. VoxPlayer shall support explicit connection testing without requiring a user to submit synthesis text.

10.6. When the server publishes API-version policy metadata, VoxPlayer shall record the effective and supported version information for diagnostics. Trace: DI §2.4.18-§2.4.19, §3.3.

10.7. When the server rejects a request because of unsupported API version, VoxPlayer shall present a compatibility error that includes the requested version, the server default version, and the supported versions when those values are available. Trace: DI §2.4.18-§2.4.19.

10.8. VoxPlayer shall treat `UNIMPLEMENTED` responses for optional compatibility surfaces such as `GetGlobalSynthesisLimits` or direct streaming convenience as compatibility states rather than generic transport corruption. Trace: DI §2.4.20, §3.4.

10.9. VoxPlayer should support transport security and authentication configuration, but the specific production authentication profile remains outside the verified current VoxPlayer baseline and shall be treated as deployment-specific until separately specified. Trace: DI §4.8.

---

## 11. Diagnostics, Telemetry, and User-Visible Error Requirements

11.1. VoxPlayer shall expose user-visible status for the current request lifecycle and playback lifecycle.

11.2. VoxPlayer shall retain structured diagnostics for each request, including at minimum:

* route,
* endpoint,
* submission time,
* local state transitions,
* remote session identifier when available,
* remote request identifier when available,
* received warning codes,
* terminal result classification,
* error code and message when failed.

11.3. VoxPlayer shall preserve VoxSpeak machine-readable warning codes received in-band or in terminal metadata without lossy remapping in diagnostic storage. Trace: DI §1.3, §2.3.14, §3.3.

11.4. VoxPlayer shall distinguish at least the following user-visible failure classes:

* local validation error,
* connection failure,
* remote compatibility/version failure,
* authorization failure,
* admission/overload failure,
* remote validation/argument failure,
* remote cancellation,
* remote unknown-session or expired-session failure,
* local playback-device failure,
* local decoding/rendering failure.

11.5. VoxPlayer shall preserve final synthesis metadata when available for diagnostics and post-request inspection. Trace: DI §1.3, §3.2.

11.6. VoxPlayer should provide an end-user-friendly message for each failure class while also retaining the original machine-readable transport or warning details for support workflows.

11.7. VoxPlayer may emit local telemetry for performance and support analysis, but such telemetry shall be disableable by configuration.

11.8. VoxPlayer shall not require operator-only VoxSpeak runtime metrics for basic user diagnostics, though it may optionally surface them in advanced troubleshooting views if later approved. Trace: DI §3.5.2.

---

## 12. Session, History, and Persistence Requirements

12.1. VoxPlayer shall maintain a local request history for the current user profile.

12.2. Each history record shall capture, at minimum:

* local request record identifier,
* submitted text or redacted equivalent,
* input-source type,
* selected route,
* endpoint,
* playback state transitions,
* remote session identifier when available,
* resolved personality and engine identifiers when available,
* warning summary,
* completion or failure outcome,
* timestamps for submission and terminal state.

12.3. VoxPlayer shall make history records available for user review within the application.

12.4. VoxPlayer shall support configurable local retention for history records.

12.5. Audio received from VoxSpeak shall not be required to persist automatically as a durable local file for the initial design.

12.6. If VoxPlayer later supports saving received audio locally, that behavior shall be optional and shall not be required for baseline playback.

12.7. VoxPlayer shall persist user configuration across application restarts, including at minimum:

* endpoint configuration,
* default route,
* selected or default playback device policy,
* preferred streaming mode when the UI exposes it,
* clipboard-send hotkey or invocation preference when such a preference is user-configurable,
* dynamic-region capture hotkey when configurable,
* fixed-region capture hotkey when configurable,
* pre-defined fixed screen capture regions when configured,
* diagnostics/telemetry preference,
* history-retention preference.

12.8. VoxPlayer shall keep configuration persistence logically separate from transient remote session state.

12.9. VoxPlayer shall recover persisted configuration on application start without automatically replaying prior requests.

12.10. If VoxPlayer supports personality editing, it may preserve unsaved working copies for the current user session or restoreable draft state, but such draft persistence shall remain logically separate from explicit persistent modification of the stored personality.

---

## 13. Windows Desktop Application Requirements

13.1. VoxPlayer shall operate as a Windows desktop application with a persistent windowed user interface.

13.2. The application shall remain responsive during network operations, active streaming, and playback.

13.3. The application shall support standard keyboard text entry for submission content.

13.4. The application shall support copy/paste of text into the submission surface.

13.5. The application shall present the currently active route, endpoint, playback status, and active input method within the main user workflow.

13.6. The application shall support background invocation of clipboard-send without requiring the VoxPlayer window to become focused.

13.7. The application shall support background invocation of screen-region capture without requiring the VoxPlayer window to become focused.

13.8. When screen-region capture is supported in the UI, the application shall provide a user workflow for both ad hoc region marking and selection of a pre-defined fixed capture region.

13.9. The application shall expose two separately configurable hotkeys for the two screen-region capture modes when those modes are enabled.

13.10. Screen-region capture workflows shall allow region definition across the full Windows desktop space presented by all attached monitors supported by the application runtime.

13.11. The application shall handle normal window close or application-exit actions by stopping local playback safely and releasing local audio resources.

13.12. On application exit, VoxPlayer should attempt best-effort cancellation or cleanup of active remote sessions that it owns, but failure to complete that cleanup shall be surfaced only diagnostically and shall not block application shutdown.

13.13. VoxPlayer shall support Windows consumer identity signaling needed for VoxSpeak deployments that enforce Windows-only or platform-matching subscription policy. Trace: DI §2.2.9-§2.2.10.

---

## 14. Future-Facing Extension Points and TBD / Assumption-Constrained Requirements

14.1. VoxPlayer shall preserve an application-layer extension point for routing user text through a future upstream module before VoxSpeak.

14.2. The following items are TBD / assumption-constrained and shall not be treated as verified present-day contracts:

* the VoxThink request/response format,
* whether VoxThink returns transformed text, audio, or both,
* whether VoxThink proxies, terminates, or replaces the VoxSpeak streaming/session model,
* whether VoxThink owns request IDs, session IDs, personalities, engine selection, or queue-priority policy,
* whether VoxThink introduces conversation-state or dialogue-management semantics,
* whether VoxThink changes interruption, retry, or history semantics.

Trace: DI §4, §6.

14.3. Until a future upstream-router specification exists, VoxPlayer shall model the upstream route as an abstract text-routing stage whose guaranteed present-day functional requirements are limited to:

* selectable routing state,
* persisted route preference,
* per-request route traceability,
* failure classification distinct from direct VoxSpeak route failures.

14.4. VoxPlayer may later add route-specific preflight, transformed-text review, or richer diagnostics for upstream routing only through a future approved revision.

---

## 15. Out of Scope for Initial VoxPlayer Design

15.1. The following items are out of scope for the initial VoxPlayer design baseline:

* defining a concrete VoxThink transport, API, protobuf, or IPC contract,
* conversational AI or dialogue-state management,
* mandatory local persistence of generated audio files,
* mandatory editing or authoring of VoxSpeak personalities,
* operator administration workflows such as personality reload or lint execution,
* mandatory use of runtime metrics dashboards,
* compressed audio transport codecs,
* multi-user profile synchronization across machines,
* cloud account management or identity provisioning UX,
* automated retry policies beyond explicit user- or policy-authorized retry behavior.

---

End of Document
