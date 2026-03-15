# VoxCore – VoxSpeak TTS Subsystem

## Architecture Specification, Version 1.4 (Approved)

**Document ID:** VoxSpeak-ARCH-v1.4
**Derived from:** VoxSpeak-FR-v1.4
**Project:** VoxCore
**Subsystem:** VoxSpeak (`vox-speak`)
**Status:** Approved

**Revision note:** VoxSpeak-ARCH-v1.4 updates architectural contracts for control-plane operations, version policy signaling, role/platform enforcement, and queue/admission semantics.

---

## 1. Purpose and Architectural Goals

1.1. This document defines the high-level architecture for VoxSpeak, the VoxCore TTS subsystem, to satisfy VoxSpeak-FR-v1.4.

1.2. Primary architectural goals:

* Multi-engine support via stable adapter boundaries.
* Personality-centric configuration resolving to engine/model-specific parameters.
* Unified batch and streaming synthesis surfaces.
* Pluggable preprocessing and post-processing pipelines.
* Optional caching, QC, and orchestration without entangling the core.
* Strong observability and reproducibility (metadata-first).
* Reference-sample conditioning inputs are recorded in synthesis metadata; engine behavior remains backend-dependent and may be nondeterministic.

---

## 2. System Context

2.1. VoxSpeak is a library-first subsystem designed for headless operation. Optional front-ends (CLI, web/GUI) consume the same API layer.

2.2. VoxSpeak integrates with:

* TTS engines (Piper, Coqui) via local binaries/libraries.
* External audio tools (FFmpeg/Sox) via a constrained invocation layer.
* Downstream consumers via in-memory buffers, streaming, or optional file persistence.

---

## 3. Architectural Overview

3.1. VoxSpeak is decomposed into the following layers:

* API Layer
* Orchestration Layer
* Synthesis Core
* Pipeline Layer
* Persistence Layer (Optional)
* Tooling / Front-Ends (Optional)

3.2. Dependency direction shall flow from top to bottom; lower layers shall not import optional front-ends.

---

## 4. Core Components and Responsibilities

### 4.1. Public API Layer

* Expose `synthesize()` (batch) and `stream()` (streaming) entrypoints.
* Provide typed request/response objects.
* Normalize error types and guarantee stable semantics.

### 4.2. Personality Registry

* Load and validate personality definitions from structured config.
* Resolve a personality into a `ResolvedVoiceProfile` for a chosen engine.
* Provide introspection.
* Support hot-reload (optional, feature-flagged).

### 4.3. Engine Abstraction

* Common adapter interface implemented by each engine.
* Capability discovery and parameter constraints.
* Batch synthesis and streaming where possible.

### 4.4. Text Pipeline

* Apply per-personality preprocessing rules.
* Validate markup for selected engine.
* Produce engine-ready input text and an audit trail.

### 4.5. Audio Pipeline

* Normalize audio into a standard internal representation.
* Apply post-processing via internal DSP and/or external tools through a constrained runner.
* Run optional QC checks.

### 4.6. Orchestration

* Manage job lifecycle.
* Enforce concurrency limits.
* Provide prioritization.
* Implement cancellation propagation.

### 4.7. Persistence and Cache

* Optional file output (off by default).
* Metadata sidecar generation.
* Optional caching keyed by full config.

### 4.8. Tooling and Front-Ends (Optional)

* CLI for batch and interactive use.
* Optional web/GUI front-end, disabled by default.
* Test harness and regression runner.

---

## 5. Data Flow

### 5.1. Batch Synthesis Flow (default: no file output)

1. API receives request.
2. Personality resolved.
3. Text preprocessing/validation.
4. Cache check (optional).
5. Engine synthesis → PCM buffer.
6. Post-processing (optional).
7. QC (optional).
8. Return in-memory result + metadata.
9. If file output enabled: write audio + sidecar.

### 5.2. Streaming Synthesis Flow (preferred when not saving)

1. API receives streaming request.
2. Personality resolve + text preprocessing/validation.
3. Engine streams PCM chunks (if supported).
4. Streaming-capable post-processing stages applied where possible.
5. StreamHandle yields chunks.
6. Cancellation propagates consumer → orchestration → engine.
7. Finalize stream, emit final metadata.

Note: If an engine lacks native streaming, implementation may optionally pseudo-stream a full buffer; this shall be labeled accordingly.

---

## 6. Streaming Semantics

6.1. Standard streaming output: PCM chunks with explicit framing metadata.

6.2. StreamHandle control surface: iteration/async iteration, cancel, status, final metadata.

6.3. Supported modes: real-time (paced) and fast-as-possible.

---

## 7. Capability Discovery and Negotiation

7.1. Engine adapters expose capabilities used for routing and validation.

7.2. Personality routing rules may prefer engines based on streaming/quality.

---

## 8. Caching Strategy

8.1. Cache keys derived from normalized text, engine/model, resolved params, post-processing params, and optional tool versions.

8.2. Cache policy may be per personality and per request.

8.3. Cache lookup occurs after text processing and before synthesis.

---

## 9. Metadata and Reproducibility

9.1. VoxSpeak emits structured metadata for each synthesis regardless of output mode.

9.2. Metadata includes original/processed text, resolved profile, engine/model, parameters, post-processing, QC, cache status, timings, and warnings.

---

## 10. Safety and External Tool Invocation

10.1. External tool invocation is mediated by a runner that avoids shell evaluation, enforces allowlists, and records structured logs.

---

## 11. Cross-Cutting Architectural Contracts

11.1. The architecture shall provide a control-plane capability set including synthesis validation, personality linting, and personality reload operations.

11.2. Control-plane operations shall return structured diagnostics with stable severity and machine-readable identifiers.

11.3. The architecture shall support explicit requester and consumer role signaling and shall enforce role-gated operations according to configured policy.

11.4. The architecture shall support optional consumer platform hints and shall define validation/enforcement behavior as a deploy-time policy choice.

11.5. The architecture shall include admission control with explicit request-priority semantics, deterministic default priority behavior, and structured overload outcomes.

11.6. The architecture shall emit stable lifecycle stage/event notifications for synthesize, stream, and batch workflows.

11.7. The architecture shall expose API version-policy metadata, including requested and effective version outcomes, across all network-visible interfaces.

11.8. When request-level version input is omitted, the architecture shall apply a documented server-defaulting rule without weakening compatibility enforcement.

---

## 12. Deferred Decisions

12.1. Define which post-processors are stream-safe.

12.2. Decide whether pseudo-streaming fallback is enabled by default or opt-in.

12.3. Hot-reload may be feature-flagged and/or dev-only.

---

## Appendix A – Distributed Deployment Clarification (v1.1)

A.1. VoxSpeak is expected to operate in distributed deployments where the VoxSpeak subsystem, callers, and audio consumers may run on different machines and operating systems (e.g., Windows and Ubuntu).

A.2. The architectural layering and responsibilities defined in this document remain valid for both local and remote deployments.

A.3. VoxSpeak shall be deployable behind a service boundary (RPC or equivalent) without altering internal module boundaries.

A.4. Streaming, cancellation, safety limits, and metadata propagation shall be preserved across process and network boundaries.

A.5. This appendix is a non-breaking clarification and does not supersede any existing architectural decision.

---

## Appendix B – Deployment Topology Clarification (v1.2)

B.1. VoxSpeak service deployment target is Ubuntu.

B.2. VoxThink is expected to run on Ubuntu and to call VoxSpeak as a client.

B.3. A Windows host is expected to act as a client and/or an audio consumer.

B.4. The architecture shall support efficient audio streaming from Ubuntu (VoxSpeak) to Windows clients.

B.5. The architecture shall support multiple client roles simultaneously (e.g., VoxThink initiating synthesis while Windows consumes the resulting stream).

B.6. This appendix is a non-breaking clarification. It narrows the expected topology but does not remove support for other distributed arrangements.

---

## Appendix C – Client Role Clarification (v1.3)

C.1. VoxThink shall never act as an audio consumer.

C.2. The architecture shall assume Windows clients are the sole consumers of VoxSpeak audio streams.

C.3. The architecture shall support a split-role interaction pattern where VoxThink initiates synthesis requests while Windows clients receive audio streams.

C.4. This implies a transport/session mechanism or equivalent routing capability between requester and consumer roles.

C.5. This appendix is a non-breaking clarification and is authoritative for client-role constraints in the v1.x architectural baseline.

---

End of Document
