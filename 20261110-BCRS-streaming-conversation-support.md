# Proposal: Streaming support for the Dapr Conversation API (Alpha3)

- Author(s): Whit Waldo (@whitwaldo)
- State: Draft
- Updated: 2026-11-12

## Summary

Add a server-streaming `ConverseStreamAlpha3` RPC and a `POST /v1.0-alpha3/conversation/{name}/converse?stream=true` SSE endpoint to the Dapr Conversation building block, so that an app receives partial completions as the LLM produces them instead of one blocking response. The Alpha2 API ([reference](https://docs.dapr.io/reference/api/conversation_api/)) returns a complete `ConversationResponseAlpha2` only after the provider finishes; the proto itself leaves `StreamOptions` as a TODO in `ConversationRequestAlpha2` ([ai.proto](https://github.com/dapr/dapr/blob/master/dapr/proto/runtime/v1/ai.proto)).

This proposal adds an incremental delivery mechanism to the existing Conversation API. It preserves request, response, tool-calling, and context semantics. Content events support append-only deltas and replace-whole snapshots so autoregressive and diffusion models can share the same API and reconstruct the existing response shape.

The stream is a sequence of `ConversationStreamResponseAlpha3` events. Each event carries a chunk whose content is either a delta (append, the autoregressive case: Claude, GPT, Gemini, Mistral) or a snapshot (replace, the diffusion case; the open-weight text model [LLaDA-8B-Instruct](https://huggingface.co/GSAI-ML/LLaDA-8B-Instruct), with a proposed adapter exposing its intermediate denoising states as full-text snapshots, is the example used throughout). Tool-call arguments, usage, finish reason and the existing response's optional `context_id` are delivered as typed events, so SDKs can reconstruct a field-equivalent `ConversationResponseAlpha2`-shaped result from the chunks, except for usage explicitly omitted by the client. The non-streaming `ConverseAlpha3` RPC keeps the Alpha2 request and response shapes and behaviour unchanged apart from the suffix, so existing SDK code migrates by renaming.

The runtime work follows the pattern Dapr already uses for streaming: pub/sub subscription streams, workflow dispatch and the 1.18 `SubscribeActorEvents` stream that [Dapr.Actors.Next](https://docs.dapr.io/developing-applications/sdks/dotnet/dotnet-actors-next/) is built on. Component authors implement one new optional interface, `ConverseStream`, and the runtime falls back to delivering a buffered response as events for components that do not.

## Background: Alpha2 today

Alpha2 is a single unary call in both transports, so the sidecar must buffer the whole provider response before it can answer. Over HTTP the app calls `POST /v1.0-alpha2/conversation/{llm-name}/converse` and gets one JSON body back; over gRPC it calls `ConverseAlpha2(ConversationRequestAlpha2) returns (ConversationResponseAlpha2)`. The request already carries everything a streaming call needs (`inputs`, `tools`, `tool_choice`, `response_format`, `temperature`, `max_tokens`, `scrub_pii`, `metadata`, `prompt_cache_retention`); only the response path is the problem.

The Alpha2 response shape, which the stream must be able to reconstruct:

| Alpha2 field | Type | Where it comes from in the provider stream |
| --- | --- | --- |
| `outputs[].choices[].index` | int64 | Present on every chunk (OpenAI `choices[].index`; Anthropic has one choice) |
| `outputs[].choices[].message.content` | string | Accumulated from text deltas, or the last snapshot for diffusion |
| `outputs[].choices[].message.tool_calls[]` | `ConversationToolCalls` | Accumulated per tool-call index from argument fragments (OpenAI `tool_calls[].function.arguments` deltas; Anthropic `input_json_delta`) |
| `outputs[].choices[].finish_reason` | string | Last chunk for the choice (`stop`, `length`, `tool_calls`, `content_filter`) |
| `outputs[].model` | string | Per-output metadata (`model` field or Anthropic `message_start`) |
| `outputs[].usage` | `ConversationResultAlpha2CompletionUsage` | Final chunk (OpenAI with `stream_options.include_usage`; Anthropic `message_delta.usage`) |
| `context_id` | string | Existing Conversation response semantics, unchanged; delivered in `done` if present |

Three existing behaviors need explicit treatment on the streaming delivery path:

1. **PII scrubbing runs on the complete response.** A regex over a half-arrived token cannot see an email address split across two chunks. When `scrub_pii` is enabled, the initial implementation buffers the complete response and applies the existing scrubber before delivering response data.
2. **Resiliency wraps the whole call.** The [API reference](https://docs.dapr.io/reference/api/conversation_api/) says retries re-run the entire Converse invocation and the timeout bounds the request context. The existing timeout continues to bound the entire operation; transparent retries stop once a response event has been sent to the app.
3. **The HTTP handler returns one JSON body.** The new streaming branch adds `text/event-stream` delivery; the unary status code and response semantics remain unchanged.

The proto comments in `ai.proto` note that the Alpha2 field grouping (`inputs[].messages[]`) and the unused `parameters`/`context_id` fields are to be revisited when the API goes stable. This proposal keeps the Alpha2 grouping unchanged so that Alpha3 is additive; the flattening is left for the stable (`v1.0`) cut.

## Goals and non-goals

**Goals**

- Stream text, tool-call arguments, finish reason and usage from conversation components to the app over gRPC (server streaming) and HTTP (SSE), without full-response buffering when native streaming, the requested content mode, and existing scrubbing requirements permit it.
- One wire format that is correct for autoregressive providers (append-only deltas) and diffusion providers (replace-whole snapshots), chosen by the component per chunk, so an app can render either without knowing the model family.
- Chunks that fold into the Alpha2 response shape, so an SDK can offer `ConverseStream` and `Converse` over the same response types without changing the existing unary implementation.
- Preserve `scrub_pii` and resiliency policy semantics, with explicit buffering and retry limitations on the streaming path; retain tracing and metrics.
- Additive: Alpha2 endpoints and messages are untouched; Alpha3 is a new RPC pair and a new URL version. Components without streaming support still work through a runtime fallback.

**Non-goals**

- Bidirectional streaming (app sends messages while the model is generating) and audio/voice; the request stays unary. Session APIs belong in a separate proposal.
- Streaming of reasoning/thinking content as a first-class part. Alpha2 has no reasoning content field; reasoning APIs belong in a separate proposal.
- Changing the Alpha2 request grouping (`inputs[].messages[]`), `parameters` or `context_id` semantics. Those are stable-cut concerns called out in `ai.proto`.
- Changing unary HTTP status codes, context persistence, PII detection, resiliency configuration, or Alpha2's deprecation schedule.
- Introducing early tool execution or changing tool validation and execution responsibilities.
- Guaranteeing identical token boundaries across providers. Chunk size is provider-defined; only the folded result is normative.
- Adding a conversation component or inference adapter for a diffusion model. LLaDA-8B-Instruct is the concrete example because its official implementation demonstrates masked-text denoising states; exposing those states during generation as full-text snapshots requires a separate adapter. Neither that adapter nor a component for it is committed to here.
- Partial-revision updates, such as a stable-prefix offset that lets a block-diffusion model re-send only the unsettled tail. `REPLACE` re-sends the whole text; a finer-grained update mode can be added later under the opt-in rule in the design considerations.

## Design considerations

The central design decision is that a content chunk carries an explicit update mode, `APPEND` or `REPLACE`, rather than assuming deltas. Everything else follows from reconciling two existing provider stream styles and a proposed diffusion snapshot source into one event sequence. The LLaDA column describes adapter requirements, not an existing hosted API contract.

| Concern | Autoregressive via OpenAI-style SSE (GPT, Mistral) | Anthropic Messages SSE (Claude) | Diffusion snapshots (proposed LLaDA-8B-Instruct adapter) | Alpha3 mapping |
| --- | --- | --- | --- | --- |
| Text | `choices[].delta.content` fragments, append | `content_block_delta` with `text_delta`, append | adapter emits full response-text snapshots from denoising states ([official visualization code](https://github.com/ML-GSAI/LLaDA/blob/main/app.py)), then the final decoded text | `ContentChunk{update_mode: APPEND\|REPLACE, text}` |
| Tool-call args | `delta.tool_calls[i].function.arguments` string fragments keyed by index | `content_block_start` (id, name) then `input_json_delta` partial JSON | not demonstrated by the reference implementation; no tool-call support assumed | `ToolCallChunk{index, id?, name?, arguments, complete}`, append-only |
| Multiple choices | `choices[].index`, interleaved | one choice | one choice in this adapter example | `choice_index` on every chunk; default 0 |
| Finish reason | last chunk per choice, `finish_reason` | `message_delta.stop_reason` (`end_turn`, `max_tokens`, `tool_use`) | adapter maps actual termination after final text; no native finish-reason event assumed | `FinishChunk{finish_reason}` normalised to Alpha2 strings |
| Usage | final chunk only if `stream_options.include_usage` | `message_start.usage` (input) + `message_delta.usage` (output) | only if the adapter supplies authoritative token counts; otherwise omitted | `UsageChunk` at most once per output, before `done` |
| Model name | every chunk | `message_start.message.model` | adapter supplies the loaded model ID at output start | `OutputStart{output_index, model}` once per output |
| Errors mid-stream | HTTP 200 already sent; error as SSE `error` event or dropped connection | `error` SSE event | adapter propagates inference failures; no upstream SSE error format assumed | gRPC: status after partial chunks; HTTP: `error` SSE event then close |

**Why `REPLACE` is a mode and not a separate RPC.** A diffusion serving implementation may expose finalized blocks as ordinary `APPEND` content or intermediate denoising states as full-text snapshots. These are serving capabilities, not guarantees implied by the model family. Making the mode per chunk lets one component emit either, lets the runtime fold both into the same Alpha2 result (`REPLACE` overwrites the accumulator; `APPEND` concatenates), and lets an SDK expose both through one iterator. Requesting replacements splits tolerance from preference so generic clients do not accidentally trigger expensive full-text snapshots on dual-mode implementations: `ALLOW_REPLACE` tells the runtime and component that the client can render `REPLACE` chunks without buffering if a provider naturally emits them, while dual-mode providers keep their default `APPEND` stream; `PREFER_REPLACE` additionally permits a component to opt into snapshot production, such as enabling per-step state emission in a future LLaDA adapter. This is not a standard LLaDA request flag. Clients that only want incremental text keep the default `APPEND`.

**Append-only is a strict delivery guarantee.** With `content_mode = APPEND` (the default), every emitted content event MUST be `APPEND`. Components should select native append streaming where available. Otherwise, the runtime buffers the choice's revisable content until it finishes and emits the final text as one `APPEND`. It cannot safely turn arbitrary snapshots into incremental deltas: `"cat"` followed by `"car"` requires revising already-sent text, and a common prefix between consecutive snapshots does not guarantee future stability. This buffering is a documented delivery limitation, not a change to the final response.

**Intermediate snapshots are not authoritative.** An `APPEND` stream that stops early leaves a valid prefix of the answer; a `REPLACE` stream does not. An intermediate snapshot may contain placeholder or later-revised text, and with `response_format` it is not a prefix of the final JSON. Only the content in effect when the choice's `finish` arrives is the model's answer, and components MUST ensure the last content chunk before `finish` leaves the final text in effect. Whereas an empty `APPEND` delta is a no-op, a `REPLACE` chunk with `text = ""` is meaningful—for example, when a diffusion model denoises noisy text down to empty content while pivoting to `tool_calls`—and resets the accumulated choice text to `""`. Because protobuf JSON omits empty strings, normalisers, SSE readers, and SDK fold helpers MUST NOT ignore or drop `REPLACE` events whose `text` field is empty or omitted. After an error, cancellation, or EOF without `done`, clients should discard snapshot content rather than present it as a partial answer.

**Update modes are negotiated, never assumed.** An unknown event kind can be skipped; an unknown update mode cannot, because the client would no longer know the accumulated content. So the runtime never emits `UNSPECIFIED`, a client that receives an unspecified or unrecognized update mode MUST fail the stream rather than guess, and any update mode added later is emitted only to clients that opt in through `content_mode`. Tool-call arguments are append-only for the same reason: no provider mapped here revises arguments, and an append-only client must never receive a replacement of any kind.

**Why folding must be lossless.** Streaming is another delivery format for the existing response, not a different Conversation result. For a fixed provider response, folding its events must reproduce the existing response fields after existing response processing, except for usage explicitly omitted by the client. This is field-level equivalence, not byte-identical serialization or an expectation that two independent model generations return identical text.

**Why usage and finish are separate events, not fields on every chunk.** The component combines provider usage updates into one authoritative usage value per output. Each choice has its own finish event. Separate events avoid repeating final metadata on content chunks and preserve the existing per-output and per-choice response structure.

**Tool calls and the agent loop.** Streaming does not change the Alpha2 tool-calling contract: the app still executes the tool and sends an `ofTool` message in the next request. `ToolCallChunk.complete` marks the last fragment for an index, not permission to execute a tool or a new guarantee of valid JSON. SDKs assemble the existing tool-call representation; the app uses its existing validation and execution path after successful stream completion. Early tool execution is outside this proposal.

**Backpressure.** gRPC provides transport flow control; the runtime must also bound queued component events. For SSE, if the app reads slowly the sidecar buffers up to `conversation.streamBufferBytes` (default 1 MiB) then cancels the provider request rather than growing the delivery queue without bound. Before it gets that far, the runtime coalesces snapshots on both transports: a newer `REPLACE` supersedes any queued, unsent content events for the same output and choice, so only the latest is kept. This is lossless for the fold, keeps a diffusion stream from disconnecting a slow reader, and limits the cost of re-sending the full text on every step. Content that no later `REPLACE` supersedes, and all tool-call fragments, are never dropped. A `RESOURCE_EXHAUSTED`-equivalent `error` event is best-effort when the client cannot read; closing without `done` still signals an incomplete response. Full-response buffering remains subject to existing Conversation response limits.

## Proposed proto changes (Alpha3)

Two RPCs are added to `Dapr` in `dapr.proto`; all message additions go in `ai.proto` with the `Alpha3` suffix. `ConversationRequestAlpha3` is `ConversationRequestAlpha2` plus one field (`stream_options`), and `ConversationResponseAlpha3` is a pure rename, so the non-streaming path is a mechanical migration.

```protobuf
// dapr.proto — service Dapr
// Non-streaming; identical semantics to ConverseAlpha2.
rpc ConverseAlpha3(ConversationRequestAlpha3) returns (ConversationResponseAlpha3) {}

// Streaming; one request, a server stream of events, ending with a `done` event
// (or a gRPC error status, which may follow partial events).
rpc ConverseStreamAlpha3(ConversationRequestAlpha3) returns (stream ConversationStreamResponseAlpha3) {}
```

```protobuf
// ai.proto — additions

// ConversationRequestAlpha3 is ConversationRequestAlpha2 with streaming options.
// Fields 1–12 are copied verbatim from ConversationRequestAlpha2 (same numbers,
// same JSON names) so that a client can reuse serialized Alpha2 requests.
message ConversationRequestAlpha3 {
  string name = 1;
  optional string context_id = 2;
  repeated ConversationInputAlpha2 inputs = 3;
  map<string, google.protobuf.Any> parameters = 4;
  map<string, string> metadata = 5;
  optional bool scrub_pii = 6;
  optional double temperature = 7;
  repeated ConversationTools tools = 8;
  optional string tool_choice = 9;
  optional google.protobuf.Struct response_format = 10;
  optional google.protobuf.Duration prompt_cache_retention = 11;
  optional int64 max_tokens = 12;

  // Only honoured by ConverseStreamAlpha3; ignored by ConverseAlpha3.
  // inspired by openai.ChatCompletionStreamOptionsParam
  optional ConversationStreamOptionsAlpha3 stream_options = 13;
}

message ConversationStreamOptionsAlpha3 {
  // How the client wants text content delivered.
  // APPEND (default, including UNSPECIFIED): every ContentChunk is APPEND.
  //   Use native append streaming where available; otherwise buffer revisable
  //   content until the choice finishes and emit its final text as one APPEND.
  // ALLOW_REPLACE: tolerance. The client can handle both APPEND and REPLACE,
  //   avoiding buffering when a provider naturally emits snapshots, without
  //   asking dual-mode providers to switch away from default APPEND streaming.
  // PREFER_REPLACE: preference and permission. The client wants snapshot
  //   updates when supported; a component MAY select a snapshot-producing
  //   provider mode (e.g. per-step denoising snapshots). Scrubbing or caching
  //   requirements may still deliver a buffered stream.
  optional ConversationContentModeAlpha3 content_mode = 1;
  // Emit a UsageChunk per output when available, before done. Default true. Maps to OpenAI
  // stream_options.include_usage; Anthropic always reports usage.
  optional bool include_usage = 2;
  // Emit a ToolCallChunk per argument fragment (true) or only once the tool call is
  // complete (false, default). Default false delivers assembled arguments.
  // Arguments are append-only in both cases.
  // Existing tool-call validation semantics remain unchanged.
  optional bool partial_tool_calls = 3;
}

enum ConversationContentModeAlpha3 {
  CONVERSATION_CONTENT_MODE_ALPHA3_UNSPECIFIED = 0;
  CONVERSATION_CONTENT_MODE_ALPHA3_APPEND = 1;
  CONVERSATION_CONTENT_MODE_ALPHA3_ALLOW_REPLACE = 2;
  CONVERSATION_CONTENT_MODE_ALPHA3_PREFER_REPLACE = 3;
}

// ConversationResponseAlpha3 is the non-streaming response; same shape as Alpha2.
message ConversationResponseAlpha3 {
  optional string context_id = 1;
  repeated ConversationResultAlpha2 outputs = 2;
}

// ConversationStreamResponseAlpha3 is one event on the ConverseStreamAlpha3 stream.
// Successful stream: exactly one `start`; one `output_start` per output before its
// choice events; content/tool_call events interleaved across choices; exactly one
// `finish` per choice; at most one `usage` per output after its choices finish;
// exactly one `done` after all outputs finish. Errors terminate without `done`.
// Clients MUST ignore event kinds they do not recognise (forward compatibility).
// Update modes are the exception; see ConversationUpdateModeAlpha3.
message ConversationStreamResponseAlpha3 {
  // Contiguous per stream, starting at 0. Assigned when the event is sent, so
  // coalesced snapshots leave no gaps. Does not imply replay or resumability.
  uint64 sequence = 1;
  oneof event {
    ConversationStreamStartAlpha3     start     = 2;
    ConversationContentChunkAlpha3    content   = 3;
    ConversationToolCallChunkAlpha3   tool_call = 4;
    ConversationFinishChunkAlpha3     finish    = 5;
    ConversationUsageChunkAlpha3      usage     = 6;
    ConversationStreamDoneAlpha3      done      = 7;
    ConversationOutputStartAlpha3    output_start = 8;
  }
}

// Emitted once when the stream produces its first usable response event.
// Carries no output count because output cardinality is determined by the
// component during generation; `output_start` and `done` frame all outputs.
message ConversationStreamStartAlpha3 {}

message ConversationOutputStartAlpha3 {
  uint32 output_index = 1;
  // The model used for this output (ConversationResultAlpha2.model).
  optional string model = 2;
  // How text content for this output is delivered. Informational: lets a UI show
  // a progress indicator instead of a typing effect. Not part of the fold.
  ConversationStreamDeliveryAlpha3 delivery = 3;
}

enum ConversationStreamDeliveryAlpha3 {
  CONVERSATION_STREAM_DELIVERY_ALPHA3_UNSPECIFIED = 0;
  // Content events are forwarded as the provider produces them.
  CONVERSATION_STREAM_DELIVERY_ALPHA3_NATIVE = 1;
  // Text is withheld until it is final, then sent as one APPEND per choice:
  // a component without streaming support, existing response processing that
  // needs the whole response, or a snapshot provider serving an append-only client.
  CONVERSATION_STREAM_DELIVERY_ALPHA3_BUFFERED = 2;
}

// Clients MUST fail the stream on an unspecified or unrecognised value rather than
// guess. New modes are only emitted to clients that opt in through content_mode.
enum ConversationUpdateModeAlpha3 {
  // Never emitted.
  CONVERSATION_UPDATE_MODE_ALPHA3_UNSPECIFIED = 0;
  // text is a fragment to concatenate onto the accumulated content.
  CONVERSATION_UPDATE_MODE_ALPHA3_APPEND = 1;
  // text is the complete content so far and replaces the accumulated content
  // (diffusion providers re-sending the denoised sequence).
  CONVERSATION_UPDATE_MODE_ALPHA3_REPLACE = 2;
}

message ConversationContentChunkAlpha3 {
  // Which output this belongs to (ConversationResponseAlpha2.outputs[i]).
  uint32 output_index = 1;
  // Which choice within the output (ConversationResultChoices.index, widened to
  // int64 when folded). 32-bit so it serialises as a JSON number.
  uint32 choice_index = 2;
  ConversationUpdateModeAlpha3 update_mode = 3;
  // Text fragment (APPEND) or full text (REPLACE). Mirrors ConversationMessageContent.text.
  // For REPLACE, an empty string (omitted in protobuf JSON) is valid and resets
  // the accumulated content for this choice to "".
  string text = 4;
}

// One fragment of a tool call. Folds into ConversationResultMessage.tool_calls[index].
// inspired by openai.ChatCompletionChunkChoiceDeltaToolCall and Anthropic input_json_delta
message ConversationToolCallChunkAlpha3 {
  uint32 output_index = 1;
  uint32 choice_index = 2;
  // Index of the tool call within the message; stable across fragments.
  uint32 index = 3;
  // Provider tool-call id; set on the first fragment for this index.
  optional string id = 4;
  oneof tool_types {
    ConversationToolCallChunkOfFunctionAlpha3 function = 5;
  }
  // True on the last fragment for this index; no further fragments follow.
  // The folded arguments retain existing validation semantics; this flag alone
  // does not guarantee valid JSON or authorize execution.
  bool complete = 6;
}

message ConversationToolCallChunkOfFunctionAlpha3 {
  // Set on the first fragment for this index.
  optional string name = 1;
  // Fragment of the JSON arguments string, concatenated in order. Arguments are
  // append-only; there is no replace mode for tool calls.
  string arguments = 2;
}

// Emitted once per choice when the provider stops generating for it.
message ConversationFinishChunkAlpha3 {
  uint32 output_index = 1;
  uint32 choice_index = 2;
  // Normalised to the Alpha2 set: "stop", "length", "tool_calls", "content_filter".
  // Anthropic end_turn/stop_sequence -> stop, max_tokens -> length, tool_use -> tool_calls.
  string finish_reason = 3;
}

message ConversationUsageChunkAlpha3 {
  uint32 output_index = 1;
  ConversationResultAlpha2CompletionUsage usage = 2;
}

message ConversationStreamDoneAlpha3 {
  // Existing Conversation response context_id, if present; no new persistence semantics.
  optional string context_id = 1;
}
```

**Fold rule (normative).** Given the ordered events of one successful stream, the folded `ConversationResponseAlpha3` is: `context_id` from `done`; for each `output_index`, `model` from its `output_start`, `usage` from its optional `usage` event, and one `ConversationResultChoices` per `choice_index` whose `message.content` is the concatenation of `APPEND` text chunks, overwritten by any `REPLACE` chunk (including `text = ""`, which resets the accumulated content to `""`), whose `message.tool_calls` are assembled per tool-call `index` by concatenating argument fragments, and whose `finish_reason` comes from its `finish` event. Accumulators are keyed by output, choice, and, where applicable, tool-call index, so interleaving cannot mix results. Output and tool-call arrays retain their indexed order. `delivery` is informational and is not part of the folded response. With `include_usage = false`, usage is intentionally absent; otherwise, preserve available usage without fabricating values when a provider omits it.

**Successful termination (normative).** Each output begins with `output_start`; each choice finishes exactly once, after all its content and finalized tool calls, and accepts no further content or tool-call updates. Usage, if present, follows all finishes for its output. `done` is emitted only after every output and existing response processing complete successfully. Any output failure terminates the request without `done`; the API does not introduce partial-success responses. SDK folding requires `done` and successful transport termination (including an OK gRPC status). EOF without `done` or a transport error is an incomplete response, even if some choices finished.

Conformance uses fixed provider-response fixtures and their expected existing response fields. Independent live generations, even with temperature zero and a seed, are not required to produce identical text.

**Why not add `stream` to `ConversationRequestAlpha2` and return `stream ConversationResponseAlpha2`?** gRPC cannot vary the response streaming-ness by a request field, and reusing `ConversationResponseAlpha2` as the chunk type (OpenAI's approach, where a chunk looks like a completion with `delta` instead of `message`) would force every chunk to carry empty `choices[].message` and make the diffusion `REPLACE` mode an out-of-band convention. The explicit event `oneof` is longer but self-describing, and it is the same style as Dapr's `SubscribeTopicEventsResponseAlpha1` and `SubscribeActorEvents` streams.

## Proposed HTTP API changes

One new URL version, one endpoint, and streaming selected per request. Alpha3 JSON uses the protobuf JSON mapping (camelCase field names, full enum names, and strings for 64-bit integers). The new stream messages use 32-bit indices so they serialise as JSON numbers; only the reused Alpha2 usage counters are 64-bit strings. In the prose, enum values are abbreviated to `APPEND`, `REPLACE`, `ALLOW_REPLACE`, `PREFER_REPLACE`, `NATIVE`, and `BUFFERED`.

```
POST http://localhost:<daprPort>/v1.0-alpha3/conversation/<llm-name>/converse
POST http://localhost:<daprPort>/v1.0-alpha3/conversation/<llm-name>/converse?stream=true
```

| Aspect | Non-streaming | Streaming (`?stream=true`, or `Accept: text/event-stream`) |
| --- | --- | --- |
| Request body | `ConversationRequestAlpha3` JSON; `streamOptions` ignored | `ConversationRequestAlpha3` JSON; `streamOptions` honoured |
| Success status | `200 OK`, unchanged from Alpha2's `UniversalHTTPHandler` implementation (the Alpha2 docs table lists `202`, which does not match runtime behaviour) | `200 OK`, `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `X-Accel-Buffering: no` |
| Body | `ConversationResponseAlpha3` JSON | SSE; `event:` = the `oneof` case name, `data:` = the selected event payload's protobuf JSON (not the enclosing response), `id:` = decimal `sequence` |
| Errors before the stream opens | `400` / `500` as Alpha2 | request validation and component resolution failures: same status codes, JSON error body, no SSE |
| Errors after the stream opens | n/a | `event: error` with a Dapr error JSON (`errorCode`, `message`), then the connection closes. Includes a provider failure before any response event, once retries are exhausted |
| Keep-alive | n/a | `: ping` comment every 15 s from the moment the stream opens, including while the sidecar waits on the provider, retries, or buffers |

The stream opens, meaning `200` and the SSE headers are flushed, as soon as the request is validated and the component resolved, before the provider is invoked. Keep-alive comments can then flow during a long wait, at the cost that provider failures are reported in-band rather than as an HTTP status.

**Streaming response example** (Claude-style, append):

```text
id: 0
event: start
data: {}

id: 1
event: output_start
data: {"outputIndex":0,"model":"claude-sonnet-4-6","delivery":"CONVERSATION_STREAM_DELIVERY_ALPHA3_NATIVE"}

id: 2
event: content
data: {"outputIndex":0,"choiceIndex":0,"updateMode":"CONVERSATION_UPDATE_MODE_ALPHA3_APPEND","text":"Dapr is a"}

id: 3
event: content
data: {"outputIndex":0,"choiceIndex":0,"updateMode":"CONVERSATION_UPDATE_MODE_ALPHA3_APPEND","text":" portable runtime"}

id: 4
event: finish
data: {"outputIndex":0,"choiceIndex":0,"finishReason":"stop"}

id: 5
event: usage
data: {"outputIndex":0,"usage":{"promptTokens":"12","completionTokens":"8","totalTokens":"20"}}

id: 6
event: done
data: {}
```

Default-valued fields are shown for clarity; protobuf JSON serializers may omit them. `done` carries `contextId` only if the existing Conversation response supplies one. SSE clients reconstruct the response envelope from `id`, `event`, and `data`; these IDs do not support replay or resuming an interrupted generation.

**Streaming response excerpt** (illustrative output from the proposed LLaDA-8B-Instruct snapshot adapter, not a captured model response; `___` represents masked positions for readability; replace; client sent `streamOptions.contentMode: "CONVERSATION_CONTENT_MODE_ALPHA3_PREFER_REPLACE"`; after `start` and `output_start`):

```text
id: 2
event: content
data: {"outputIndex":0,"choiceIndex":0,"updateMode":"CONVERSATION_UPDATE_MODE_ALPHA3_REPLACE","text":"Dapr ___ a portable ___ for ___ applications"}

id: 3
event: content
data: {"outputIndex":0,"choiceIndex":0,"updateMode":"CONVERSATION_UPDATE_MODE_ALPHA3_REPLACE","text":"Dapr is a portable runtime for distributed applications"}
```

With the default `contentMode` (`APPEND`), the component selects native append streaming where available. If it instead produces the snapshots above, the sidecar withholds revisable text until the choice finishes, then emits `"Dapr is a portable runtime for distributed applications"` as one `APPEND` content event before `finish`, and reports `delivery: BUFFERED` on `output_start`. It never sends `REPLACE` to an append-only client.

**Tool-call streaming excerpt** (`streamOptions.partialToolCalls: true`; after `start` and `output_start`):

```text
id: 2
event: tool_call
data: {"outputIndex":0,"choiceIndex":0,"index":0,"id":"call_Uwa41","function":{"name":"get_weather","arguments":"{\"location\":"},"complete":false}

id: 3
event: tool_call
data: {"outputIndex":0,"choiceIndex":0,"index":0,"function":{"arguments":"\"San Francisco, CA\"}"},"complete":true}

id: 4
event: finish
data: {"outputIndex":0,"choiceIndex":0,"finishReason":"tool_calls"}
```

**Request-side additions** to the reference table:

| Field | Description |
| --- | --- |
| `streamOptions.contentMode` | `APPEND` (default), `ALLOW_REPLACE`, or `PREFER_REPLACE`, using the full protobuf enum name in JSON. `APPEND` never emits replacements; snapshot-only content is buffered until final. `ALLOW_REPLACE` accepts replacements without asking dual-mode providers to switch away from `APPEND`. `PREFER_REPLACE` additionally permits the component to select a snapshot-producing provider mode. Optional |
| `streamOptions.includeUsage` | Emit a `usage` event per output when available. Default `true`; `false` intentionally omits usage from the folded result. Optional |
| `streamOptions.partialToolCalls` | Emit `tool_call` events per argument fragment instead of once complete. Default `false`. Optional |

Query-string metadata (`?metadata.user_id=…`) and the Alpha2 precedence rules apply unchanged. Either `Accept: text/event-stream` or `?stream=true` selects streaming on this POST endpoint. An explicit `stream` query parameter takes precedence over `Accept`: `?stream=false` returns the unary response even when the client accepts `text/event-stream`, and any value other than `true` or `false` is a `400`. Browser clients use `fetch` with a request body and an SSE reader, or an SDK; native `EventSource` cannot issue this POST. No GET endpoint or automatic reconnection is introduced.

## Runtime changes in dapr/dapr

The sidecar gains one streaming delivery pipeline shared by both transports: component events pass through normalization, any required buffering and existing response processing, then leave as gRPC messages or SSE frames. Fold helpers preserve the existing response shape. `ConverseAlpha3` retains the existing unary invocation path; implementing unary calls through streaming is not required by this proposal.

**gRPC (`pkg/api/grpc`)**

- Add `ConverseAlpha3` and `ConverseStreamAlpha3` handlers. The stream handler validates the request, resolves the component, and runs the pipeline, calling `stream.Send` per event. Context cancellation from the client cancels the provider request.
- Return gRPC status errors after partial output when the provider fails mid-stream; the trailing status carries the Dapr error code (`ERR_CONVERSATION_INVOKE`) and the `sequence` of the last event sent, so the client knows what it has.

**HTTP (`pkg/api/http`)**

- Register `/v1.0-alpha3/conversation/{name}/converse`. Branch on the `stream` query parameter, falling back to `Accept: text/event-stream` when it is absent. The streaming branch validates the request and resolves the component, then commits `200` with the SSE headers before invoking the provider. It flushes after each frame (`http.Flusher`), sends `: ping` comments on a 15-second ticker from that point and stops on client disconnect (`r.Context().Done()`). Failures after the headers are committed are reported as `event: error`.
- Preserve the existing unary status code (`200 OK` via `UniversalHTTPHandler`) and response processing.

**Normaliser (`pkg/conversation/stream`)**

- Converts `components-contrib` stream items into `ConversationStreamResponseAlpha3`, sets `delivery` on each `output_start`, assigns `sequence` as each event is sent, and enforces the per-output and per-choice event grammar. The runtime emits one global `start` and emits `done` only on successful completion.
- Passes the requested content mode to streaming components. For `APPEND`, the component selects native append delivery if supported; otherwise it declares the output revisable on its start chunk, and the runtime reports `delivery = BUFFERED`, withholds that output's text until each choice's `finish`, and emits only the final text as `APPEND`. A `REPLACE` chunk from an output not declared revisable is a contract violation that fails the stream rather than sending an unexpected `REPLACE`. For `ALLOW_REPLACE` and `PREFER_REPLACE`, both update modes can be delivered (including `REPLACE` with empty `text`, which resets accumulated content to `""`) and delivery is `NATIVE` unless buffered by scrubbing or caching.
- Buffers tool-call fragments by `(output_index, choice_index, index)` until `complete` when `partial_tool_calls` is false, then emits the assembled call once. With partial delivery enabled, fragments are forwarded as they arrive; arguments are append-only and no fragments follow `complete`.
- Coalesces queued `REPLACE` content under backpressure as described in the design considerations. `sequence` stays contiguous because it is assigned at send time.

**PII scrubbing**

- When `scrub_pii` is enabled, use the existing unary invocation and full-response scrubbing path, then deliver its processed result using the same event emitter as the non-streaming-component fallback. No provider text, intermediate diffusion snapshots, or tool fragments are sent before existing response processing completes.
- This preserves the current scrubber, its field coverage, and its error behaviour. It intentionally provides no incremental response-data latency benefit while scrubbing is enabled. There is no fixed-window detector or late replacement of previously disclosed text in this proposal.
- When `scrub_pii` is false, native incremental delivery is available subject to the content-mode policy. Changing PII detection or adding safe incremental scrubbing requires separate work.

**Resiliency (`pkg/resiliency`)**

- Timeout: retain the existing outbound policy timeout and request-context deadline semantics for the entire operation. Neither stream events nor SSE keep-alive comments reset or reinterpret that timeout. This proposal adds no first-event or idle-timeout policy fields.
- Retries: existing policies apply only before the first response event, including `start`, has been sent to the app. Delay `start` until the component has produced its first usable event, or a buffered unary response is ready, so a synthetic event does not prematurely disable retries. Over HTTP the SSE headers and keep-alive comments are already flowing during this window; they are not response events and do not end it. Discard any unsent events and fold state before retrying.
- Once a response event is out, a provider failure terminates the stream through its transport's error mechanism; the circuit breaker still records the failure. There is no transparent replay of partial output.

**Context handling**

- Preserve existing `context_id` handling, including any current limitations; do not introduce persistence, append turns, or assign new IDs as part of this proposal. `done.context_id` carries only the value the existing response processing supplies.
- Where existing processing requires persistence or other finalization, it must succeed before `done` is emitted. An incomplete stream is not a completed Conversation response; no new rollback or exactly-once persistence guarantees are introduced.

**Observability**

- Metrics: `dapr_conversation_stream_count`, `dapr_conversation_stream_ttft_ms` (time to first content event, labelled by `delivery` and by the first content event's update mode, because buffered delivery makes it equal to total latency and a first diffusion snapshot is not yet readable text), `dapr_conversation_stream_events_total`, plus the existing count and latency metrics for the unary call.
- Tracing: one span per stream with events for `first_event` and `done`; `usage` token counts attached as span attributes, as the unary path does today. Token counts a provider reported before a failed or cancelled stream are recorded on the span too, since no `usage` event reaches the app in that case.

**Feature flag**

- `ConversationStreaming` preview feature gate, on by default in the release that ships it, off-able in `Configuration.spec.features`.

## Component contract (components-contrib)

Components opt in through one new interface; the runtime discovers it with a type assertion and otherwise simulates a stream from the unary `Converse`, so no existing component breaks.

```go
// conversation/conversation.go — additions

// Streamer is implemented by conversation components that can stream.
// The runtime detects it via type assertion on the Conversation instance.
type Streamer interface {
    // ConverseStream sends chunks to out until the provider finishes, then returns.
    // It MUST close nothing: the runtime owns `out`. A non-nil error after chunks
    // have been sent is reported to the app as a mid-stream error.
    ConverseStream(ctx context.Context, req *Request, opts StreamOptions, out chan<- StreamChunk) error
}

// Streaming delivery options are separate from the existing unary Request.
type StreamOptions struct {
    ContentMode  ContentMode // Append (default), AllowReplace, or PreferReplace
    IncludeUsage bool        // request usage from the provider where that is opt-in
}

type ContentMode int

const (
    ContentModeAppend        ContentMode = iota + 1 // request native append delivery
    ContentModeAllowReplace                         // accept replacements if native; keep default append on dual-mode providers
    ContentModePreferReplace                        // prefer snapshot/denoising mode when supported
)

type UpdateMode int

const (
    UpdateModeAppend  UpdateMode = iota + 1 // fragment to concatenate
    UpdateModeReplace                       // full text so far (diffusion snapshots; "" clears prior text)
)

// StreamChunk is a provider-neutral event; exactly one of the pointer fields is set.
type StreamChunk struct {
    OutputIndex int
    ChoiceIndex int
    Start    *StreamStart     // Model, Revisable; once per output, mapped to output_start
    Content  *ContentChunk    // Text + Mode
    ToolCall *ToolCallChunk   // Index, ID, Name, Arguments (append-only fragment), Complete
    Finish   *FinishChunk     // Reason, already normalised to stop|length|tool_calls|content_filter
    Usage    *Usage           // same struct the unary Response uses
}
```

`ContentMode` conveys the API's content-mode preference without altering the existing unary request. With `ContentModeAppend`, a component unable to produce native append content sets `Revisable` on the output's `StreamStart`, allowing the runtime to buffer that output before any text is exposed. With `ContentModeAllowReplace`, native snapshot outputs stream unbuffered while dual-mode providers retain their default append mode. With `ContentModePreferReplace`, a component MAY select a snapshot-producing provider mode. `IncludeUsage` is forwarded where the provider makes usage opt-in; the runtime drops any usage chunk the client declined. Text and argument fragments MUST be split on UTF-8 code-point boundaries, so a component holds back a partial multi-byte sequence until it is complete, and the last content chunk before `Finish` MUST leave the final text in effect. Components respect context cancellation, including when blocked on sending to `out`; the runtime owns global `start` and `done` events.

**Response caching (`responseCacheTTL`).** Components that enable `responseCacheTTL` (`conversation.CacheResponses`) cache only the finalized response after a stream completes successfully—never intermediate diffusion snapshots, partial chunks, or cancelled or failed streams. Cache hits replay the cached final response as a buffered stream (`delivery = BUFFERED`).

**Runtime fallback.** If the component does not implement `Streamer`, `ConverseStreamAlpha3` uses the existing unary invocation and response-processing path, then emits `start` and, for each output, `output_start` with `delivery = BUFFERED`, one `APPEND` content event per choice with the full text, the tool calls as complete `tool_call` events, `finish` per choice, and `usage` when available and requested. It emits `done` after all outputs. The app sees a valid buffered stream and `dapr_conversation_stream_simulated` is incremented so operators can tell. The `scrub_pii` path uses this same response-to-events conversion.

**Provider mapping for the components that exist today.** Most components are built on `langchaingo`, whose `GenerateContent` accepts `llms.WithStreamingFunc`; that callback delivers text deltas only, so tool-call and usage streaming need provider-native clients. The per-component plan:

| Component | Stream source | Content mode | Tool-call args | Usage | Notes |
| --- | --- | --- | --- | --- | --- |
| OpenAI, Azure OpenAI, DeepSeek, Mistral, HuggingFace (OpenAI-compatible) | Chat Completions SSE (`stream: true`, `stream_options.include_usage: true`), plus OpenAI Responses SSE (`/responses` when `useResponsesAPI: true`: `response.output_text.delta`, `response.function_call_arguments.delta`, `response.completed`) | APPEND | per-index fragments | final chunk / `response.completed` | shared Chat Completions streamer plus Responses API streamer in `conversation/openai` |
| Anthropic (Claude) | Messages SSE (`message_start`, `content_block_start`, `content_block_delta`, `message_delta`, `message_stop`) | APPEND | `input_json_delta` per content block, block index → tool-call index | `message_start` input + `message_delta` output, merged into one `Usage` | `stop_reason` mapped per the proto comment |
| Google AI (Gemini) | `streamGenerateContent` | APPEND | function-call parts arrive complete | `usageMetadata` on final chunk | emit tool calls with `Complete: true` |
| AWS Bedrock | `ConverseStream` API events | APPEND | `toolUse` input deltas | `metadata` event | native SDK already event-shaped |
| Ollama | `/api/chat` NDJSON | APPEND | arrive complete | final object | local models |
| echo (test component) | synthetic | APPEND and REPLACE | synthetic | synthetic | gains a `mode` metadata field so the conformance suite can exercise both paths |

No component is added by this proposal. [LLaDA-8B-Instruct](https://huggingface.co/GSAI-ML/LLaDA-8B-Instruct) is the concrete text-diffusion example: its [official inference code](https://github.com/ML-GSAI/LLaDA/blob/main/generate.py) fills masked positions through denoising and supports semi-autoregressive blocks, while its [visualization demo](https://github.com/ML-GSAI/LLaDA/blob/main/app.py) records intermediate response states. The demo collects those states before replaying them; it is evidence of the denoising process, not a live snapshot-streaming API. A future adapter would need to emit states during inference, render each as the entire response text (excluding the prompt and retaining placeholders for unresolved positions), and emit the final decoded text before `Finish`. Tool-call support, usage reporting, and an OpenAI-compatible SSE endpoint are not assumed.

Such an adapter's component must explicitly identify snapshots as `UpdateModeReplace`; neither a diffusion model name nor an OpenAI-compatible chunk shape proves replacement semantics. A snapshot-only adapter sets `Revisable` and relies on runtime buffering for `ContentModeAppend`, while `ContentModeAllowReplace` and `ContentModePreferReplace` permit incremental snapshots when scrubbing and caching do not require buffering. If a future adapter also supports native append delivery, it keeps that default for `ContentModeAppend` and `ContentModeAllowReplace`, selecting snapshots only for `ContentModePreferReplace`. Adding the adapter and component is separate work; until then the `echo` component's replace mode exercises the path.

## SDK impact and examples

Each SDK adds one streaming method beside its existing `Converse` and ships a fold helper so the streamed and unary results have the same type. Existing unary implementations remain unchanged. Fold helpers assemble tool-call fragments rather than treating the final fragment as a complete call, and return a completed response only after successful termination. The gRPC stream is the primary path for SDKs; SSE is for browsers, scripts and non-SDK clients.

| SDK | New API | Shape |
| --- | --- | --- |
| Go | `client.ConverseStreamAlpha3(ctx, req) (iter.Seq2[*ConversationStreamResponseAlpha3, error], error)` | Go 1.23 range-over-func iterator; `conversation.Fold(seq)` returns the response |
| .NET | `IAsyncEnumerable<ConversationStreamEvent> ConverseStreamAsync(request, cancellationToken)` on `DaprAIClient` (Dapr.AI package) | typed event record hierarchy; `events.FoldAsync()` returns `ConversationResponse` |
| Python | `async for event in client.converse_stream_alpha3(...)` plus a sync generator | dataclass events; `fold(events)` |
| JavaScript | `for await (const ev of client.conversation.converseStream(...))` | `ReadableStream` of events in browsers via SSE, gRPC in Node |
| Java | `Flux<ConversationStreamEvent> converseStream(request)` | Reactor, matching the existing async client style |

**.NET sketch** (the `Dapr.AI` package, in the style used across the modern SDK and `Dapr.Actors.Next`: options pattern, nullable annotations, no reflection on the hot path):

```csharp
var request = new ConversationRequest("anthropic")
{
    Inputs = [ new ConversationInput([ new UserMessage("Summarise Dapr in one line") ]) ],
    StreamOptions = new() { ContentMode = ContentMode.AllowReplace } // or PreferReplace to request denoising snapshots
};

await foreach (var ev in client.ConverseStreamAsync(request, ct))
{
    switch (ev)
    {
        case ContentEvent { UpdateMode: UpdateMode.Append } c: console.Write(c.Text); break;
        case ContentEvent { UpdateMode: UpdateMode.Replace } c: console.Rewrite(c.Text); break; // diffusion
        case DoneEvent d: contextId = d.ContextId; break;
    }
}
```

**JavaScript sketch** (proposed SDK API, not available today; the illustrative objects use camelCase fields and SDK enums, which the SDK maps to and from the wire enums on both transports). `renderChoice(outputIndex, choiceIndex, text)` replaces the app's preview for that choice, and `clearPreviews()` discards those previews if the stream fails. The SDK propagates cancellation via `signal`, throws on SSE errors, transport failures or EOF without `done`, and only completes iteration after successful transport termination.

```javascript
import { ContentMode, UpdateMode } from "@dapr/dapr";

const request = {
  name: "anthropic",
  inputs: [{
    messages: [{
      ofUser: { content: [{ text: "Summarise Dapr in one line" }] }
    }]
  }],
  streamOptions: {
    contentMode: ContentMode.AllowReplace
    // Use ContentMode.PreferReplace to request snapshots.
  }
};

const textByChoice = new Map();
let contextId;
let completedContextId;

try {
  for await (const ev of client.conversation.converseStream(request, { signal })) {
    if (ev.content) {
      const { outputIndex = 0, choiceIndex = 0, text = "", updateMode } = ev.content;
      const key = `${outputIndex}:${choiceIndex}`;
      let nextText;
      switch (updateMode) {
        case UpdateMode.Append:
          nextText = (textByChoice.get(key) ?? "") + text;
          break;
        case UpdateMode.Replace:
          nextText = text; // An empty snapshot clears the previous content.
          break;
        default:
          throw new Error(`Unsupported conversation update mode: ${updateMode}`);
      }
      textByChoice.set(key, nextText);
      renderChoice(outputIndex, choiceIndex, nextText);
    } else if (ev.done) {
      completedContextId = ev.done.contextId;
    }
  }
  contextId = completedContextId;
} catch (error) {
  clearPreviews();
  throw error;
}
```

**curl** (no SDK):

```bash
curl -N -H 'Accept: text/event-stream' \
  http://localhost:3500/v1.0-alpha3/conversation/openai/converse \
  -d '{"inputs":[{"messages":[{"ofUser":{"content":[{"text":"What is Dapr?"}]}}]}]}'
```

**Agent loop.** The sketches above render content only; they do not execute tools. For a tool-calling request, fold all events from the same stream into the existing response type, wait for successful completion, then validate and execute the assembled tool calls through the app's existing path. Append an `ofAssistant` message with the folded `toolCalls` and one `ofTool` message per result to the next request, and stream again. No second model invocation or new early-execution contract is required.

## Backward compatibility and migration

Alpha3 is additive: nothing in Alpha1 or Alpha2 changes, and the Alpha2 messages are reused by reference rather than copied, so a bug fix in `ConversationResultAlpha2CompletionUsage` reaches both.

- `ConverseAlpha1` and `ConverseAlpha2` retain their existing serving and deprecation status. This proposal does not schedule Alpha2 deprecation.
- Request migration is a rename: `ConversationRequestAlpha2` → `ConversationRequestAlpha3` with identical field numbers and JSON names. The HTTP path moves from `/v1.0-alpha2` to `/v1.0-alpha3`.
- Unary HTTP status (`200 OK`), tool-calling, scrubbing, resiliency configuration, and context semantics remain unchanged. The new streaming transport uses HTTP `200` with SSE; buffering and post-emission retry restrictions apply only to streaming delivery.
- Events use a `oneof`, so a future kind (reasoning, citations, refusal) is a new field number that old clients skip. Clients are told to ignore unknown events; SDK fold helpers must not fail on them. Update modes are the exception: an unrecognised one fails the stream, so a new mode is only ever emitted to clients that opt in through `content_mode`.
- The `Streamer` interface is optional for components; the runtime fallback means a component can be converted in any release without coordinating with the runtime.
- Stable API naming and any field flattening are outside this proposal.

## Testing, observability and rollout

**Conformance (`tests/conformance/conversation`).** For each component, fixed provider-response fixtures establish that (1) the folded response matches expected existing response fields, except for intentionally omitted usage; (2) the event grammar holds across interleaved outputs and choices, with model and usage associated with the correct output; (3) `sequence` is contiguous; (4) `content_mode = APPEND` never emits `REPLACE`, including when snapshots revise earlier text; (5) tool-call fragments assemble correctly and receive no updates after completion; (6) client cancellation stops provider work, including blocked channel sends; and (7) the last content before each `finish` is the final text and fragments split only on UTF-8 code-point boundaries. Live-provider checks verify supported delivery behaviour, not equality between independent generations.

**Integration (`tests/integration/suite/daprd/conversation`).** Use the `echo` component with `mode: append` and `mode: replace`, a fault-injecting component that errors after `n` chunks, and a slow-reader SSE client to exercise snapshot coalescing and the buffer limit. Add a synthetic LLaDA-style denoising-state fixture at the component-event boundary (full-text snapshots with masked positions followed by final decoded text, not an asserted upstream SSE format) and an OpenAI `/responses` SSE fixture so both paths are tested without a live key or model deployment. Verify immediate snapshots with `ALLOW_REPLACE` and `PREFER_REPLACE` (and that dual-mode components only switch to snapshots on `PREFER_REPLACE`), final-only buffered text for snapshot-only providers with `APPEND`, final-only response caching when `responseCacheTTL` is set, and full-response buffering through the existing scrubber with `scrub_pii`. Cover errors and EOF before `done`, missing usage, tool-argument truncation, and failure of one of several outputs. Verify the unchanged total timeout, retries before the first response event but not after it, and the unchanged unary HTTP status (`200 OK`) and context behaviour. Verify that the SSE stream opens before the provider responds, that later provider failures arrive as `error` events, that `stream` takes precedence over `Accept`, and that `delivery` is reported correctly for native, fallback and buffered-snapshot outputs.

**Unit.** Append/replace folding and snapshot buffering, including `"cat"` followed by `"car"` and empty-text `REPLACE` resetting prior text to `""` (including omitted `text` in protobuf JSON); per-output metadata and usage; interleaved tool-call assembly; rejection of unspecified or unknown update modes; and provider `stop_reason` mappings. Scrubbing fixtures split entities across chunks and confirm the buffered path delivers exactly the existing scrubber's processed result. SSE fixtures use the selected payload's protobuf JSON and round-trip to the same typed events as gRPC.

**Operational signals.** `dapr_conversation_stream_ttft_ms`, filtered to native delivery, is the metric operators should alert on; the `error` SSE event and the gRPC trailing status both carry the Dapr error code so dashboards do not need to parse bodies.

**Rollout.**

1. Proto and runtime behind the `ConversationStreaming` preview feature (one release, default on), with the `echo` and OpenAI-compatible components streaming natively and every other component on the fallback.
2. Anthropic, Google AI, Bedrock, Ollama converted. Go and .NET SDKs ship `ConverseStream`.
3. Python, JavaScript, Java SDKs; docs reference page for `v1.0-alpha3`, including diffusion rendering and the documented buffering limitations. Alpha2's deprecation status remains unchanged.

## Open questions

None outstanding. Whether an app can tell native from buffered delivery is settled by `delivery` on `output_start`, and a component for a diffusion provider is separate components-contrib work.

The proposal fixes `APPEND` as the default and permits text replacements only with `ALLOW_REPLACE` or `PREFER_REPLACE`. Tool-call arguments are append-only, and no fragments follow completion. Changes to PII processing, resiliency policies, reasoning, bidirectional sessions, and deprecation decisions are separate work, not open design dependencies for streaming.

## References

- [Conversation API reference (Alpha2)](https://docs.dapr.io/reference/api/conversation_api/), Dapr docs v1.19
- [`dapr/proto/runtime/v1/ai.proto`](https://github.com/dapr/dapr/blob/master/dapr/proto/runtime/v1/ai.proto), dapr/dapr master, including the `StreamOptions` TODO
- [dapr/dapr](https://github.com/dapr/dapr/) runtime repository
- [Dapr.Actors.Next overview](https://docs.dapr.io/developing-applications/sdks/dotnet/dotnet-actors-next/), the 1.18 bidirectional gRPC streaming pattern this proposal follows
- [LLaDA official repository](https://github.com/ML-GSAI/LLaDA) and [LLaDA-8B-Instruct model weights](https://huggingface.co/GSAI-ML/LLaDA-8B-Instruct), open-weight masked-diffusion text model with multi-turn conversation support
- [LLaDA inference](https://github.com/ML-GSAI/LLaDA/blob/main/generate.py) and [denoising visualization](https://github.com/ML-GSAI/LLaDA/blob/main/app.py), evidence for intermediate states and block generation, not an existing live snapshot-streaming API
- [OpenAI chat completion streaming](https://platform.openai.com/docs/api-reference/chat/create) and [Anthropic Messages streaming](https://docs.anthropic.com/en/api/messages-streaming), the two provider stream shapes mapped above
