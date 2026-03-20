# VoxCore – VoxSpeak TTS Subsystem

## API Specification, Version 1.5 (Approved)

**Document ID:** VoxSpeak-API-v1.5
**Derived from:** VoxSpeak-FR-v1.5, VoxSpeak-ARCH-v1.5
**Project:** VoxCore
**Subsystem:** VoxSpeak (`vox-speak`)
**Status:** Approved

**Revision note:** VoxSpeak-API-v1.5 aligns the public Python API with the shipped request types, regression workflows, control-plane results, and transport-compatibility behavior.

---

## 1. Scope

1.1. This document specifies the public Python API contracts for VoxSpeak.

1.2. The API shall remain library-first and suitable for automation, agent runtimes, and optional front-ends.

1.3. This specification defines:

* single-shot synthesis
* streaming synthesis
* batch synthesis
* comparative synthesis
* regression-corpus workflows
* validation and personality governance
* personality and engine introspection
* global-limits introspection
* transport-only deployed-operator runtime-metrics export semantics
* observability hooks and error contracts

---

## 2. Design Principles

2.1. The API shall provide a stable public surface.

2.2. The API shall use typed contracts.

2.3. The API shall be metadata-first.

2.4. The API shall default to non-persistent outputs.

2.5. The API shall remain engine-agnostic.

---

## 3. Public Package Surface

3.1. The `voxspeak` package shall expose public entry points including:

* `synthesize(...)`
* `stream(...)`
* `synthesize_batch(...)`
* `synthesize_comparative(...)`
* `load_regression_corpus(...)`
* `synthesize_regression_corpus(...)`
* `build_regression_metadata_artifact(...)`
* `write_regression_metadata_artifact(...)`
* `diff_regression_metadata_artifacts(...)`
* `validate_synthesis(...)`
* `lint_personalities()`
* `reload_personalities()`
* `list_personalities()`
* `describe_personality(...)`
* `list_engines()`
* `engine_capabilities(...)`
* `get_global_synthesis_limits()`
* stage-event callback registration functions

3.2. The package shall expose public DTOs and enums through `voxspeak.types`.

3.3. The package shall expose public exception types through `voxspeak.errors`.

3.4. The package shall expose client configuration entry points through `voxspeak.config`.

---

## 4. Configuration Entrypoints and Precedence

4.1. The configuration surface shall provide these entry points:

* `load_client_config_from_env(...)`
* `set_default_client_config(...)`
* `get_default_client_config()`
* `resolve_client_config(...)`

4.2. Configuration resolution shall apply the following precedence order from highest to lowest:

1. direct function arguments to `resolve_client_config(...)`
2. an explicit `ClientConfig` object passed as `config=...`
3. `VOXSPEAK_*` environment variables
4. built-in defaults

4.3. `verbose_debug_logging` shall support tri-state resolution semantics until effective configuration is computed.

4.4. The default transport client shall apply the same precedence rules.

---

## 5. Core Data Types

### 5.1. Identifiers and Enums

5.1.1. `EngineId` shall be a string identifier.

5.1.2. `PersonalityId` shall be a string identifier.

5.1.3. `OutputMode` shall support `MEMORY`, `STREAM`, and `FILE`.

5.1.4. `StreamMode` shall support `REALTIME` and `FAST_AS_POSSIBLE`.

### 5.2. `AudioFormat`

5.2.1. `AudioFormat` shall include:

* `sample_rate_hz: int`
* `channels: int`
* `sample_format: str | enum-compatible value`

### 5.3. `SynthesisRequest`

5.3.1. `SynthesisRequest` shall include:

* `text: str`
* `personality_id: PersonalityId`
* `engine_id: EngineId | None`
* `output_mode: OutputMode = MEMORY`
* `audio_format: AudioFormat | None`
* `enable_postprocessing: bool = True`
* `enable_qc: bool = False`
* `cache_policy: CachePolicy | None`
* `file_output: FileOutputOptions | None`
* `stream_options: StreamOptions | None`
* `request_id: str | None`
* `verbose_debug_logging: bool | None`

5.3.2. `SynthesisRequest` shall not require transport-only role, platform, token, or queue-priority fields.

5.3.3. `file_output` shall be required when `output_mode=FILE`.

5.3.4. `stream_options` shall be required when `output_mode=STREAM`.

### 5.4. `StreamOptions`

5.4.1. `StreamOptions` shall include:

* `mode: StreamMode`
* `chunk_duration_ms: int | None`
* `prefetch_ms: int | None`

### 5.5. `FileOutputOptions`

5.5.1. `FileOutputOptions` shall include:

* `output_dir: str`
* `filename_template: str | None`
* `write_metadata_sidecar: bool = True`
* `container: str | None`

### 5.6. `CachePolicy`

5.6.1. `CachePolicy` shall include:

* `enabled: bool`
* `backend: str | None`
* `bypass: bool = False`
* `invalidate_key: str | None`
* `force_regenerate: bool = False`
* `invalidate_prefix: str | None`

### 5.7. `SynthesisResult`

5.7.1. `SynthesisResult` shall include:

* `request_id: str`
* `output_mode: OutputMode`
* exactly one of `audio`, `stream`, or `file`
* `metadata: SynthesisMetadata`

### 5.8. `StreamHandle` and `AudioChunk`

5.8.1. `StreamHandle` shall support iteration, cancellation, status inspection, and final metadata retrieval.

5.8.2. `AudioChunk` shall include explicit format metadata, chunk index, PCM bytes, and an end-of-stream indicator.

### 5.9. `SynthesisMetadata`

5.9.1. `SynthesisMetadata` shall include, when available:

* timestamps
* original and processed text
* personality, engine, and model identifiers
* synthesis parameters
* post-processing chain description
* quality-control results
* cache information
* timing information
* warning codes
* optional text-processing trace entries

5.9.2. The current public `SynthesisMetadata` contract shall not require requested-versus-effective reproducibility outcome fields.

### 5.10. Control-Plane Result Types

5.10.1. `ValidationResult` shall contain `ok`, validation diagnostics, and resolved configuration preview data.

5.10.2. `PersonalityLintResult` shall contain `ok`, personality diagnostics, and discovery metadata.

5.10.3. `PersonalityReloadResult` shall contain `ok`, `reloaded`, personality diagnostics, and discovery metadata.

5.10.4. `GlobalSynthesisLimits` shall include `available` plus published server-global limits.

5.10.5. When `GlobalSynthesisLimits.available` is `False`, callers shall treat all remaining fields as compatibility placeholders rather than as effective runtime limits.

---

## 6. Public Functions

### 6.1. `synthesize()`

6.1.1. `synthesize()` shall default to `OutputMode.MEMORY`.

6.1.2. `synthesize()` shall always return metadata.

6.1.3. When `OutputMode.FILE` is selected, `synthesize()` shall return a file artifact rather than an in-memory buffer.

### 6.2. `stream()`

6.2.1. `stream()` shall default to real-time streaming unless overridden.

6.2.2. `stream()` shall prefer native streaming when available.

6.2.3. When pseudo-stream fallback is used, the final metadata shall include machine-readable warning codes.

### 6.3. `synthesize_batch()`

6.3.1. `synthesize_batch()` shall accept either a `BatchSynthesisRequest` or an iterable of batch items.

6.3.2. Returned item ordering shall preserve request ordering.

6.3.3. The batch contract shall support deterministic fail-fast and continue-on-error policies.

6.3.4. In fail-fast mode, unscheduled items shall be reported as skipped rather than silently omitted.

### 6.4. `synthesize_comparative()`

6.4.1. `synthesize_comparative()` shall synthesize one text input across a caller-specified personality and engine matrix.

6.4.2. Comparative item identifiers shall remain stable and matrix-derived.

### 6.5. Regression Workflows

6.5.1. `load_regression_corpus(...)` shall load UTF-8 text corpora, ignore blank lines and comment lines beginning with `#`, and reject empty corpora.

6.5.2. `synthesize_regression_corpus(...)` shall normalize corpus input, expand phrase/personality/engine combinations deterministically, and return a batch result.

6.5.3. The regression-corpus contract shall preserve stable item identifiers and request identifiers derived from deterministic matrix labels.

6.5.4. `build_regression_metadata_artifact(...)` shall build a deterministic comparison artifact from a batch result.

6.5.5. `write_regression_metadata_artifact(...)` shall persist that artifact.

6.5.6. `diff_regression_metadata_artifacts(...)` shall return concise machine-readable mismatch data.

### 6.6. Control-Plane Operations

6.6.1. `validate_synthesis(...)` shall provide preflight diagnostics and resolved-configuration preview without generating audio.

6.6.2. `lint_personalities()` shall return `PersonalityLintResult`.

6.6.3. `reload_personalities()` shall return `PersonalityReloadResult`.

6.6.4. `get_global_synthesis_limits()` shall return published server-global operational limits for client preflight.

6.6.5. When the connected server does not implement the global-limits RPC, `get_global_synthesis_limits()` shall return a typed unavailable result rather than raising a hard compatibility failure.

6.6.6. The current public `voxspeak` package surface shall not require a `get_runtime_metrics()` function or a typed public runtime-metrics DTO.

6.6.7. Deployed runtime-metrics retrieval shall remain a transport/operator RPC (`GetRuntimeMetrics`) rather than a required public package entry point in the current API baseline.

### 6.7. Introspection

6.7.1. `list_personalities()` and `describe_personality(...)` shall expose loaded personality identity and resolved-description data.

6.7.2. `list_engines()` and `engine_capabilities(...)` shall expose engine identity and capability data.

---

## 7. Error Taxonomy

7.1. All public exceptions shall inherit from `VoxSpeakError`.

7.2. The public error surface shall include structured configuration, lookup, synthesis, streaming, post-processing, cache, file-output, cancellation, and transport-mapped failures.

7.3. Errors shall include a stable error code, a human-readable message, and structured details when available.

7.4. Compatibility failures for unsupported remote API versions shall be surfaced as structured API errors.

---

## 8. Observability Contracts

8.1. The API shall support stage-event callback registration and deregistration.

8.2. Stage events shall include stage identity, event type, request correlation data when available, and error details for failure events.

8.3. Request identifiers shall propagate into metadata and structured events.

8.4. The public API shall not require response DTO fields for remote API-version policy metadata.

8.5. Transport-visible API-version policy metadata may be exposed by the transport layer separately from public result DTOs.

---

## 9. Transport-Aware API Constraints

9.1. The default remote client shall serialize the approved public request DTOs onto the approved protobuf schema.

9.2. Transport metadata shall carry role, platform, and queue-priority inputs for remote workflows.

9.3. Session-based remote streaming shall remain semantically equivalent to the public streaming contract, subject to remote authorization and transport policy enforcement.

9.4. Transport control-plane RPCs may include deployed-operator surfaces such as `GetRuntimeMetrics` that are not required functions on the current public `voxspeak` package surface.

---

## 10. Compatibility Policy

10.1. Request-level wire-version input may be omitted by clients using the approved transport.

10.2. When omitted, the server-default wire version shall be applied.

10.3. When supplied and unsupported, the remote API shall fail with a structured compatibility error that identifies the supported version set.

---

End of Document
