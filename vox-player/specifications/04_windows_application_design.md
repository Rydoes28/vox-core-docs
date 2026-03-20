# VoxCore – VoxPlayer Windows Client

## Windows Application Design Specification, Version 0.1 (Draft)

**Document ID:** VoxPlayer-WINAPP-v0.1  
**Derived from:** VoxPlayer-DI-00; VoxPlayer-FR-v0.1; VoxPlayer-ARCH-v0.1; VoxPlayer-TRANSPORT-v0.1; verified `vox-speak/` protobuf, service, transport-client, and test behavior  
**Project:** VoxCore  
**Subsystem:** VoxPlayer (`docs/vox-player`)  
**Status:** Draft

**Revision note:** VoxPlayer-WINAPP-v0.1 defines the Windows desktop application design for the initial VoxPlayer client. It is grounded in the currently verified direct-to-VoxSpeak integration model and intentionally avoids inventing additional transport or upstream-routing behavior.

---

## 1. Purpose and Scope

1.1. This document defines the intended Windows desktop application design for VoxPlayer as a single-user playback client.

1.2. This document covers:

* application screens and navigation,
* user workflows,
* controls and user-visible states,
* queue and playback UX,
* diagnostics and configuration surfaces,
* failure and recovery UX,
* accessibility expectations,
* MVP scope and phased expansion.

1.3. This document does **not** redefine the remote transport contract. The verified present-day integration baseline remains the VoxSpeak gRPC-over-HTTP/2 contract described in the design-input, functional, architecture, and transport specifications.

1.4. Where a design choice depends on remote behavior, the current VoxSpeak implementation and transport specification are the source of truth.

1.5. Future routing through an upstream system such as VoxThink remains an application-route extension point only. Any route-specific UX beyond route selection is TBD until a verified upstream contract exists.

---

## 2. Design Principles

2.1. The Windows application shall prioritize **fast submission, low-friction playback, and diagnosable failure handling** for the direct-to-VoxSpeak route.

2.2. The main workflow shall be optimized for these common user intents:

* type text and play it,
* send clipboard text without switching focus,
* capture on-screen text and play it,
* review the active queue and current request state,
* understand why playback is delayed, degraded, or failed.

2.3. The design shall expose user-facing concepts rather than transport-internal terminology wherever possible. For example, the application should present concepts such as **Connecting**, **Queued**, **Awaiting audio**, **Receiving audio**, **Playing**, **Cancelled**, or **Playback device unavailable** instead of raw RPC names.

2.4. The design shall preserve advanced inspection surfaces for support and developer workflows so machine-readable VoxSpeak details such as warning codes, gRPC status, session ID, canonical request ID, and API-version policy remain inspectable.

2.5. The design shall treat queue state, remote session state, and local playback state as related but distinct concerns, consistent with the architecture and verified transport model.

2.6. The design shall remain implementation-technology-neutral. A desktop MVVM-style presentation pattern is recommended because it aligns with the documented separation between UI, orchestration, playback, and transport adapters, but no specific Windows UI framework is required by this specification.

---

## 3. Verified Behavioral Baseline That Shapes the UI

3.1. The application design assumes the current direct VoxSpeak baseline:

1. create a synthesis session,
2. subscribe to the session stream,
3. receive zero or more audio chunks and optional warnings,
4. treat terminal `FinalMetadata` as authoritative successful remote completion,
5. optionally cancel or inspect status.

3.2. The design shall not require `StreamSynthesis` to exist, even if the transport adapter opportunistically uses it.

3.3. The UI shall not imply that audio playback success is confirmed solely by receipt of a final chunk flag, because the verified server behavior uses terminal metadata as the authoritative success terminator.

3.4. The UI shall treat pseudo-streaming fallback as a normal compatibility case, not as an undefined or exceptional transport mode.

3.5. The UI shall not expose role metadata, queue-priority metadata, consumer tokens, or platform metadata as free-form text-entry fields in the primary submission workflow. These are profile/policy concerns.

3.6. **Documented/implemented nuance carried into the UI design:** transport prose may describe `StreamSynthesis` as a consumer convenience path, but the implementation internally reuses session creation and subscription logic. Therefore, advanced diagnostics should not present direct-stream use as bypassing requester-side session semantics entirely.

---

## 4. Application Information Architecture

### 4.1. Top-level surfaces

4.1.1. The Windows application shall be organized around these primary surfaces:

* **Main Player window**
* **Queue and History panel**
* **Request Details / Diagnostics panel**
* **Settings window or settings view**
* **Connection / Endpoint test dialog**
* **Capture-region configuration surface**
* **Optional transient overlays/notifications** for background hotkey workflows and non-blocking status changes

4.1.2. The Main Player window shall be the default landing surface and the primary host for text submission, active playback, queue status, and common actions.

4.1.3. Diagnostics and configuration shall be accessible without crowding the primary playback workflow.

### 4.2. Recommended persistent layout

4.2.1. The default desktop layout should use three functional zones:

* **Submission pane**: text input, route/endpoint/personality choices, submit actions.
* **Active playback pane**: now playing, transport/playback state, main controls, buffering/warning indicators.
* **Queue/history pane**: pending items, recent items, item selection for details.

4.2.2. The diagnostics detail surface may appear as:

* a right-side inspector panel,
* a bottom expandable details drawer, or
* a separate dialog/window.

4.2.3. The design shall allow compact operation on smaller Windows laptop displays while remaining usable on larger multi-monitor desktops.

---

## 5. Screen and Surface Definitions

### 5.1. Main Player window

5.1.1. The Main Player window shall include, at minimum:

* application title and connection summary,
* text submission editor,
* active route selector,
* active endpoint selector,
* optional personality selector,
* optional engine selector,
* submit actions,
* playback controls for the active item,
* queue summary,
* current-state banner/indicator,
* access to diagnostics and settings.

5.1.2. The primary text editor shall support:

* multi-line text entry,
* paste,
* clear,
* direct submit,
* keyboard submission shortcut,
* visible empty-state guidance.

5.1.3. The main window shall expose the currently active input source for the active draft or request, such as:

* Typed text,
* Clipboard send,
* Screen capture (dynamic region),
* Screen capture (fixed region).

5.1.4. The main window shall display the active route as a user-facing choice. For MVP, the only fully defined route is **Direct to VoxSpeak**. A future **Via upstream router** route may be visible only if explicitly enabled and shall be marked TBD or unavailable until specified.

5.1.5. The main window shall display endpoint identity clearly enough for users to distinguish environments such as local, LAN, staging, or production.

### 5.2. Queue and History panel

5.2.1. The queue/history surface shall show request records in a list with concise status summaries.

5.2.2. Each item row should expose:

* local request identifier or short display token,
* submission timestamp,
* input source,
* route,
* endpoint/profile,
* condensed text preview or redacted placeholder,
* composite state summary,
* warning/error badge when applicable.

5.2.3. The active item shall be visually distinct from queued, completed, cancelled, or failed items.

5.2.4. The queue section and completed-history section may be visually separated, but both shall remain part of the same request-record model.

### 5.3. Request Details / Diagnostics surface

5.3.1. The request details surface shall present a selected request record in greater depth.

5.3.2. It shall include, where available:

* full or redacted submitted text,
* route and endpoint information,
* selected personality and engine,
* submission, connection, remote, and playback state timeline,
* warnings,
* failure classification,
* local and canonical remote request identifiers,
* session ID,
* consumer-platform hint state when applicable,
* queue-priority class when configured,
* terminal synthesis metadata summary,
* machine-readable warning codes,
* API-version negotiation details,
* retry/cancellation events.

5.3.3. Advanced diagnostics may include rawer transport-facing details, but the surface shall remain reviewable by non-developer users.

### 5.4. Settings surface

5.4.1. The settings surface shall be organized into logical sections rather than a single flat form.

5.4.2. Recommended settings groups:

* **General**
* **Playback**
* **Input & Hotkeys**
* **Routes & Endpoints**
* **Diagnostics & Privacy**
* **Advanced compatibility/integration**

5.4.3. Settings that map to verified VoxSpeak metadata or compatibility policy shall be clearly labeled as advanced or deployment-facing settings.

### 5.5. Connection / Endpoint test dialog

5.5.1. The application shall provide an explicit connection-test workflow without requiring user text submission.

5.5.2. The test result surface shall distinguish:

* endpoint unreachable,
* transport security/authentication problem,
* version incompatibility,
* optional-surface unavailable (`UNIMPLEMENTED` compatibility case),
* reachable and compatible baseline.

### 5.6. Capture-region configuration surface

5.6.1. The application shall provide a configuration surface for a fixed capture region.

5.6.2. That surface shall allow a user to define, review, replace, and clear the saved region.

5.6.3. The design shall account for multi-monitor coordinates and scaling differences without assuming a single-monitor desktop.

### 5.7. Transient notifications and overlays

5.7.1. Background workflows such as clipboard send, dynamic screen capture, fixed-region capture, and non-fatal playback state changes should use lightweight transient feedback.

5.7.2. Recommended notification uses include:

* capture started,
* no text found,
* request queued,
* pseudo-streaming fallback warning,
* playback device lost,
* remote request cancelled,
* request failed.

5.7.3. Notifications shall not be the sole carrier of critical information; the same information shall remain available from the queue/history and diagnostics surfaces.

---

## 6. Primary User Workflows

### 6.1. Typed text submission workflow

6.1.1. Baseline flow:

1. user opens or focuses the Main Player window,
2. enters or pastes text,
3. reviews route and endpoint,
4. optionally adjusts personality, engine, or playback preferences,
5. submits,
6. request enters local queue,
7. active item proceeds through connection/submission/stream/playback states,
8. user may pause, resume, stop local playback, skip, or cancel remote synthesis as permitted by state,
9. completed or failed item remains in history with diagnostics.

6.1.2. The UI shall validate local prerequisites before remote invocation.

6.1.3. Validation failures shall be shown inline near the relevant control and summarized in a top-level status area when appropriate.

### 6.2. Clipboard-send workflow

6.2.1. Clipboard-send shall support invocation while another Windows application has focus.

6.2.2. Recommended flow:

1. user triggers a global hotkey or background command,
2. application snapshots current clipboard text,
3. if text is present, the application creates a submission using an independently configured clipboard-send background submission profile,
4. request is queued without requiring the main window to take focus,
5. a transient notification confirms queued submission or reports local validation failure.

6.2.3. The queued request record shall identify clipboard as the submission source.

6.2.4. The independently configured clipboard-send background submission profile shall support an option to follow the currently active profile instead of overriding it with a dedicated background profile.

6.2.5. If clipboard text is absent or not convertible to supported text content, the application shall reject locally and report that no usable text was found.

### 6.3. Dynamic screen-region capture workflow

6.3.1. Dynamic region capture shall support invocation while another Windows application has focus.

6.3.2. Recommended flow:

1. user triggers the dynamic capture hotkey,
2. application enters a transient cross-monitor capture-overlay mode,
3. user marks a screen region,
4. application extracts text using the chosen text-capture implementation,
5. extracted text is submitted directly,
6. request is queued and represented in history as a dynamic capture submission.

6.3.3. Dynamic screen-region capture shall default to direct-submit behavior rather than review-first behavior.

6.3.4. If extraction yields no text, the application shall report a local capture/input failure rather than a remote synthesis failure.

### 6.4. Fixed-region capture workflow

6.4.1. Fixed-region capture shall use a previously configured screen region.

6.4.2. Recommended flow:

1. user triggers the fixed-region hotkey,
2. application captures the stored region,
3. application extracts text,
4. application submits directly,
5. request record is created with fixed-region source metadata.

6.4.3. If no fixed region is configured, the application shall reject locally and guide the user to the capture-region configuration surface.

### 6.5. Queue management workflow

6.5.1. Users shall be able to review pending and completed items from the queue/history surface.

6.5.2. Recommended per-item actions include:

* focus/open details,
* cancel request if still remotely active,
* stop local playback,
* remove from visible queue/history when allowed by retention policy,
* retry/re-submit using current compatible settings,
* play again from locally buffered audio only if that capability exists in the current implementation.

6.5.3. The UI shall not imply that replay is always possible, because automatic durable audio persistence is not part of the initial baseline.

### 6.6. Settings and connection workflow

6.6.1. The user shall be able to create, edit, select, and test endpoint profiles.

6.6.2. The connection-test workflow should reveal compatibility before the user submits text and should also run opportunistically when an endpoint profile becomes active.

6.6.3. Advanced profile edits that affect transport metadata policy shall be separated visually from ordinary user-facing settings.

6.6.4. If personality or engine discovery compatibility is required for the active workflow and discovery compatibility cannot yet be established, the submission workflow shall block until compatibility is understood.

---

## 7. Controls and Command Set

### 7.1. Primary commands

7.1.1. The primary controls in the main workflow shall include:

* Submit / Play
* Clear text
* Paste
* Route selector
* Endpoint selector
* Personality selector when enabled
* Engine selector when enabled
* Open settings
* Open diagnostics/history

### 7.2. Active playback controls

7.2.1. The active item shall expose, at minimum, the controls required by the functional requirements:

* Pause
* Resume
* Stop local playback
* Skip / next item
* Cancel remote request

7.2.2. **Stop local playback** and **Cancel remote request** shall be visually and textually distinct actions.

7.2.3. If an item has already reached remote terminal completion but is still draining local playback buffers, the UI shall disable or relabel remote cancellation accordingly.

7.2.4. If a request is queued locally but not yet submitted remotely, queue removal shall not be labeled as remote cancellation.

### 7.3. Queue controls

7.3.1. Recommended queue-level controls:

* Clear queued items not yet active
* Remove completed items from the visible list
* Filter by state
* Search recent history

7.3.2. Reordering controls may be deferred, but if implemented they shall affect only the local player queue and shall not imply remote queue reprioritization beyond the verified VoxSpeak metadata class semantics.

### 7.4. Hotkeys

7.4.1. The application shall support configurable hotkeys for:

* clipboard send,
* dynamic screen-region capture,
* fixed-region capture.

7.4.2. Additional local playback hotkeys are recommended but not required for MVP.

7.4.3. Hotkey collisions, invalid registrations, or denied registration attempts shall be surfaced in settings diagnostics.

---

## 8. User-Visible State Model

### 8.1. Composite request state presentation

8.1.1. Each visible request item should present a composite state derived from the architecture’s separate submission, connection, and playback state models.

8.1.2. Recommended user-facing state labels:

* Draft
* Ready
* Validating input
* Queued locally
* Connecting
* Submitting request
* Awaiting server admission
* Awaiting audio
* Receiving audio
* Buffered / ready to play
* Playing
* Paused
* Draining playback
* Completed
* Cancelling
* Cancelled
* Failed
* Needs attention

8.1.3. The UI may collapse some internal states into simpler labels for the main list, but the details surface shall preserve greater specificity.

### 8.2. State clarifications required by the verified transport model

8.2.1. **Awaiting audio** shall mean the request was accepted or is in progress remotely, but no playable audio is yet available locally.

8.2.2. **Receiving audio** shall mean compatible audio chunks are arriving, even if playback has not yet started because local buffering thresholds have not been met.

8.2.3. **Playing** shall mean local audio render has started; it shall not by itself imply remote completion.

8.2.4. **Draining playback** shall mean the remote stream has completed successfully or no more audio is expected, but locally buffered audio is still rendering.

8.2.5. **Completed** shall require both:

* authoritative successful remote completion, and
* local playback drain completion.

8.2.6. **Cancelled** shall be distinct from **Failed** even if transport cleanup details remain uncertain.

8.2.7. **Needs attention** is recommended for recoverable or user-actionable states such as device loss, no fixed capture region configured, version mismatch, or pseudo-streaming with unexpectedly long startup delay.

### 8.3. Connection state presentation

8.3.1. Endpoint-level connection state should be visible independently from any single request.

8.3.2. Recommended connection indicators:

* Offline / unreachable
* Connecting
* Reachable
* Reachable with degraded compatibility
* Authentication or transport-security issue
* Version incompatible

8.3.3. `UNIMPLEMENTED` on optional surfaces such as `GetGlobalSynthesisLimits` or direct-stream convenience shall map to a degraded/compatibility indicator rather than a generic broken-state indicator.

---

## 9. Queue and Playback UX Design

### 9.1. Local queue behavior

9.1.1. The queue shall be presented as **player-local work ordering**, not as a direct mirror of remote admission state.

9.1.2. The UI shall preserve submission order unless a user explicitly reorders or replaces items.

9.1.3. For MVP, a single active playback item with a sequential local queue is the recommended baseline.

9.1.4. If the remote server delays admission or playback begins late because of pseudo-streaming fallback, the active item shall remain visibly active with explanatory status rather than appearing stalled without context.

### 9.2. Active playback presentation

9.2.1. The active playback pane shall show:

* text preview,
* current personality/engine when selected,
* endpoint/profile,
* elapsed playback time when available,
* current playback device,
* buffer/progress indication,
* remote-state summary,
* warnings/fallback indicators.

9.2.2. Progress presentation shall be careful not to imply unavailable precision. Where only best-effort or partial progress exists, the UI should label it accordingly.

9.2.3. The UI may present a dual progress model when useful:

* **remote request progress** when available via status or metadata,
* **local playback progress** based on buffered/rendered audio.

### 9.3. Buffering and start-latency UX

9.3.1. The application shall make the reason for delayed playback legible.

9.3.2. Recommended buffering messages include:

* Preparing request
* Waiting for server audio
* Buffering before playback
* Server is buffering audio before chunk delivery
* Waiting for playback device

9.3.3. When pseudo-streaming fallback warning codes indicate server-side buffering before chunking, the active item shall display a non-fatal notice such as **Playback will start after server buffering completes**.

### 9.4. Pause/resume/stop/skip UX

9.4.1. Pause and resume shall affect local playback.

9.4.2. Stop local playback shall stop current audio render and transition the playback state without implying that the remote request was cancelled unless a separate remote-cancel action is taken.

9.4.3. The MVP interruption policy shall be:

* enqueue a new submission behind the current playback item when the active item’s remaining playback time is below a configurable threshold,
* otherwise interrupt local playback and re-queue the interrupted request.

9.4.4. Skip shall advance local focus to the next queue item according to the configured interruption policy.

9.4.5. When interruption does not automatically cancel the remote request, the skipped or interrupted item shall remain trackable in history until its terminal remote state is known or the user discards it.

### 9.5. Device-loss UX

9.5.1. If the active output device becomes unavailable during playback, the UI shall:

* attempt automatic switch-to-default-device when possible,
* freeze or safely end playback state if automatic switching is not possible or fails,
* preserve request and diagnostics state,
* show a device-specific warning,
* guide the user to select a valid device or confirm the fallback device when required,
* allow replay or retry only within the capabilities of the current implementation.

9.5.2. Device-loss presentation shall be distinct from remote request failure.

---

## 10. Failure, Warning, and Recovery UX

### 10.1. Failure classes the UI shall distinguish

10.1.1. The application shall distinguish, at minimum:

* local validation error,
* capture/input acquisition failure,
* endpoint unreachable / network failure,
* authentication or transport-security failure,
* permission/role/platform policy failure,
* unsupported API version / compatibility failure,
* unsupported optional feature (`UNIMPLEMENTED` compatibility case),
* remote synthesis validation failure,
* remote synthesis/runtime failure,
* remote cancellation,
* local playback/render/device failure,
* contract violation such as incompatible audio format within one stream.

### 10.2. Failure presentation model

10.2.1. Failure presentation should use three layers:

* **inline/local guidance** near the relevant control when the failure is immediately actionable,
* **request-level banner or status card** for the active item,
* **details/diagnostics timeline entry** preserving support-grade detail.

10.2.2. Error copy shall be user-comprehensible first, with technical detail available on demand.

10.2.3. The UI shall preserve the original machine-readable warning code or transport classification in diagnostics even when the user-facing message is simplified.

### 10.3. Warnings and degraded-but-usable cases

10.3.1. The UI shall present warnings distinctly from hard failures.

10.3.2. Important non-fatal warning examples include:

* pseudo-streaming fallback,
* optional capability unavailable,
* endpoint reached with degraded compatibility,
* local retention redaction applied.

10.3.3. Diagnostic severity badges shall use these user-facing labels and severity colors:

* **Fatal (F)** – red,
* **High (H)** – orange,
* **Medium (M)** – yellow/amber,
* **Low (L)** – blue.

10.3.4. The UI shall use the severity colors as supporting indicators only and shall still include text labels or icons so the meaning does not depend on color alone.

10.3.5. Warnings should use non-blocking visuals such as banners, badges, or timeline annotations rather than modal dialogs unless user action is required.

### 10.4. Recovery actions

10.4.1. Where safe and supported, the UI should provide explicit recovery actions such as:

* Retry submission
* Reconnect / test endpoint
* Change endpoint
* Select another playback device
* Open settings
* Copy diagnostics summary
* Dismiss warning

10.4.2. The UI shall not silently retry with weaker streaming semantics unless an explicit policy or user setting authorizes that behavior.

10.4.3. If a request fails because native streaming was required but unsupported, the recovery guidance may suggest changing the streaming preference, but shall not imply that the application automatically retried.

---

## 11. Diagnostics Surface Design

### 11.1. Diagnostics objectives

11.1.1. Diagnostics shall support both end-user troubleshooting and deeper support/developer investigation.

11.1.2. The diagnostics design shall preserve verified transport facts without forcing every user to interpret them.

### 11.2. Request timeline

11.2.1. Each request should have a chronological diagnostic timeline.

11.2.2. Recommended timeline events:

* draft created,
* submission validated,
* queued locally,
* endpoint selected,
* request sent,
* session created,
* subscription started,
* first audio received,
* warning received,
* playback started,
* playback paused/resumed/stopped,
* cancellation requested,
* remote completion received,
* playback drained,
* request completed/failed/cancelled.

11.2.3. Where `StreamSynthesis` is used, the timeline may show a normalized direct-stream event while still preserving the effective remote/session identifiers if available.

### 11.3. Summary fields

11.3.1. The default diagnostics summary shall include:

* local request ID,
* result,
* a very short snippet of requested text,
* total duration in milliseconds when the request succeeds, or failure timestamp when the request fails.

11.3.2. The expanded advanced diagnostics view shall include the default diagnostics summary plus:

* full requested text,
* canonical remote request ID,
* remote session ID,
* route,
* endpoint profile,
* queue-priority class when configured,
* requester/consumer platform policy indicators when relevant,
* effective API version,
* server-published version-policy details when available,
* warning codes,
* terminal metadata summary,
* duration breakdown details,
* any error codes.

### 11.4. Diagnostics export/copy

11.4.1. The application should support copying a diagnostic summary to the clipboard.

11.4.2. Export to file may be deferred, but the underlying diagnostics model should remain compatible with later export.

11.4.3. Redaction settings shall apply consistently to copied/exported diagnostics.

---

## 12. Configuration Surface Design

### 12.1. General settings

12.1.1. General settings should include:

* startup behavior,
* window behavior,
* default route,
* default submission behavior for clipboard/capture workflows.

12.1.2. The independently configured clipboard-send background submission profile shall be configured from the general settings surface and shall support either:

* a dedicated background profile, or
* a follow-active-profile mode.

### 12.2. Playback settings

12.2.1. Playback settings shall include:

* follow system default device vs fixed device,
* selected output device,
* local volume if implemented,
* buffering preference if exposed,
* interruption policy.

12.2.2. Buffering controls, if exposed, shall be described as local playback behavior and shall not imply a change to the VoxSpeak wire contract.

### 12.3. Input and hotkey settings

12.3.1. Input settings shall include:

* clipboard-send enablement,
* clipboard-send hotkey,
* dynamic capture hotkey,
* fixed-region capture hotkey,
* direct-submit capture policy,
* fixed capture region configuration entry point.

### 12.4. Route and endpoint settings

12.4.1. Endpoint profiles shall be first-class configuration entities.

12.4.2. Profile fields may include, as applicable:

* profile name,
* route kind,
* server address/port,
* transport security/auth settings,
* default personality/engine preferences,
* API-version preference,
* queue-priority class,
* platform identity hints,
* advanced metadata/policy toggles.

12.4.3. Queue-priority configuration shall be constrained to the verified bounded semantics `0..2` and should use descriptive labels rather than exposing arbitrary integer entry.

12.4.4. Platform identity controls shall be framed as deployment compatibility settings, not everyday content settings.

12.4.5. Consumer tokens shall not appear as persistent user-editable settings because they are session-scoped runtime data returned by VoxSpeak.

### 12.5. Diagnostics and privacy settings

12.5.1. Diagnostics/privacy settings shall include:

* history retention period,
* text retention policy,
* redaction behavior,
* diagnostic verbosity,
* optional telemetry enablement if implemented.

12.5.2. The current design default shall be full retention with no privacy redaction.

12.5.3. The user shall be able to understand whether full text retention is enabled and what diagnostic/request history data is preserved locally.

---

## 13. Accessibility and Inclusive UX

13.1. The Windows application shall be operable with keyboard-only navigation.

13.2. All primary actions shall have accessible names and predictable focus order.

13.3. The design shall support screen-reader-friendly labeling for:

* text editor,
* route selector,
* endpoint selector,
* queue list,
* active playback status,
* warnings/errors,
* diagnostics timeline controls,
* hotkey configuration fields.

13.4. Status changes that materially affect playback or submission outcome shall be announced through accessible mechanisms rather than color alone.

13.5. Visual distinction for states shall not depend on color alone; icons, text labels, or pattern changes shall also be used.

13.6. The design should support high-contrast themes and Windows accessibility settings.

13.7. Text-entry, queue interaction, and diagnostics review shall remain usable at common desktop scaling levels, including mixed-DPI monitor environments.

13.8. Global hotkey workflows shall provide accessible confirmation feedback that does not require the user to visually inspect the main window immediately.

13.9. Capture-region overlays shall remain perceivable in multi-monitor and high-DPI setups and should include keyboard escape/cancel behavior.

13.10. Where playback device errors or compatibility issues occur, the UI shall describe the recovery action in plain language.

---

## 14. Recommendations on Implementation Technology

14.1. This specification does not require a specific Windows UI toolkit, hotkey library, OCR/text extraction library, or audio backend.

14.2. A framework choice is acceptable if it supports, at minimum:

* responsive asynchronous UI updates,
* global hotkey registration,
* accessible controls and automation support,
* multi-monitor and DPI-aware capture workflows,
* separation between view models and transport/playback internals,
* reliable PCM playback to Windows output devices.

14.3. An MVVM-capable desktop stack is recommended because it aligns with the existing architecture’s separation of UI, orchestration, route adapters, playback, and persistence.

14.4. Any framework-specific decision remains an implementation recommendation, not a protocol or product requirement.

---

## 15. MVP Scope and Phasing

### 15.1. MVP scope

15.1.1. The MVP Windows application should include:

* a persistent main desktop window,
* typed-text submission,
* direct-to-VoxSpeak route selection,
* endpoint profile selection and connection testing,
* session-based VoxSpeak interoperability baseline,
* optional direct-stream optimization hidden behind the adapter,
* active playback controls,
* local sequential queue,
* request history with per-request diagnostics,
* playback-device selection including follow-system-default mode,
* clipboard-send workflow,
* accessibility baseline for keyboard and screen-reader operation,
* clear failure and warning presentation for the verified VoxSpeak integration model.

15.1.2. MVP diagnostics shall preserve enough detail to troubleshoot:

* connectivity problems,
* permission/role/platform policy denials,
* pseudo-streaming fallback,
* version incompatibility,
* local playback device problems,
* cancellation outcomes.

### 15.2. Deferred scope

15.2.1. The following items are recommended for post-MVP unless independently prioritized:

* dynamic and fixed screen-region capture,
* queue reordering UI,
* richer offline discovery caching,
* advanced replay from persisted audio,
* personality working-copy editing workflows,
* developer-grade diagnostics export packages,
* richer theming customization,
* background/minimized-only operation modes,
* advanced telemetry dashboards,
* any route-specific UX for a future upstream router.

### 15.3. Future-route guarded scope

15.3.1. The following remain explicitly TBD pending a verified upstream contract:

* transformed-text preview from an upstream router,
* upstream conversational/session semantics,
* upstream-specific cancellation semantics,
* upstream non-streaming response modes,
* upstream-specific diagnostics fields,
* any UX that assumes VoxThink request or transport shapes.

---

## 16. Discrepancies, Constraints, and Design Notes

16.1. No repository-verified evidence currently supports exposing additional transport modes beyond the documented VoxSpeak gRPC-over-HTTP/2 baseline; therefore the application design shall not imply alternate verified transports.

16.2. The direct-stream convenience path exists, but the repository also verifies that interoperable correctness must remain anchored to the session-based flow. Therefore the UI design is intentionally session-compatible first.

16.3. The implementation verifies that subscription success is authoritatively terminated by `FinalMetadata`, while emitted subscription `AudioChunk` messages use `is_last=false`. The UI and completion semantics in this document therefore follow the implementation rather than any simpler chunk-terminal assumption.

16.4. The implementation verifies that queue priority is bounded metadata rather than an arbitrary request-body tuning field. The configuration design therefore constrains queue-priority UX to bounded classes and places it in advanced profile settings.

16.5. The implementation verifies deployment-policy handling for consumer tokens and Windows/platform identity. The application design therefore surfaces these as compatibility and diagnostics concerns, not as ordinary user content inputs.

---

## 17. Summary

17.1. VoxPlayer’s Windows desktop design centers on one primary workflow: acquire text, submit it to the verified direct-to-VoxSpeak route, play streamed PCM audio locally, and make queue, warning, and failure state legible.

17.2. The application is organized around a Main Player window, queue/history review, request diagnostics, endpoint/settings management, and lightweight background workflow notifications.

17.3. The design deliberately separates local queue/playback UX from remote session semantics while still exposing enough verified transport detail for troubleshooting and deployment compatibility.

17.4. Completion, buffering, fallback, cancellation, and diagnostics behavior are aligned with the verified VoxSpeak session-and-stream model rather than invented desktop-client assumptions.

---

## 18. MVP Scope Summary

18.1. MVP should deliver a usable Windows desktop player for typed text and clipboard-send workflows against the current direct VoxSpeak integration baseline.

18.2. MVP should include endpoint configuration, connection testing, local queue/history, active playback controls, diagnostics, playback-device management, and accessible keyboard-first operation.

18.3. MVP should treat pseudo-streaming fallback, optional-surface `UNIMPLEMENTED`, and platform/role/version policy failures as first-class UX cases.

---

## 19. Deferred Scope Summary

19.1. Deferred items include screen-region capture, advanced queue manipulation, personality working-copy editing, persisted-audio replay, richer diagnostics export, and non-essential presentation enhancements.

19.2. All future-route-specific UX beyond route selection remains deferred until an upstream contract is verified.

---

## 20. UX and Operational Decision Record

20.1. Clipboard-send shall use an independently configured background submission profile, and that profile shall support a follow-active-profile option.

20.2. Screen-region capture shall default to direct-submit behavior once text extraction succeeds.

20.3. Diagnostic severity badging shall use Fatal (red), High (orange), Medium (yellow/amber), and Low (blue) color associations while still preserving non-color indicators.

20.4. The default diagnostics view shall show only ID, result, a very short text snippet, and total-duration-in-milliseconds on success or failure timestamp on failure. The expanded advanced view shall add full requested text, duration breakdown, and error codes.

20.5. Endpoint compatibility testing shall be available on explicit user request and shall also run opportunistically when a profile becomes active.

20.6. The MVP interruption policy shall enqueue behind current playback when the remaining time is below a configurable threshold; otherwise it shall interrupt and re-queue the interrupted request.

20.7. Device-loss recovery shall prefer automatic switch-to-default-device when possible.

20.8. When personality or engine discovery compatibility is required for the active workflow, the UI shall block until compatibility is understood.

20.9. The current design default for retention/privacy shall be full retention with no privacy redaction.

20.10. No additional unresolved UX or operational questions are introduced by this revision beyond future-route topics that remain explicitly TBD elsewhere in this document.
