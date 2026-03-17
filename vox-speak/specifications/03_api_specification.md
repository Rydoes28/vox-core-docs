# VoxCore – VoxSpeak TTS Subsystem

## API Specification, Version 1.4 (Approved)

**Document ID:** VoxSpeak-API-v1.4
**Derived from:** VoxSpeak-FR-v1.4, VoxSpeak-ARCH-v1.4
**Project:** VoxCore
**Subsystem:** VoxSpeak (`vox-speak`)
**Status:** Approved

**Revision note:** VoxSpeak-API-v1.4 adds normative contracts for control-plane operations, reproducibility controls, batch/comparative workflows, and API version-policy metadata.

---

## 1. Scope

1.1. This document specifies the public Python API contracts for VoxSpeak.

1.2. The API is library-first and designed to be consumed by automation, agent runtimes, and optional front-ends (CLI/web/GUI).

1.3. This specification defines:

* Single-shot synthesis interface
* Batch and comparative synthesis interfaces
* Streaming synthesis interface
* Personality and capability introspection
* Control-plane validation and governance interfaces
* Caching and persistence controls
* Observability and metadata outputs
* Error taxonomy and overload semantics

---

## 2. Design Principles

2.1. Stable surface.

2.2. Typed contracts.

2.3. Metadata-first.

2.4. Explicit modes (default non-persistent).

2.5. Engine-agnostic.

---

## 3. Public Package Surface (Proposed)

* `voxspeak`:

  * `synthesize(...)`
  * `synthesize_batch(...)`
  * `synthesize_comparative(...)`
  * `stream(...)`
  * `validate_synthesis(...)`
  * `lint_personalities(...)`
  * `reload_personalities(...)`
  * `list_personalities()`
  * `describe_personality(personality_id, ...)`
  * `list_engines()`
  * `engine_capabilities(engine_id)`

* `voxspeak.types` (DTOs, enums)

* `voxspeak.errors` (public exception types)

* `voxspeak.config` (configuration entrypoints)

---

## 3.1. Configuration Entrypoints and Precedence

The `voxspeak.config` surface is implemented via these entrypoints:

* `load_client_config_from_env(...)`
* `set_default_client_config(...)`
* `get_default_client_config()`
* `resolve_client_config(...)`

Configuration precedence is deterministic. When multiple sources provide the same field, the highest-precedence value wins.

1. Function arguments passed directly to `resolve_client_config(...)` (for example `endpoint=...`, `auth_headers=...`, `timeouts=...`, `retries=...`, `verbose_debug_logging=...`).
2. Explicit `ClientConfig` object passed as `config=...`.
3. Environment variables (`VOXSPEAK_*`) loaded via `load_client_config_from_env(...)`.
4. Built-in defaults (for example endpoint `127.0.0.1:50051` and `verbose_debug_logging=False`).

`ClientConfig.verbose_debug_logging` uses tri-state merge semantics during resolution (`None` = unset, `False`, `True`), so omitted values can be cleanly distinguished from explicit opt-out/opt-in until `resolve_client_config(...)` computes the effective boolean used by transport execution.

`transport.get_default_client()` applies the same precedence order by resolving module-level explicit config (`set_default_client_config(...)`) against process environment and built-in defaults before constructing the transport client.

### Migration Notes for Existing Callers

* Existing callers that rely on environment-only configuration continue to work unchanged.
* Existing callers that already call `set_default_client_config(...)` continue to override environment variables globally for default-client construction.
* Callers that need per-call overrides should prefer `resolve_client_config(..., endpoint=..., ...)`, where function arguments intentionally take precedence over explicit/global config and env values.
* If multiple sources are populated, callers should assume the above precedence order rather than implicit merge behavior.

## 4. Core Data Types

### 4.1. Enums and Identifiers

* `EngineId`: string (e.g., `"piper"`, `"coqui"`)
* `PersonalityId`: string (e.g., `"ship_ai"`)
* `OutputMode`: `MEMORY` (default), `STREAM`, `FILE`
* `StreamMode`: `REALTIME`, `FAST_AS_POSSIBLE`

### 4.2. Audio Format Descriptor

`AudioFormat`:

* `sample_rate_hz: int`
* `channels: int`
* `sample_format: str` (e.g., `"s16"`, `"f32"`)

### 4.3. Synthesis Request

`SynthesisRequest`:

* `text: str`
* `personality_id: PersonalityId`
* `engine_id: EngineId | None` (optional override)
* `output_mode: OutputMode = MEMORY`
* `audio_format: AudioFormat | None`
* `enable_postprocessing: bool = True`
* `enable_qc: bool = False`
* `cache_policy: CachePolicy | None`
* `reproducibility: ReproducibilityOptions | None`
* `priority: str | None`
* `requester_role: str | None`
* `consumer_role: str | None`
* `consumer_platform: str | None`
* `file_output: FileOutputOptions | None` (required when output_mode=FILE)
* `stream_options: StreamOptions | None` (required when output_mode=STREAM)
* `request_id: str | None`
* `verbose_debug_logging: bool | None` (optional per-request override for verbose/debug trace emission)


### 4.3A. Reproducibility Controls

`ReproducibilityOptions`:

* `seed: int | None`
* `flags: dict[str, bool] | None`

When provided on a request, reproducibility options shall be interpreted as best-effort controls and shall not require engine support for every option.

### 4.4. Streaming Options

`StreamOptions`:

* `mode: StreamMode`
* `chunk_duration_ms: int | None`
* `prefetch_ms: int | None`

### 4.5. File Output Options

`FileOutputOptions`:

* `output_dir: str`
* `filename_template: str | None`
* `write_metadata_sidecar: bool = True`
* `container: str | None` (e.g., `"wav"`, `"flac"`)

### 4.6. Cache Policy

`CachePolicy`:

* `enabled: bool`
* `backend: str | None` (e.g., `"memory"`, `"file"`)
* `bypass: bool = False`
* `invalidate_key: str | None`
* `force_regenerate: bool = False`
* `invalidate_prefix: str | None`

### 4.7. Synthesis Result

`SynthesisResult`:

* `request_id: str`
* `output_mode: OutputMode`
* `audio: AudioBuffer | None` (MEMORY)
* `stream: StreamHandle | None` (STREAM)
* `file: FileArtifact | None` (FILE)
* `metadata: SynthesisMetadata`

### 4.8. Audio Buffer

`AudioBuffer`:

* `format: AudioFormat`
* `pcm: bytes | memoryview`
* `duration_ms: int | None`

### 4.9. Stream Handle and Chunks

`StreamHandle`:

* iteration (`__iter__`) yielding `AudioChunk`
* optional async iteration (`__aiter__`) yielding `AudioChunk`
* `cancel() -> None`
* `status() -> StreamStatus`
* `final_metadata() -> SynthesisMetadata`

`AudioChunk`:

* `format: AudioFormat`
* `index: int`
* `pcm: bytes | memoryview`
* `is_last: bool`

### 4.10. Metadata

`SynthesisMetadata` includes:

* timestamps
* original and processed text
* personality_id, engine_id, model_id
* resolved params
* post-processing chain
* QC results (if enabled)
* cache info (enabled/hit/key/backend/bypass/force_regenerate/effective/warning/invalidation outcomes)
* timings
* warnings (e.g., pseudo-streaming)
* reproducibility request/effective outcome
* optional text-preprocessing trace entries (rule_id, input_fragment, output_fragment, warning_level)

---

## 5. Public Functions

### 5.1. `synthesize()`

* Defaults to `OutputMode.MEMORY`.
* If `OutputMode.FILE`, writes audio and returns file artifact.
* Always returns metadata.

### 5.2. `synthesize_batch()`

* Accepts an ordered set of synthesis requests.
* Shall preserve request order in returned result ordering.
* Shall define failure policy as all-or-partial completion and return structured per-item outcomes.
* May expose caller controls for parallelism bounds.

### 5.3. `synthesize_comparative()`

* Accepts a base text input with a personality/engine comparison matrix.
* Shall return a structured comparative result set that preserves matrix identity for each output.
* Shall include per-item metadata sufficient for reproducibility audits.

### 5.4. `stream()`

* Defaults to `StreamMode.REALTIME` unless overridden.
* Prefers native streaming.
* If pseudo-streaming fallback is used, marks warnings accordingly.

### 5.5. Control-Plane Operations

* `validate_synthesis(request) -> ValidationResult` shall provide preflight diagnostics and resolved-config preview without generating audio.
* `lint_personalities() -> LintResult` shall return structured diagnostics for loaded personality definitions.
* `reload_personalities() -> ReloadResult` shall return reload outcome and diagnostics.

### 5.6. Personality Introspection

* `list_personalities() -> list[PersonalitySummary]`
* `describe_personality(personality_id: str, engine_id: str | None = None) -> PersonalityDescription`

### 5.7. Engine Introspection

* `list_engines() -> list[EngineSummary]`
* `engine_capabilities(engine_id: str) -> EngineCapabilities`
## 6. Error Taxonomy

6.1. All public exceptions inherit from `VoxSpeakError`.

6.2. Proposed exception types:

* `ConfigurationError`
* `PersonalityNotFoundError`
* `EngineNotFoundError`
* `ModelNotFoundError`
* `MarkupNotSupportedError`
* `ParameterOutOfRangeError`
* `SynthesisError`
* `StreamingNotSupportedError`
* `PostProcessingError`
* `QualityControlError` (optional)
* `CacheError`
* `FileOutputError`
* `ExternalToolError`
* `CancelledError`

6.3. Errors are structured and include `code`, `message`, `details`, and optional `cause`.

---

## 7. Observability Contracts

7.1. The API shall support logger and/or callback hooks for stage lifecycle and structured events with stable stage and event-type identifiers.

7.2. Each request shall include a correlation id (`request_id`) that propagates to logs/metadata.

7.3. API responses shall expose version-policy metadata that includes requested API version (if supplied), effective API version, default server version, supported versions, and policy mode.

---

## 8. Minimal Usage Examples (Non-Normative)

* In-memory:

  * `result = voxspeak.synthesize("Hello", "ship_ai")`
* Streaming:

  * `handle = voxspeak.stream("Hello", "ship_ai", mode="realtime")`
  * `for chunk in handle: play(chunk.pcm)`
* File output (explicit opt-in):

  * `result = voxspeak.synthesize("Hello", "ship_ai", output_mode="file", output_dir="./out")`

---

## 9. API Versioning and Compatibility Policy

9.1. Request-level `api_version` input may be omitted by clients.

9.2. When omitted, the server-default version shall be applied and exposed as the effective version in response metadata.

9.3. When supplied and unsupported, the API shall return a structured compatibility error that includes supported versions.

---

## 10. Open API Decisions

10.1. Whether to provide an explicit `VoxSpeakClient` object for dependency injection.

10.2. Whether async is first-class (async-only streaming) or dual-mode.

10.3. Whether pseudo-streaming fallback is enabled by default or opt-in.

---

## Appendix A – Distributed Deployment Clarification (v1.1)

A.1. VoxSpeak shall support deployment where callers, VoxSpeak itself, and downstream audio consumers run on different machines and operating systems (e.g., Windows and Ubuntu).

A.2. VoxSpeak shall support both in-process usage and out-of-process (client/server) usage without changing the conceptual synthesis or streaming semantics defined in this document.

A.3. VoxSpeak shall define at least one network-capable invocation model for transmitting text requests and audio results between machines.

A.4. Audio transmitted between processes or machines shall use an explicit, OS-agnostic representation with fully specified format metadata.

A.5. Streaming semantics, cancellation, safety limits, and metadata requirements remain applicable across network boundaries.

A.6. This appendix is a non-breaking clarification and does not supersede any existing API contract.

---

## Appendix B – Deployment Topology Clarification (v1.2)

B.1. VoxSpeak servers are expected to run on Ubuntu hosts.

B.2. VoxSpeak shall support receiving text requests from:

* VoxThink (Ubuntu), and/or
* Windows clients (direct).

B.3. VoxSpeak shall support streaming audio outputs to Windows clients for playback/consumption.

B.4. VoxSpeak shall support scenarios where a requester and a consumer are different clients (e.g., VoxThink triggers synthesis while Windows subscribes to the resulting stream), subject to transport/session support.

B.5. This appendix is a non-breaking clarification. It narrows the expected topology but does not remove support for other distributed arrangements.

---

## Appendix C – Client Role Clarification (v1.3)

C.1. VoxThink shall never be an audio consumer; it shall not receive or play VoxSpeak audio.

C.2. Windows clients are the sole consumers of VoxSpeak audio streams.

C.3. The API shall support a split-role workflow where a requester (e.g., VoxThink) initiates synthesis and a separate consumer (Windows client) receives the resulting audio stream.

C.4. Where requester and consumer are separate clients, the API/transport shall provide a session or subscription mechanism enabling the Windows client to attach to the correct stream.

C.5. This appendix is a non-breaking clarification and is authoritative for client-role constraints in the v1.x API baseline.

---

End of Document
