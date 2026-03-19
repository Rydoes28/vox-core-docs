# VoxCore – VoxSpeak TTS Subsystem

## Architecture Specification, Version 1.5 (Approved)

**Document ID:** VoxSpeak-ARCH-v1.5
**Derived from:** VoxSpeak-FR-v1.5
**Project:** VoxCore
**Subsystem:** VoxSpeak (`vox-speak`)
**Status:** Approved

**Revision note:** VoxSpeak-ARCH-v1.5 aligns the architectural baseline with the current public contracts for regression workflows, transport metadata policy, pseudo-stream fallback signaling, and control-plane compatibility behavior.

---

## 1. Purpose and Architectural Goals

1.1. This document defines the high-level architecture for VoxSpeak to satisfy VoxSpeak-FR-v1.5.

1.2. Primary architectural goals:

* multi-engine support via stable adapter boundaries
* personality-centric configuration resolving to engine/model-specific parameters
* unified single-shot, batch, comparative, regression, and streaming synthesis surfaces
* pluggable preprocessing and post-processing pipelines
* optional caching, quality control, and orchestration without entangling the synthesis core
* strong observability and metadata-first reproducibility review

---

## 2. System Context

2.1. VoxSpeak shall be a library-first subsystem designed for headless operation.

2.2. Optional front-ends, including CLI and GUI-oriented clients, shall consume the same public API and transport contracts.

2.3. VoxSpeak shall integrate with:

* local or embedded TTS engines
* approved external audio-processing tools
* downstream consumers through memory, streaming, or optional file persistence

---

## 3. Architectural Overview

3.1. VoxSpeak shall be decomposed into the following layers:

* Public API Layer
* Transport Layer
* Orchestration Layer
* Synthesis Core
* Text and Audio Pipeline Layer
* Persistence and Cache Layer
* Optional Tooling Layer

3.2. Dependency direction shall flow from public surfaces toward execution layers.

3.3. Optional tooling shall not redefine core synthesis semantics.

---

## 4. Core Components and Responsibilities

### 4.1. Public API Layer

4.1.1. The public API layer shall expose stable contracts for single-shot synthesis, streaming, batch synthesis, comparative synthesis, regression-corpus workflows, validation, introspection, and personality governance.

4.1.2. The public API layer shall provide typed request and result objects for in-process callers.

4.1.3. Transport-only concerns, including caller role, caller platform metadata, queue priority, and session-subscription authorization, shall not be required fields on the in-process `SynthesisRequest` object.

### 4.2. Transport Layer

4.2.1. The transport layer shall map the approved public contracts onto the approved remote protocol.

4.2.2. The transport layer shall carry requester role, consumer role, requester platform, consumer platform, and queue-priority signals through transport metadata and session-oriented request fields where the wire contract requires them.

4.2.3. The transport layer shall expose compatibility metadata for API-version negotiation without requiring additional fields on every response body.

4.2.4. The transport layer shall support graceful degradation when communicating with older servers that do not implement the global-limits RPC.

### 4.3. Personality Registry

4.3.1. The personality registry shall load, validate, lint, describe, list, and reload personality definitions from structured configuration.

4.3.2. The personality registry shall resolve a personality into an engine-specific effective configuration.

### 4.4. Engine Abstraction

4.4.1. Each engine shall implement a common synthesis contract.

4.4.2. Engine capability discovery shall provide routing, validation, and compatibility information.

4.4.3. Engine adapters may differ in streaming capability, markup support, and reference-sample behavior without changing the public surface.

### 4.5. Text Pipeline

4.5.1. The text pipeline shall apply per-personality preprocessing rules and markup validation.

4.5.2. The text pipeline shall be able to emit an audit-oriented trace of text transformations.

### 4.6. Audio Pipeline

4.6.1. The audio pipeline shall normalize synthesis output into a standard internal audio representation.

4.6.2. The audio pipeline shall support post-processing through internal or approved external stages.

4.6.3. The audio pipeline shall distinguish stream-safe and buffered-only processing behavior.

### 4.7. Orchestration

4.7.1. The orchestration layer shall manage job lifecycle, queueing, cancellation, and admission control.

4.7.2. Admission control shall support a bounded queue-priority contract with deterministic default behavior.

4.7.3. The orchestration layer shall publish structured overload outcomes.

4.7.4. The orchestration layer shall preserve deterministic ordering for batch-result publication even when execution is parallelized.

### 4.8. Persistence and Cache

4.8.1. File persistence shall remain optional.

4.8.2. The persistence layer shall generate metadata sidecars when file persistence is used.

4.8.3. The cache layer shall support cache hits, bypass, forced regeneration, and targeted invalidation.

### 4.9. Optional Tooling Layer

4.9.1. Optional tooling may expose smoke, regression, or operator workflows.

4.9.2. Optional tooling shall not redefine approved public semantics.

---

## 5. Architectural Data Flows

### 5.1. Single-Shot and Batch Flow

5.1.1. The architecture shall support the following ordered stages:

1. request intake and validation
2. personality resolution
3. text preprocessing and markup validation
4. cache lookup when enabled
5. engine synthesis
6. post-processing when enabled
7. quality-control checks when enabled
8. result publication with metadata
9. optional artifact persistence

5.1.2. Batch and comparative flows shall preserve caller-visible item identity and ordering.

5.1.3. Regression-corpus flows shall expand corpus phrases in a deterministic order before batch execution.

### 5.2. Streaming Flow

5.2.1. The architecture shall support native streaming when the resolved engine, requested mode, and processing chain permit it.

5.2.2. The architecture shall support pseudo-stream fallback when native streaming is unavailable or when buffered processing is required by policy or pipeline safety.

5.2.3. Pseudo-stream fallback shall emit machine-readable warning codes that distinguish fallback causes.

5.2.4. Streaming flows shall terminate with final metadata or a transport error.

---

## 6. Streaming Semantics

6.1. Standard streaming output shall consist of framed PCM chunks plus terminal metadata.

6.2. The stream-handle abstraction shall support iteration, cancellation, status inspection, and final metadata retrieval.

6.3. The architecture shall support real-time and fast-as-possible streaming modes.

6.4. Streaming requests that require native streaming and cannot be satisfied shall fail explicitly rather than silently degrading.

---

## 7. Capability Discovery and Compatibility

7.1. Engine adapters shall expose capabilities used for routing and validation.

7.2. The architecture shall expose server-global synthesis limits separately from engine-local capabilities.

7.3. The architecture shall treat absence of the global-limits RPC on older servers as a compatibility state rather than as a malformed response.

7.4. API-version policy shall be advertised through transport-visible metadata, and capability responses may include an advisory copy of that policy.

---

## 8. Regression and Metadata Architecture

8.1. The architecture shall support loading regression corpora from file-based or in-memory phrase sources.

8.2. Regression workflows shall synthesize a deterministic phrase/personality/engine matrix.

8.3. Regression metadata artifacts shall preserve stable item labels, executed matrix order, resolved personality context, and concise mismatch reporting inputs.

8.4. Regression artifact diffing shall ignore approved volatile fields while preserving meaningful contract differences.

---

## 9. Diagnostics and Observability

9.1. VoxSpeak shall emit structured synthesis metadata for every completed request.

9.2. The architecture shall support stage-oriented event publication for synthesize, stream, validate, and batch workflows.

9.3. Personality governance operations shall use a dedicated personality-diagnostic schema that is stable across lint and reload operations.

9.4. Validation operations shall use a validation-diagnostic schema that is stable across preflight responses.

9.5. Warning codes shall be suitable for automated interpretation.

---

## 10. Security and Policy Enforcement

10.1. The architecture shall support requester-role and consumer-role enforcement at the transport boundary.

10.2. Session subscription shall validate a session-scoped consumer token by default.

10.3. The architecture may permit deployment-specific relaxation of missing-token enforcement, but supplied tokens shall still be validated.

10.4. Consumer-platform hints shall be optional session-creation inputs.

10.5. Platform restrictions, including Windows-only consumption, shall be deploy-time policy controls rather than unconditional architectural assumptions.

---

## 11. Safety and External Tool Invocation

11.1. External tool invocation shall be mediated by a constrained runner that avoids shell interpretation and supports allow-list enforcement.

11.2. External tool usage shall remain observable through structured diagnostics or logging.

---

## 12. Deferred Decisions

12.1. The architecture may define additional stream-safe post-processors in future revisions.

12.2. The architecture may add additional remote transports only through a formally approved transport revision.

12.3. The architecture may extend regression artifact schemas in a backward-compatible manner.

---

## Appendix A – Distributed Deployment Clarification

A.1. VoxSpeak shall support distributed deployments where the service, callers, and consumers operate on different hosts or operating systems.

A.2. Distributed deployment shall not change the approved semantics for streaming, cancellation, metadata propagation, or policy enforcement.

---

End of Document
