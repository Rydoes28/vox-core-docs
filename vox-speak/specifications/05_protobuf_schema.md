# VoxCore – VoxSpeak TTS Subsystem

## Protobuf Definitions, Version 1.1 (Approved)

**Document ID:** VoxSpeak-PROTO-v1.1  
**Derived from:** VoxSpeak-TRANSPORT-v1.3 (Approved), VoxSpeak-API-v1.4 (Approved)  
**Project:** VoxCore  
**Subsystem:** VoxSpeak (`vox-speak`)  
**Status:** Approved

**Approval note:** This document is approved as-is.

---

## 1. Scope

1.1. This document defines the `.proto` schema baseline for the VoxSpeak gRPC transport.

1.2. It expresses:
- Session-based synthesis (required)
- Streaming subscription (Windows consumer)
- Control-plane validation and personality-governance RPCs
- Introspection RPCs
- Core message types for requests, reproducibility controls, audio chunks, metadata, and errors

---

## 2. Protobuf File: `voxspeak/v1/voxspeak.proto`

```proto
syntax = "proto3";

package voxspeak.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";

option go_package = "voxspeak/v1;voxspeakv1";

// VoxSpeakService exposes the VoxSpeak subsystem over gRPC.
// Transport baseline: VoxSpeak-TRANSPORT-v1.3 (Approved).
service VoxSpeakService {
  // ----------------------
  // Control-plane
  // ----------------------
  rpc ValidateSynthesis(ValidateSynthesisRequest) returns (ValidateSynthesisResponse);
  rpc LintPersonalities(LintPersonalitiesRequest) returns (LintPersonalitiesResponse);
  rpc ReloadPersonalities(ReloadPersonalitiesRequest) returns (ReloadPersonalitiesResponse);

  // ----------------------
  // Introspection
  // ----------------------
  rpc ListPersonalities(ListPersonalitiesRequest) returns (ListPersonalitiesResponse);
  rpc DescribePersonality(DescribePersonalityRequest) returns (DescribePersonalityResponse);

  rpc ListEngines(ListEnginesRequest) returns (ListEnginesResponse);
  rpc EngineCapabilities(EngineCapabilitiesRequest) returns (EngineCapabilitiesResponse);
  // Publish server-global planning/admission limits for client preflight.
  rpc GetGlobalSynthesisLimits(GetGlobalSynthesisLimitsRequest) returns (GetGlobalSynthesisLimitsResponse);

  // ----------------------
  // Session-based synthesis (required)
  // ----------------------
  rpc CreateSynthesisSession(CreateSynthesisSessionRequest) returns (CreateSynthesisSessionResponse);

  // Windows consumers subscribe to session streams.
  rpc SubscribeSynthesisSession(SubscribeSynthesisSessionRequest) returns (stream StreamMessage);

  // Optional lifecycle helpers.
  rpc CancelSynthesisSession(CancelSynthesisSessionRequest) returns (CancelSynthesisSessionResponse);
  rpc GetSessionStatus(GetSessionStatusRequest) returns (GetSessionStatusResponse);

  // ----------------------
  // Convenience: direct streaming (optional server implementation)
  // ----------------------
  rpc StreamSynthesis(StreamSynthesisRequest) returns (stream StreamMessage);
}

// ----------------------
// Common headers
// ----------------------

message RequestHeader {
  // Caller-supplied correlation id; if empty, server may generate.
  string request_id = 1;

  // Client API/protocol version (e.g., "1.4"). Optional.
  string api_version = 2;

  // Optional requester role hint (e.g., "voxthink", "windows").
  string requester_role = 3;
}

message ResponseHeader {
  // Correlation id for the request.
  string request_id = 1;

  // Server-reported API/protocol version.
  string api_version = 2;

  // Server implementation identifier (optional).
  string server_id = 3;

  // Version-policy metadata for negotiation transparency.
  string default_api_version = 4;
  repeated string supported_api_versions = 5;
  string version_policy_mode = 6;
  string effective_api_version = 7;
}

// ----------------------
// Audio / streaming types
// ----------------------

enum SampleFormat {
  SAMPLE_FORMAT_UNSPECIFIED = 0;
  SAMPLE_FORMAT_S16 = 1; // Signed 16-bit little-endian PCM
  SAMPLE_FORMAT_F32 = 2; // 32-bit float PCM
}

message AudioFormat {
  int32 sample_rate_hz = 1;
  int32 channels = 2;
  SampleFormat sample_format = 3;
}

enum StreamMode {
  STREAM_MODE_UNSPECIFIED = 0;
  STREAM_MODE_REALTIME = 1;
  STREAM_MODE_FAST_AS_POSSIBLE = 2;
}

message StreamOptions {
  StreamMode mode = 1;

  // Optional client hint; server may ignore.
  int32 chunk_duration_ms = 2;

  // Optional client hint; server may ignore.
  int32 prefetch_ms = 3;
}

message AudioChunk {
  AudioFormat format = 1;

  // Monotonically increasing index starting at 0.
  int64 index = 2;

  // Interleaved PCM frames.
  bytes pcm = 3;

  // True if this is the final chunk.
  bool is_last = 4;
}

// StreamMessage is the envelope for streaming responses.
// The stream terminates after a FinalMetadata message (or a gRPC error).
message StreamMessage {
  oneof payload {
    AudioChunk audio = 1;
    FinalMetadata final = 2;
    StreamWarning warning = 3;
  }
}

message StreamWarning {
  // Non-fatal warning code (e.g., "PSEUDO_STREAMING").
  string code = 1;

  // Human-readable message.
  string message = 2;

  // Optional structured details.
  google.protobuf.Struct details = 3;
}

message FinalMetadata {
  ResponseHeader header = 1;
  SynthesisMetadata metadata = 2;
}

// ----------------------
// Synthesis request/metadata
// ----------------------

enum OutputMode {
  OUTPUT_MODE_UNSPECIFIED = 0;
  OUTPUT_MODE_MEMORY = 1;
  OUTPUT_MODE_STREAM = 2;
  OUTPUT_MODE_FILE = 3;
}

message CachePolicy {
  bool enabled = 1;
  string backend = 2;      // e.g., "memory", "file"
  bool bypass = 3;
  string invalidate_key = 4;
  bool force_regenerate = 5;
  string invalidate_prefix = 6;
}

message FileOutputOptions {
  string output_dir = 1;
  string filename_template = 2;
  bool write_metadata_sidecar = 3;
  string container = 4;    // e.g., "wav", "flac"
}

message ReproducibilityOptions {
  int64 seed = 1;
  google.protobuf.Struct flags = 2;
}

message SynthesisRequest {
  // Raw text to synthesize.
  string text = 1;

  // Personality identifier (e.g., "ship_ai").
  string personality_id = 2;

  // Optional engine override (e.g., "piper").
  string engine_id = 3;

  OutputMode output_mode = 4;

  // Optional requested output audio format.
  AudioFormat audio_format = 5;

  bool enable_postprocessing = 6;
  bool enable_qc = 7;

  CachePolicy cache_policy = 8;

  // Optional reproducibility controls; server applies best-effort semantics.
  ReproducibilityOptions reproducibility = 11;

  // Optional queue-priority class; server applies default when omitted.
  string priority = 12;

  // Required iff output_mode == OUTPUT_MODE_FILE.
  FileOutputOptions file_output = 9;

  // Required iff output_mode == OUTPUT_MODE_STREAM.
  StreamOptions stream_options = 10;

  // Optional request-level override for verbose/debug trace logging; when unset, server/client defaults apply.
  optional bool verbose_debug_logging = 13;
}

message TimingsMs {
  // Stage timings in milliseconds. Fields are optional and may be zero.
  int64 total_ms = 1;
  int64 resolve_personality_ms = 2;
  int64 text_preprocess_ms = 3;
  int64 cache_lookup_ms = 4;
  int64 synthesize_ms = 5;
  int64 postprocess_ms = 6;
  int64 qc_ms = 7;
}

message CacheInfo {
  bool enabled = 1;
  bool hit = 2;
  string key = 3;
  string backend = 4;
  bool bypass = 5;
  bool force_regenerate = 6;
  bool effective = 7;
  string warning = 8;
  bool invalidated = 9;
  int32 invalidated_count = 10;
}

message TextProcessingTraceEntry {
  string rule_id = 1;
  string input_fragment = 2;
  string output_fragment = 3;
  string warning_level = 4;
}

message SynthesisMetadata {
  google.protobuf.Timestamp timestamp_utc = 1;

  string original_text = 2;
  string processed_text = 3;

  string personality_id = 4;
  string engine_id = 5;
  string model_id = 6;

  // Resolved synthesis params and post-processing chain; schema left flexible.
  google.protobuf.Struct synthesis_params = 7;
  google.protobuf.Struct postprocessing_chain = 8;

  // Optional QC results.
  google.protobuf.Struct qc_results = 9;

  CacheInfo cache = 10;
  TimingsMs timings_ms = 11;

  // e.g., pseudo-streaming fallback.
  repeated string warnings = 12;

  // Optional preprocessing trace for diagnostics/reproducibility audits.
  repeated TextProcessingTraceEntry text_processing_trace = 13;

  // Requested and effective reproducibility settings.
  ReproducibilityOptions requested_reproducibility = 14;
  ReproducibilityOptions effective_reproducibility = 15;
  bool reproducibility_fully_supported = 16;
}

// ----------------------
// Control-plane RPCs
// ----------------------

enum DiagnosticSeverity {
  DIAGNOSTIC_SEVERITY_UNSPECIFIED = 0;
  DIAGNOSTIC_SEVERITY_INFO = 1;
  DIAGNOSTIC_SEVERITY_WARNING = 2;
  DIAGNOSTIC_SEVERITY_ERROR = 3;
}

message ValidationDiagnostic {
  DiagnosticSeverity severity = 1;
  string code = 2;
  string message = 3;
  google.protobuf.Struct details = 4;
}

message ValidateSynthesisRequest {
  RequestHeader header = 1;
  SynthesisRequest request = 2;
}

message ValidateSynthesisResponse {
  ResponseHeader header = 1;
  bool ok = 2;
  repeated ValidationDiagnostic diagnostics = 3;
  google.protobuf.Struct resolved_config = 4;
}

message LintPersonalitiesRequest {
  RequestHeader header = 1;
}

message LintPersonalitiesResponse {
  ResponseHeader header = 1;
  repeated ValidationDiagnostic diagnostics = 2;
}

message ReloadPersonalitiesRequest {
  RequestHeader header = 1;
}

message ReloadPersonalitiesResponse {
  ResponseHeader header = 1;
  bool reloaded = 2;
  repeated ValidationDiagnostic diagnostics = 3;
}

// ----------------------
// Session-based RPCs
// ----------------------

message CreateSynthesisSessionRequest {
  RequestHeader header = 1;

  // Synthesis request. For session-based flow, output_mode should typically be STREAM.
  // Server may override or validate.
  SynthesisRequest request = 2;

  // Optional: declare intended consumer platform.
  string consumer_platform = 3; // e.g., "windows"

  // Optional: declare intended consumer role; server may enforce policy.
  string consumer_role = 4;
}

message CreateSynthesisSessionResponse {
  ResponseHeader header = 1;

  // Opaque session identifier.
  string session_id = 2;

  // Session expiration.
  google.protobuf.Timestamp expires_at = 3;

  // Optional scoped token for subscription authorization.
  string consumer_token = 4;

  // Immediate metadata may be returned if synthesis completed early (optional).
  SynthesisMetadata early_metadata = 5;
}

message SubscribeSynthesisSessionRequest {
  RequestHeader header = 1;
  string session_id = 2;
  string consumer_token = 3;

  // Optional: consumer may request a specific format; server may ignore.
  AudioFormat desired_audio_format = 4;
}

message CancelSynthesisSessionRequest {
  RequestHeader header = 1;
  string session_id = 2;
}

message CancelSynthesisSessionResponse {
  ResponseHeader header = 1;
  bool cancelled = 2;
}

enum SessionState {
  SESSION_STATE_UNSPECIFIED = 0;
  SESSION_STATE_QUEUED = 1;
  SESSION_STATE_RUNNING = 2;
  SESSION_STATE_COMPLETED = 3;
  SESSION_STATE_CANCELLED = 4;
  SESSION_STATE_FAILED = 5;
  SESSION_STATE_EXPIRED = 6;
}

message GetSessionStatusRequest {
  RequestHeader header = 1;
  string session_id = 2;
}

message GetSessionStatusResponse {
  ResponseHeader header = 1;
  string session_id = 2;
  SessionState state = 3;

  // Optional progress indicator (0-100). Semantics are best-effort.
  int32 progress_percent = 4;

  // If completed/failed, may include final metadata.
  SynthesisMetadata metadata = 5;
}

// Direct streaming convenience (optional).
message StreamSynthesisRequest {
  RequestHeader header = 1;
  SynthesisRequest request = 2;
}

// ----------------------
// Introspection RPCs
// ----------------------

message ListPersonalitiesRequest {
  RequestHeader header = 1;
}

message PersonalitySummary {
  string personality_id = 1;
  string display_name = 2;
  string description = 3;
}

message ListPersonalitiesResponse {
  ResponseHeader header = 1;
  repeated PersonalitySummary personalities = 2;
}

message DescribePersonalityRequest {
  RequestHeader header = 1;
  string personality_id = 2;

  // Optional engine context for resolved view.
  string engine_id = 3;
}

message DescribePersonalityResponse {
  ResponseHeader header = 1;

  // Raw config and resolved config are flexible blobs.
  google.protobuf.Struct raw_config = 2;
  google.protobuf.Struct resolved_config = 3;
}

message ListEnginesRequest {
  RequestHeader header = 1;
}

message EngineSummary {
  string engine_id = 1;
  string display_name = 2;
}

message ListEnginesResponse {
  ResponseHeader header = 1;
  repeated EngineSummary engines = 2;
}

message EngineCapabilitiesRequest {
  RequestHeader header = 1;
  string engine_id = 2;
}

message EngineCapabilitiesResponse {
  ResponseHeader header = 1;
  EngineCapabilities capabilities = 2;
}

// Request server-global planning/admission limits for client preflight.
message GetGlobalSynthesisLimitsRequest {
  RequestHeader header = 1;
}

// Published server-global limits. These values are not engine-local capability fields.
message GlobalSynthesisLimits {
  int32 max_synthesis_duration_ms = 1;
  int32 global_max_concurrent_jobs = 2;
  int32 global_max_queued_jobs = 3;
  map<string, int32> per_engine_max_concurrent_jobs = 4;
  map<string, int32> per_engine_max_queued_jobs = 5;
}

// Response envelope for published server-global planning/admission limits.
message GetGlobalSynthesisLimitsResponse {
  ResponseHeader header = 1;
  GlobalSynthesisLimits limits = 2;
}

message EngineCapabilities {
  string engine_id = 1;

  bool supports_streaming = 2;
  bool supports_markup = 3;

  // Declared supported sample formats/rates.
  repeated SampleFormat supported_sample_formats = 4;
  repeated int32 supported_sample_rates_hz = 5;

  // Parameter hints/ranges as a flexible blob.
  google.protobuf.Struct parameter_ranges = 6;

  // Operational limits.
  int32 max_text_length = 7;
  int32 max_concurrent_jobs = 8;
}
```

---

## 3. Notes and Expected Follow-ons

3.1. gRPC error mapping (status codes + rich details) should be implemented using standard gRPC status details; this schema includes `StreamWarning` only for non-fatal in-band warnings.

3.2. If you later introduce an audio codec on the wire (e.g., Opus), add a `Codec` enum and codec-specific payloads under `AudioChunk` with version gating.

3.3. Authentication can be layered using gRPC interceptors; `consumer_token` is optional and intended for session-scoped authorization.

---

End of Document
