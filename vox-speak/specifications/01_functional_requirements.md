# VoxCore – VoxSpeak TTS Subsystem

## Functional Requirements, Version 1.5 (Approved)

**Document ID:** VoxSpeak-FR-v1.5
**Project:** VoxCore
**Subsystem:** VoxSpeak (`vox-speak`)
**Status:** Approved
**Change policy:** Breaking requirement changes require VoxSpeak-FR-v2; non-breaking clarifications may be issued as VoxSpeak-FR-v1.x.

**Revision note:** VoxSpeak-FR-v1.5 aligns functional requirements with the current external contracts for regression workflows, streaming fallback semantics, session authorization, and transport-visible compatibility behavior.

---

## 1. Scope and Objectives

1.1. The system shall provide a modular TTS subsystem capable of generating speech from text using multiple backends, including Piper and Coqui TTS.

1.2. The subsystem shall support multiple personalities, where each personality defines a coherent voice profile, model choice, and audio-processing configuration.

1.3. The subsystem shall support reproducible and auditable generation to the extent permitted by the selected engine, requested options, and reference-sample inputs.

1.4. The subsystem shall expose equivalent functional behavior for in-process callers and for callers using an approved remote transport.

---

## 2. Engine and Model Support

2.1. The system shall support at least the following TTS engines:

* Piper TTS
* Coqui TTS

2.2. The system shall allow configuration of one or more models per engine.

2.3. The system shall abstract engine-specific behavior behind a common synthesis contract.

2.4. The system shall allow new engines or models to be added without requiring personality redesign.

---

## 3. Personality Definition and Management

3.1. The system shall support named personalities.

3.2. Each personality shall define, at minimum:

* target engine or engines
* specific model or models
* default pitch, speed, and volume parameters
* optional markup, phoneme, or pronunciation conventions
* optional post-processing profile

3.3. Personalities shall be able to resolve to different configurations for different engines.

3.4. Personalities shall be defined through structured configuration rather than hard-coded requirement logic.

3.5. The system shall allow runtime selection of personality.

3.6. The system shall support validation, linting, enumeration, description, and reload of personality configuration through public control-plane contracts.

---

## 4. Text Processing and Markup

4.1. The system shall accept raw text input and optional engine-compatible markup or phoneme annotations.

4.2. The system shall support per-personality preprocessing rules, including pronunciation substitutions, phoneme injection, pauses, and emphasis-oriented transformations.

4.3. The system shall provide validation that detects unsupported markup for the resolved engine.

4.4. The system shall record sufficient text-processing trace data to support diagnostics and regression review when tracing is produced.

---

## 5. Audio Generation and Post-Processing

5.1. The system shall generate audio in a standard lossless intermediate representation prior to optional conversion or persistence.

5.2. The system shall support configurable post-processing stages, which may include:

* pitch shifting
* time stretching or speed adjustment
* equalization
* dynamic range compression
* reverb or other effects

5.3. The system shall permit adjustments either inside the TTS engine, when supported, or through approved external processing tools.

5.4. Post-processing chains shall be configurable per personality and per request.

5.5. The system shall allow post-processing to be disabled for diagnostics, validation, or benchmarking workflows.

5.6. The system shall distinguish stream-safe and non-stream-safe processing behavior for streaming workflows.

---

## 6. Output Management

6.1. Saving generated audio to files shall be optional and disabled by default.

6.2. When file output is enabled, the system shall save generated audio using identifiable structured filenames that encode, at minimum:

* personality
* engine
* model
* request identity, timestamp, or equivalent run discriminator

6.3. The system shall support a configurable output directory structure.

6.4. When file output is enabled, the system shall generate accompanying metadata capturing, at minimum:

* input text
* resolved personality
* resolved engine and model
* synthesis and post-processing parameters
* generation timing information
* warnings emitted during execution

6.5. When file output is disabled, the system shall support in-memory or streaming delivery to downstream consumers.

6.6. File-output persistence shall not be required for streaming-capable integrations.

---

## 7. Batch, Comparative, and Regression Workflows

7.1. The system shall support batch synthesis of multiple requests.

7.2. Batch result ordering shall preserve request ordering regardless of execution parallelism.

7.3. The system shall support a caller-selectable batch failure policy that distinguishes fail-fast behavior from continue-on-error behavior.

7.4. The system shall support comparative synthesis of identical text across a caller-specified personality and engine matrix.

7.5. Comparative and regression workflows shall preserve stable matrix identity for each generated item.

7.6. The system shall support loading a regression corpus from structured text input and normalizing that corpus into an ordered phrase list.

7.7. The system shall support synthesizing a regression corpus across personality and engine matrices using deterministic item labeling and stable expansion order.

7.8. The system shall support building, writing, and diffing regression metadata artifacts so callers can compare observable synthesis behavior across runs.

---

## 8. Programmatic Interface

8.1. The system shall expose a Python API suitable for integration into larger automation workflows.

8.2. The public API shall support at least the following categories of operation:

* single-shot synthesis
* streaming synthesis
* batch synthesis
* comparative synthesis
* regression-corpus synthesis
* dry-run validation
* personality governance and introspection
* engine introspection
* global-limits introspection

8.3. The public in-process request contract shall remain engine-agnostic and shall not require transport-only role, platform, or queue-priority fields.

8.4. The public API shall provide structured error reporting for at least:

* missing models
* unsupported personalities
* engine failures
* post-processing failures
* cancellation
* configuration errors
* admission or resource-limit failures

---

## 9. Diagnostics, Logging, and Reproducibility

9.1. The system shall emit structured metadata for each synthesis operation regardless of output mode.

9.2. Metadata shall include sufficient information to support reproducibility review, including resolved engine/model identity, synthesis parameters, timing information, cache outcomes, and warning codes.

9.3. The system shall support verbose or debug logging that may be enabled by configuration and overridden per request where the contract permits.

9.4. The system shall support stage-oriented lifecycle notifications or equivalent structured observability hooks for synthesis-related workflows.

9.5. The system shall record requested deterministic inputs that are part of the approved public or transport contract, but it shall not claim full determinism when backend behavior remains nondeterministic.

9.6. Reference-sample conditioning inputs shall be treated as potentially nondeterministic, and the system shall preserve sufficient disclosure for regression audits.

---

## 10. Non-Functional but TTS-Relevant Constraints

10.1. The system shall be designed to run headless and be scriptable.

10.2. The system shall avoid assumptions about a single mandatory downstream consumer implementation.

10.3. The system shall prioritize repeatability and configurability over real-time performance unless a real-time mode is explicitly requested.

10.4. The system shall permit deploy-time policy enforcement for consumer-platform restrictions without making those restrictions unconditional product-wide requirements.

---

## 11. Scope Baseline

11.1. Streaming synthesis is in scope and shall be supported either through native chunked generation or through approved pseudo-stream fallback behavior.

11.2. A web or GUI front-end shall remain optional and disabled by default.

11.3. The following items remain out of scope for this version:

* mandatory loudness normalization targets
* mandatory personality versioning semantics
* mandatory compressed audio codecs on the wire

---

## 12. Streaming and Integration Semantics

12.1. The system shall expose a streaming synthesis interface supporting incremental delivery of audio data.

12.2. The streaming interface shall support synchronous iteration and may support asynchronous iteration.

12.3. The streaming output format shall carry explicit audio-format metadata.

12.4. The system shall support cancellation and interruption of active synthesis jobs.

12.5. The system shall support terminal stream finalization, including end-of-stream signaling and final metadata delivery.

12.6. The system shall support at least two streaming modes where supported by the resolved engine and policy:

* real-time or paced streaming
* fast-as-possible streaming

12.7. When native streaming cannot be used, the system may fall back to pseudo-stream buffering and chunk emission.

12.8. Pseudo-stream fallback shall be signaled through machine-readable warning codes and shall distinguish, at minimum, the following fallback classes:

* engine capability fallback
* runtime-unavailable fallback
* server-policy fallback
* file-output-path fallback
* non-stream-safe post-processing fallback

12.9. Streaming requests that explicitly require native streaming and cannot be satisfied shall fail with a structured validation or parameter error rather than silently degrading.

---

## 13. Quality Control and Voice Consistency

13.1. The system may perform post-generation validation, including clipping detection, abnormal silence detection, and duration checks.

13.2. The system shall support attaching one or more reference samples to a personality where the selected engine supports such inputs.

13.3. The system shall support optional post-generation analysis hooks.

13.4. Quality-control stages shall be enableable or disableable per run.

---

## 14. Caching, Deduplication, and Reuse

14.1. The system shall support optional caching of generated audio keyed by the effective synthesis configuration.

14.2. The system shall support configurable cache backends, including in-memory and filesystem-backed operation.

14.3. The system shall support explicit cache bypass and forced regeneration.

14.4. The system shall support targeted cache invalidation.

---

## 15. Personality and Configuration Tooling

15.1. The system shall validate personality definitions at load time.

15.2. The system shall provide mechanisms to enumerate available personalities and inspect resolved configurations.

15.3. The system may support hot reload of personality configuration where technically feasible.

15.4. The system shall expose structured personality diagnostics for lint and reload operations.

---

## 16. Control and Orchestration Layer

16.1. The system shall expose engine capability metadata.

16.2. The system shall support queued and concurrent synthesis jobs.

16.3. The system shall support configurable concurrency limits per engine and globally.

16.4. The system shall support explicit queue-priority input for remote callers.

16.5. The queue-priority contract shall define accepted values, default behavior when omitted, and observable overload outcomes.

16.6. Under overload, the system shall return structured rejection errors rather than silently dropping requests.

16.7. The system shall publish server-global synthesis limits through a control-plane contract.

16.8. Clients interacting with older servers that do not implement the global-limits contract shall be able to represent a typed limits-unavailable outcome instead of treating that condition as an unrecoverable protocol failure.

16.9. The remote session workflow shall support requester and consumer role enforcement.

16.10. Session subscription authorization shall support consumer-token validation.

16.11. Consumer-token enforcement shall be required by default and may be relaxed by deploy-time policy.

16.12. Consumer-platform restrictions, including Windows-only consumption, shall be treated as deploy-time policy options.

---

## 17. Extensibility and Experimental Support

17.1. The subsystem should permit experimental engines, processing stages, and transport-compatible extensions without breaking the approved public contracts.

---

End of Document
