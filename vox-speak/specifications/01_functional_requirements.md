# VoxCore – VoxSpeak TTS Subsystem

## Functional Requirements, Version 1.3 (Approved)

**Document ID:** VoxSpeak-FR-v1.3
**Project:** VoxCore
**Subsystem:** VoxSpeak (`vox-speak`)
**Status:** Approved
**Change policy:** Breaking requirement changes require VoxSpeak-FR-v2; non-breaking clarifications may be issued as VoxSpeak-FR-v1.x.

**Revision note:** VoxSpeak-FR-v1.3 is VoxSpeak-FR-v1.2 with an additional non-breaking client-role clarification (Appendix C). All prior requirements remain unchanged.

---

## 1. Scope and Objectives

1.1. The system shall provide a modular TTS subsystem capable of generating speech from text using multiple backends (initially Piper and Coqui TTS).

1.2. The subsystem shall support multiple “personalities,” where each personality defines a coherent voice profile, model choice, and audio processing configuration.

1.3. The subsystem shall enable reproducible generation of voice lines across engines and personalities.

---

## 2. Engine and Model Support

2.1. The system shall support at least the following TTS engines:

* Piper TTS
* Coqui TTS

2.2. The system shall allow configuration of one or more models per engine.

2.3. The system shall abstract engine-specific APIs behind a common interface.

2.4. The system shall allow new TTS engines or models to be added without refactoring existing personalities.

---

## 3. Personality Definition and Management

3.1. The system shall support named personalities.

3.2. Each personality shall define, at minimum:

* Target TTS engine(s)
* Specific model(s)
* Default pitch, speed, and volume parameters
* Optional phoneme or markup conventions
* Post-processing profile (if any)

3.3. The system shall allow personalities to map to different configurations per engine.

3.4. Personalities shall be definable via structured configuration rather than hardcoded logic.

3.5. The system shall allow runtime selection of personality.

---

## 4. Text Processing and Markup

4.1. The system shall accept raw text input and optional engine-specific markup or phoneme annotations.

4.2. The system shall support per-personality preprocessing rules (e.g., pronunciation substitutions, phoneme injection, pauses, emphasis tags).

4.3. The system shall provide a validation step to detect unsupported markup for the selected engine.

---

## 5. Audio Generation and Post-Processing

5.1. The system shall generate audio in a standard lossless intermediate format (e.g., WAV) prior to optional conversion.

5.2. The system shall support post-processing stages, which may include:

* Pitch shifting
* Time-stretching / speed adjustment
* Equalization
* Dynamic range compression
* Reverb or other effects

5.3. The system shall support performing adjustments either:

* Inside the TTS engine (when supported), or
* Externally via tools such as Sox or FFmpeg.

5.4. Post-processing chains shall be configurable per personality.

5.5. The system shall allow post-processing to be disabled for diagnostic or benchmarking runs.

---

## 6. Output Management

6.1. Saving generated audio to files shall be optional and disabled by default.

6.2. When file output is enabled, the system shall save generated audio files with identifiable, structured filenames that encode, at minimum:

* Personality
* Engine
* Model
* Timestamp or build identifier

6.3. The system shall support a configurable output directory structure.

6.4. When file output is enabled, the system shall generate accompanying metadata for each audio file (e.g., JSON sidecar) capturing:

* Input text
* Personality
* Engine and model
* All synthesis and post-processing parameters
* Generation time

6.5. When file output is disabled, the system shall support in-memory or streaming delivery of audio to downstream consumers.

---

## 7. Batch and Comparative Generation

7.1. The system shall support batch generation of multiple phrases across multiple personalities and/or engines.

7.2. The system shall support paired generation of the same text across different engines for comparison.

7.3. The system shall support deterministic or near-deterministic generation where engines allow.

---

## 8. Programmatic Interface

8.1. The system shall expose a Python API suitable for integration into larger automation workflows.

8.2. The API shall support:

* Single-shot synthesis
* Batch synthesis
* Dry-run / configuration validation

8.3. The system shall provide structured error reporting for:

* Missing models
* Unsupported personalities
* Engine failures
* Post-processing failures

---

## 9. Diagnostics, Logging, and Reproducibility

9.1. The system shall log all synthesis operations with sufficient detail to reproduce outputs.

9.2. The system shall provide a verbose/debug mode exposing raw engine calls and post-processing commands.

9.3. The system shall optionally archive the exact personality configuration used for each generation run.

---

## 10. Non-Functional but TTS-Relevant Constraints

10.1. The system shall be designed to run headless and be scriptable.

10.2. The system shall avoid assumptions about downstream consumers.

10.3. The system shall prioritize repeatability and configurability over real-time performance, unless explicitly enabled.

---

## 11. Confirmed Scope and Remaining Open Items

11.1. Streaming or real-time synthesis is in-scope and shall be supported where the underlying engine allows.

11.2. When generated audio is not saved to a file, the system shall prefer streaming output where supported.

11.3. A web or GUI front-end shall be an optional component, disabled by default.

11.4. The following items have been explicitly marked as out of scope for this version:

* Automatic loudness normalization (e.g., LUFS targets) is not required.
* Formal versioning of personalities is not required.

---

## 12. Streaming and Integration Semantics

12.1. The system shall expose a streaming synthesis interface supporting incremental delivery of audio data.

12.2. The streaming interface shall support both synchronous and asynchronous invocation patterns.

12.3. The system shall define a standardized streaming output format (e.g., PCM frames, sample rate, channel layout).

12.4. The system shall support cancellation and interruption of active synthesis jobs.

12.5. The system shall support clean finalization of streams, including end-of-stream signaling and resource release.

12.6. The system shall support at least two operational modes where supported by the underlying engine:

* Real-time or near-real-time streaming
* Fast-as-possible batch streaming

---

## 13. Quality Control and Voice Consistency

13.1. The system shall optionally perform post-generation validation, including:

* Clipping detection
* Abnormal silence detection
* Basic duration checks

13.2. The system shall support attaching one or more reference samples to each personality.

13.2.1. Reference-sample conditioning shall be treated as potentially nondeterministic even when deterministic controls (e.g., seed) are provided, and effective conditioning inputs shall be recorded in synthesis metadata to support reproducibility audits.

13.3. The system shall support optional post-generation analysis hooks.

13.4. The system shall allow quality control stages to be enabled or disabled per run.

---

## 14. Caching, Deduplication, and Reuse

14.1. The system shall optionally support caching of generated audio keyed by the full synthesis configuration.

14.2. The system shall support configurable cache backends (e.g., in-memory, filesystem).

14.3. The system shall support explicit cache bypass and forced regeneration.

14.4. The system shall support cache invalidation.

---

## 15. Personality and Configuration Tooling

15.1. The system shall validate personality definitions at load time.

15.2. The system shall provide mechanisms to enumerate available personalities and inspect their resolved configurations.

15.3. The system shall support hot-reloading of personality configurations where technically feasible.

15.4. The system shall expose structured validation and linting errors for misconfigured personalities.

---

## 16. Control and Orchestration Layer

16.1. The system shall expose engine capability metadata.

16.2. The system shall support queued and concurrent synthesis jobs.

16.3. The system shall support configurable concurrency limits per engine or globally.

16.4. The system shall support basic prioritization of synthesis jobs.

---

## 17. Extensibility and Experimental Support

17.1. The system shall support registering custom processing stages.

17.2. The system shall support feature flags or experimental options scoped to individual personalities.

17.3. The system shall allow experimental components to be integrated without modification to the core orchestration layer.

---

## 18. Testability, Regression Protection, and Safety Boundaries

18.1. The system shall support a deterministic or test mode where supported by the underlying engines.

18.2. The system shall support batch regeneration of defined test phrases across personalities and engines.

18.3. The system shall support storing and comparing metadata for regression analysis.

18.4. The system shall enforce configurable limits on input size and expected output duration.

18.5. The system shall isolate external tool invocation and prevent arbitrary command execution via configuration.

---

## Appendix A – Distributed Deployment Clarification (v1.1)

A.1. VoxSpeak shall support deployment where callers, VoxSpeak itself, and downstream audio consumers run on different machines and operating systems (e.g., Windows and Ubuntu).

A.2. VoxSpeak shall support both in-process usage and out-of-process (client/server) usage without changing the conceptual synthesis or streaming semantics defined in this document.

A.3. VoxSpeak shall define at least one network-capable invocation model for transmitting text requests and audio results between machines.

A.4. Audio transmitted between processes or machines shall use an explicit, OS-agnostic representation with fully specified format metadata.

A.5. Streaming semantics, cancellation, safety limits, and metadata requirements remain applicable across network boundaries.

A.6. This appendix is a non-breaking clarification and does not supersede any existing requirement.

---

## Appendix B – Deployment Topology Clarification (v1.2)

B.1. VoxSpeak is expected to run on Ubuntu hosts.

B.2. VoxThink is expected to run on Ubuntu hosts and may act as a client of VoxSpeak.

B.3. A Windows host is expected to act as a client of VoxSpeak and/or an audio consumer.

B.4. VoxSpeak-generated audio shall be streamable/transportable to a Windows host for playback/consumption.

B.5. Text input to VoxSpeak shall be accepted from either:

* VoxThink (Ubuntu), or
* A Windows host (direct client).

B.6. This appendix is a non-breaking clarification. It narrows the expected topology but does not remove support for other distributed arrangements.

---

## Appendix C – Client Role Clarification (v1.3)

C.1. VoxThink shall never be an audio consumer of VoxSpeak outputs.

C.2. VoxSpeak-generated audio shall be consumed only by Windows clients.

C.3. VoxSpeak shall support requests originating from VoxThink (Ubuntu) while streaming the resulting audio to Windows clients.

C.4. This appendix is a non-breaking clarification. It further constrains expected client roles but does not remove support for other distributed arrangements unless explicitly stated in later major versions.

---

End of Document
