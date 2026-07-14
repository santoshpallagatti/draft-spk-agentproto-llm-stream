---
###
# Internet-Draft for LLM Inference Streaming Wire Format
#
# Draft name: draft-spk-agentproto-llm-stream
# Target: Agent Communication Protocol (agentproto) Working Group
#
# Authors: Yaroslav Rosomakho, Santosh Pallagatti
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
#
###
title: "A Standard Wire Format for Large Language Model Inference Streaming"
abbrev: "LLM-Stream"
category: std
docname: draft-spk-agentproto-llm-stream-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: Applications and Real-Time
workgroup: Agent Communication Protocol
keyword:
 - LLM
 - inference
 - streaming
 - SSE
 - AI
venue:
  group: Agent Communication Protocol
  type: Working Group
  github: AvinashSontakke/draft-spk-agentproto-llm-stream
  latest: https://AvinashSontakke.github.io/draft-spk-agentproto-llm-stream/draft-spk-agentproto-llm-stream.html
author:
 -
    fullname: Yaroslav Rosomakho
    organization: ""
    email: ""
 -
    fullname: Santosh Pallagatti
    organization: ""
    email: ""

normative:
  WHATWG-HTML:
    title: "HTML Living Standard - Server-sent events"
    target: https://html.spec.whatwg.org/multipage/server-sent-events.html
    author:
      org: WHATWG

informative:
  gRPC:
    title: "gRPC over HTTP/2"
    target: https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md
    author:
      org: Google
  Connect:
    title: "Connect Protocol Specification"
    target: https://connectrpc.com/docs/protocol
    author:
      org: Buf Technologies
  MCP:
    title: "Model Context Protocol Specification"
    target: https://spec.modelcontextprotocol.io
    author:
      org: Anthropic / AAIF
  SignalR:
    title: "SignalR Hub Protocol"
    target: https://github.com/dotnet/aspnetcore/blob/main/src/SignalR/docs/specs/HubProtocol.md
    author:
      org: Microsoft
  W3C-WEBTRANSPORT:
    title: "WebTransport"
    target: https://www.w3.org/TR/webtransport/
    author:
      org: W3C

...

--- abstract

Large Language Model (LLM) inference endpoints stream response tokens to clients using a fragmented set of vendor-specific application-layer protocols layered on top of standardized transports (HTTP/2, HTTP/1.1, WebSocket, Server-Sent Events). While these transports carry IETF or W3C standardization, the JSON payload schemas, event taxonomies, and framing conventions used within them are entirely vendor-defined, with no RFCs or common specifications governing them.

This fragmentation imposes costs across the AI ecosystem. Client developers must build vendor-specific parsing into every SDK. LLM vendors face unnecessary integration friction that raises the barrier to customer adoption. Infrastructure providers (CDNs, load balancers, API gateways) must build vendor-specific handling for caching, routing, and observability. Enterprise customers seeking multi-vendor flexibility are forced to maintain custom translation layers. And observability, compliance, and orchestration tooling must be rebuilt for each provider independently.

This document defines a standard wire format for LLM inference streaming over Server-Sent Events (SSE) on HTTP. It specifies a request envelope media type (application/llm-request+json), a response event taxonomy, and a JSON event envelope schema that together enable any participant in the AI ecosystem, including client libraries, LLM vendors, infrastructure providers, orchestration platforms, and observability tools, to handle AI inference traffic using a single, shared protocol contract.

The scope of this document is strictly Client/Application to LLM inference endpoint streaming. Agent-to-tool interaction (e.g., MCP) and agent-to-agent communication are out of scope.

--- middle

# Introduction

## Problem Statement

The AI inference ecosystem is experiencing protocol fragmentation reminiscent of the early web. Every major vendor has independently defined its own wire format for the same fundamental operation: streaming generated tokens from a model to a client. Each vendor uses a different combination of application-layer framing and JSON payload schema:

- OpenAI streams SSE with JSON payloads using the path choices[0].delta.content for generated text.
- Anthropic streams SSE with typed event blocks, placing generated text at delta.text inside content_block_delta events.
- Google Gemini streams SSE with JSON payloads using candidates[0].content.parts[0].text.
- Microsoft Azure / Copilot uses WebSocket with the SignalR Hub Protocol's {{SignalR}} proprietary 0x1E record-separator framing.
- Cursor and Windsurf (AI coding assistants) use the Connect protocol {{Connect}} (application/connect+proto) with a 5-byte binary envelope and Protobuf-encoded payloads over HTTP/2.

The underlying transports, specifically HTTP/2 {{RFC9113}}, HTTP/1.1 {{RFC9110}}, WebSocket {{RFC6455}}, and SSE {{WHATWG-HTML}}, are all properly standardized. The problem exists entirely at the application layer: the JSON schemas, event types, stream lifecycle conventions, and payload structures that ride on top of these transports.

Additionally, even where vendors use the same sub-protocol, behavior diverges in practice. For example, Cursor keeps HTTP/2 streams open indefinitely without sending END_STREAM, while Windsurf sends END_STREAM on the same Connect+proto protocol: same framing, different stream lifecycle.

SSE is used correctly per the W3C specification by most vendors, but nothing in the SSE specification constrains what goes inside the event: and data: fields. Event type names, JSON field names, nesting structures, and payload schemas are entirely up to each provider.

## Use Cases

The absence of a standard wire format for AI inference streaming creates unnecessary cost and complexity across the AI ecosystem. The following use cases illustrate the breadth of the problem:

### Client SDK Portability

Today every LLM provider requires its own client library (openai-python, anthropic-sdk, google-genai), each with proprietary streaming response parsing. A standard format enables a single open-source client library that works with any conforming provider. Developers integrate once and switch providers by changing a URL.

### LLM Vendor Customer Acquisition

Vendor-specific wire formats force every client application to depend on a provider-specific SDK with proprietary streaming response parsing. Switching providers requires swapping SDK dependencies and updating response handling code throughout the application, creating soft lock-in that raises the cost of evaluating alternatives. Abstraction platforms such as LiteLLM, Portkey, and AWS Bedrock exist specifically to bridge this gap, each maintaining vendor-specific translation internally. A standard format would make provider-switching as simple as changing an endpoint URL, with no SDK or code changes required. Vendors compete on model quality, not integration friction.

### Multi-Provider Flexibility

Enterprises increasingly route different workloads to different providers: one vendor for coding tasks, another for summarization, a third for cost-sensitive workloads. Today this requires maintaining separate client integrations and translation layers (LiteLLM, Portkey, AWS Bedrock, Azure AI Gateway). A standard format makes multi-provider usage as simple as changing the endpoint URL.

### Observability and Debugging

Standard event types (stream.start, content.delta, stream.end) and sequence numbers enable consistent monitoring, latency measurement, error tracking, and SLA reporting across providers, without building custom instrumentation per vendor. Both client-side and server-side teams benefit from a shared vocabulary for AI inference lifecycle events.

### Browser-Native AI Consumption

As AI consumption moves into browsers, a standard SSE schema means the EventSource API has a predictable contract to build on. Today, every browser-based AI client reinvents streaming response parsing in JavaScript. A standard event taxonomy and JSON envelope would enable native browser support for AI inference streaming.

### Regulatory and Compliance Auditing

AI regulation (e.g., the EU AI Act) increasingly requires enterprises to audit AI usage: what was prompted, what was generated, which model was used, how many tokens were consumed. A standard request and response envelope makes it possible to build compliance tooling that works across all providers without vendor-specific extraction logic.

### Multi-Vendor Orchestration Platforms

Services such as LiteLLM, Portkey, AWS Bedrock, and Azure AI Gateway exist specifically to provide a unified interface across LLM providers. Today each maintains vendor-specific translation layers for every supported provider. A standard format eliminates this translation entirely: one input format, one output format, any provider.

### HTTP Intermediaries

Forward proxies, reverse proxies, and security appliances that process AI inference traffic need to extract generated text and tool invocations from streaming responses for content inspection, policy enforcement, and audit logging. Today, the generated text lives at a different JSON path per vendor, requiring a separate parser for each provider. A standard format reduces this to a single parser with one canonical field to inspect.

## Scope

This document addresses Client/Application to LLM inference endpoint streaming, specifically the path from an end-user application (web browser, IDE, mobile app, API client) to an LLM inference endpoint and back.

The following are explicitly out of scope:

- Agent-to-tool interaction (e.g., Model Context Protocol {{MCP}})
- Agent-to-agent communication
- Model training traffic
- Embedding and batch inference APIs (non-streaming)

## Design Rationale

The standard is designed around three principles:

1. Keep SSE as the transport. SSE over HTTP is already the dominant transport for AI inference streaming, used by OpenAI, Anthropic, Google, Cohere, and Mistral. It works on both HTTP/2 and HTTP/1.1, has broad ecosystem support, and requires no new transport protocol. A WebTransport {{W3C-WEBTRANSPORT}} binding MAY be defined in a future document as adoption matures.

2. Standardize only the payload. The transport layer is already standardized. This document defines only what goes inside the SSE event: and data: fields, specifically the event type taxonomy and JSON envelope schema.

3. Preserve vendor extensibility. Vendors retain full freedom to include proprietary metadata via namespaced extension fields. Conforming parsers MUST ignore unknown fields.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Inference endpoint:
: An HTTP endpoint that accepts a prompt and returns a generated completion, typically streamed token-by-token.

Implementation:
: Any software component that produces or consumes LLM inference streams conforming to this specification, including client libraries, server endpoints, API gateways, and orchestration platforms.

Stream:
: A single request-response exchange in which the response is delivered incrementally via SSE events.

Event envelope:
: The JSON object carried in each SSE data: field, conforming to the schema defined in {{event-envelope}}.

Content delta:
: An incremental fragment of generated text delivered in a single event.

# Transport Binding

A conforming inference endpoint:

- MUST accept requests via HTTP POST.
- MUST deliver streaming responses using Server-Sent Events (SSE) as defined in the WHATWG HTML Living Standard {{WHATWG-HTML}}, Section "Server-sent events."
- MUST use HTTP/2 {{RFC9113}} or HTTP/1.1 {{RFC9110}} as the underlying HTTP version.
- SHOULD prefer HTTP/2 for its multiplexing and flow-control benefits.
- MUST use TLS 1.3 {{RFC8446}} or later for production deployments.

SSE over HTTP was chosen because it is already the dominant transport for AI inference streaming. It works on both HTTP/2 (multiplexed) and HTTP/1.1 (one stream per connection), providing graceful degradation in environments that support only HTTP/1.1. In contrast, gRPC {{gRPC}} and Connect+proto mandate HTTP/2 and break in HTTP/1-only environments.

AI inference streams are characteristically long-lived and open-ended; the server does not set Content-Length and typically does not send HTTP/2 END_STREAM until generation is complete. This behavior differs from traditional HTTP request-response patterns and may affect timeout and buffering configurations in both clients and infrastructure. The same pattern is expected on HTTP/3 (QUIC).

# Media Types and Request Format

## Request Media Type

This document registers the media type application/llm-request+json.

Sent by the client in the HTTP POST body, this Content-Type indicates that the request payload conforms to the request envelope defined in {{request-envelope}}. Any component that observes this Content-Type can immediately identify the traffic as LLM inference, enabling automated routing, logging, and processing without hostname-based heuristics.

## Request Envelope {#request-envelope}

The request body is a JSON object. Its top-level fields are organized into five logical groups: protocol control, conversation history, tool declarations, generation parameters, and extensibility.

| Field | Type | Required | Description |
|---|---|---|---|
| version | string | REQUIRED | Protocol version. MUST be "1.0". |
| stream | boolean | REQUIRED | MUST be true for streaming. |
| model | string | REQUIRED | Model identifier. Opaque string. |
| messages | array | REQUIRED | Conversation history. See {{messages}}. |
| tools | array | OPTIONAL | Tool/function definitions. See {{tools}}. |
| parameters | object | OPTIONAL | Generation parameters. See {{gen-parameters}}. |
| metadata | object | OPTIONAL | Request metadata. See {{metadata-field}}. |
| extensions | object | OPTIONAL | Vendor extensions. See {{extensions-field}}. |

## Messages (Conversation History) {#messages}

The messages field is an ordered array of message objects representing the conversation history. The model reads this full history to understand context before generating a response.

| Field | Type | Required | Description |
|---|---|---|---|
| role | string | REQUIRED | One of: "system", "user", "assistant", "tool". |
| content | string or array | REQUIRED | The message content. |
| name | string | OPTIONAL | Display name for the participant. |
| tool_call_id | string | CONDITIONAL | REQUIRED when role is "tool". |

The role field identifies who produced the message:

"system":
: Instructions that configure the model's behavior for the conversation. Typically the first message. Not visible to end users.

"user":
: Input from the end user: the prompt, question, or instruction the model should respond to.

"assistant":
: The model's own prior responses. Included in multi-turn conversations so the model has context of what it previously generated.

"tool":
: A result returned from a tool invocation. When the model requests a tool call (via a tool.call event in the response), the client executes the tool locally and sends the result back as a "tool" message in the next request. The tool_call_id field MUST reference the id from the corresponding tool.call event.

A typical multi-turn conversation:

~~~json
"messages": [
  {"role": "system",
   "content": "You are a coding assistant."},
  {"role": "user",
   "content": "Write a Python function to sort a list."},
  {"role": "assistant",
   "content": "def sort_list(items):\n    return sorted(items)"},
  {"role": "user",
   "content": "Now make it sort in reverse order."}
]
~~~

### The Content Field

The content field takes one of two forms depending on whether the message contains only text or includes non-text attachments.

Text-only messages use a simple string:

~~~json
{"role": "user", "content": "What is the capital of France?"}
~~~

Multimodal messages use an array of typed content parts:

~~~json
{"role": "user", "content": [
  {"type": "text", "text": "What is shown in this image?"},
  {"type": "image", "media_type": "image/png",
   "data": "<base64-encoded image>"}
]}
~~~

| Type | Required Fields | Description |
|---|---|---|
| "text" | text | Plain text content. |
| "image" | media_type, data | Base64-encoded image. |
| "document" | media_type, data | Base64-encoded document. |
| "audio" | media_type, data | Base64-encoded audio. |

When content is a string, it is semantically equivalent to an array containing a single text part. Implementations MUST support both forms.

## Tools (Function Declarations) {#tools}

The tools field is an optional array of tool definitions. These are NOT part of the conversation; they are declarations of capabilities that the client can execute locally if the model decides to invoke them.

The model reads these definitions and, when it determines that a tool would help answer the user's request, emits a tool.call event in the response stream instead of generating text. The model itself does not execute anything; it outputs a structured request asking the client to run the specified function with the given arguments.

| Field | Type | Required | Description |
|---|---|---|---|
| name | string | REQUIRED | Function name. |
| description | string | REQUIRED | Human-readable description. |
| parameters | object | REQUIRED | JSON Schema for input parameters. |

~~~json
"tools": [
  {"name": "get_weather",
   "description": "Get current weather for a city.",
   "parameters": {
     "type": "object",
     "properties": {
       "city": {"type": "string",
                "description": "City name, e.g. Vienna"},
       "units": {"type": "string",
                 "enum": ["celsius", "fahrenheit"]}
     },
     "required": ["city"]
   }}
]
~~~

The complete tool interaction flow is described in {{tool-flow}}.

## Generation Parameters {#gen-parameters}

The parameters field contains knobs that control how the model generates its response.

| Field | Type | Description |
|---|---|---|
| max_tokens | integer | Maximum number of tokens to generate. |
| temperature | number | Randomness. 0.0 = deterministic, 1.0 = creative. |
| top_p | number | Nucleus sampling threshold. |
| stop_sequences | array | Strings that immediately end generation. |

All fields within parameters are OPTIONAL. Vendors MAY define additional parameters. Unknown parameters MUST be ignored.

## Request Metadata {#metadata-field}

The metadata field carries request-level information for tracing, billing, and audit. This data is NOT sent to the model and does NOT influence generation.

~~~json
"metadata": {
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "user_id": "user-12345",
  "session_id": "sess-67890"
}
~~~

The request_id field, if present, SHOULD be echoed in the stream.start and stream.end response events.

## Vendor Extensions {#extensions-field}

The extensions field is an escape hatch for vendor-specific features. Keys MUST be namespaced using the format "vendor:name" to prevent collisions.

~~~json
"extensions": {
  "vendor:openai": {"logprobs": true},
  "vendor:anthropic": {
    "thinking": {"type": "enabled",
                 "budget_tokens": 10000}}
}
~~~

Conforming implementations MUST ignore extension namespaces they do not recognize.

## Tool Interaction Model {#tool-flow}

The model is stateless and cannot execute tools itself; it can only emit a structured request for the client to execute a tool on its behalf.

1. The client sends a request with messages and tools.
2. The model emits a tool.call event with the tool name and arguments, followed by content.stop with stop_reason "tool_use".
3. The client executes the named function locally.
4. The client sends a new request with the original history, the assistant's tool call, and a "tool" role message with the result.
5. The model reads the tool result and generates its final text response.

~~~
Request 1:
{"messages": [{"role": "user",
  "content": "What is the weather in Vienna?"}],
 "tools": [{"name": "get_weather", ...}]}

Response 1:
event: tool.call
data: {"type":"tool.call","id":"call_1",
  "name":"get_weather",
  "arguments":"{\"city\":\"Vienna\"}"}
event: content.stop
data: {"type":"content.stop",
  "stop_reason":"tool_use"}

Client executes get_weather("Vienna"), gets result.

Request 2:
{"messages": [
  {"role": "user",
   "content": "What is the weather in Vienna?"},
  {"role": "assistant", "content": null,
   "tool_calls": [{"id": "call_1",
     "name": "get_weather",
     "arguments": "{\"city\":\"Vienna\"}"}]},
  {"role": "tool", "tool_call_id": "call_1",
   "content": "Vienna: 24C, sunny"}],
 "tools": [{"name": "get_weather", ...}]}

Response 2:
event: content.delta
data: {"type":"content.delta",
  "delta":{"text":"The weather in Vienna is 24C."}}
~~~

The tool definitions travel with every request because the model is stateless.

## Response Media Type

The SSE response uses the standard text/event-stream media type. Conformance is signaled by the combination of application/llm-request+json on the request and a stream.start event as the first event in the response.

# Response Streaming Format

The response is delivered as a Server-Sent Events (SSE) stream over HTTP.

## SSE Event Framing

Each event on the wire consists of field lines followed by a blank line:

~~~
event: <event-type>\n
data: <JSON event envelope>\n
\n
~~~

The blank line (\n\n) is the event boundary. The HTTP transport may split bytes arbitrarily across frames or chunks; the SSE layer reassembles them.

~~~
HTTP layer:  Delivers raw bytes (DATA frames / chunks)
                  |
SSE layer:   Reassembles lines, uses \n\n as boundary
                  |
This spec:   Parses JSON envelope from data: field
~~~

The following constraints MUST be observed:

- Each SSE event MUST contain exactly one complete, valid JSON object in its data: field.
- A JSON event envelope MUST NOT span multiple SSE events.
- An endpoint MUST NOT emit partial JSON that requires a subsequent SSE event to complete.

## Event Envelope {#event-envelope}

Every event carries a JSON object with three common fields:

| Field | Type | Required | Description |
|---|---|---|---|
| v | string | REQUIRED | Protocol version. MUST be "1.0". |
| seq | integer | REQUIRED | Zero-based, monotonically increasing. |
| type | string | REQUIRED | Event type. MUST match SSE event: field. |

The envelope follows a TLV-style structure:

T (Type):
: The SSE event: field. Identifies the event type without parsing JSON.

L (Length):
: Implicit. The SSE \n\n boundary defines the extent of the JSON body. See {{framing-tradeoff}}.

V (Value):
: The JSON event envelope in the data: field.

Conforming parsers MUST ignore unknown fields (forward compatibility).

### Sequence Numbers

The seq field is zero-based, incrementing by one per event. stream.start MUST have seq 0.

1. Gap detection: seq values 0, 1, 2, 5 indicates events 3 and 4 were lost.
2. Ordering: provides canonical ordering when events are buffered.

## Event Types

| Event Type | Direction | Purpose |
|---|---|---|
| stream.start | First | Stream initiation, request metadata |
| content.delta | Repeated | Incremental text fragment |
| content.stop | Once/block | End of content block with stop reason |
| tool.call | Zero or more | Model requests tool invocation |
| usage | One or more | Token consumption |
| error | Zero or one | Generation error |
| stream.end | Last | Clean stream termination |

### stream.start {#stream-start}

MUST be the first event. Signals the endpoint has accepted the request.

| Field | Type | Required | Description |
|---|---|---|---|
| v | string | REQUIRED | "1.0" |
| seq | integer | REQUIRED | MUST be 0. |
| type | string | REQUIRED | "stream.start" |
| request_id | string | OPTIONAL | Echoed from request metadata. |
| model | string | REQUIRED | Model generating the response. |
| created | integer | REQUIRED | Unix timestamp (seconds). |

~~~
event: stream.start
data: {"v":"1.0","seq":0,"type":"stream.start",
  "request_id":"550e8400-...",
  "model":"example-model","created":1749292800}
~~~

### content.delta {#content-delta}

Carries an incremental text fragment. Most frequent event in a stream.

| Field | Type | Required | Description |
|---|---|---|---|
| v | string | REQUIRED | "1.0" |
| seq | integer | REQUIRED | Sequence number. |
| type | string | REQUIRED | "content.delta" |
| index | integer | REQUIRED | Content block index. 0 for primary. |
| role | string | REQUIRED | "assistant". |
| delta | object | REQUIRED | Contains incremental content. |

delta.text (string, REQUIRED):
: The generated text fragment. This is the canonical field for generated text: the single location where generated content always appears, regardless of vendor. Standardizing this path is the primary goal of this specification.

~~~
event: content.delta
data: {"v":"1.0","seq":1,"type":"content.delta",
  "index":0,"role":"assistant",
  "delta":{"text":"The weather in Vienna"}}

event: content.delta
data: {"v":"1.0","seq":2,"type":"content.delta",
  "index":0,"role":"assistant",
  "delta":{"text":" is currently 24C"}}

event: content.delta
data: {"v":"1.0","seq":3,"type":"content.delta",
  "index":0,"role":"assistant",
  "delta":{"text":" and sunny."}}
~~~

Concatenating delta.text from seq 1 through 3 produces: "The weather in Vienna is currently 24C and sunny."

### content.stop {#content-stop}

Signals the end of a content block.

| Field | Type | Required | Description |
|---|---|---|---|
| v | string | REQUIRED | "1.0" |
| seq | integer | REQUIRED | Sequence number. |
| type | string | REQUIRED | "content.stop" |
| index | integer | REQUIRED | Block index being terminated. |
| stop_reason | string | REQUIRED | Why generation stopped. |

stop_reason values:

"end_turn":
: The model finished its response naturally.

"max_tokens":
: Generation reached the max_tokens limit.

"tool_use":
: The model is requesting a tool invocation.

"content_filter":
: The endpoint's safety system terminated generation.

### tool.call {#tool-call}

Emitted when the model determines a tool invocation would help. The model does not execute the tool; it outputs a structured request for the client. See {{tool-flow}}.

| Field | Type | Required | Description |
|---|---|---|---|
| v | string | REQUIRED | "1.0" |
| seq | integer | REQUIRED | Sequence number. |
| type | string | REQUIRED | "tool.call" |
| index | integer | REQUIRED | Content block index. |
| id | string | REQUIRED | Unique invocation identifier. |
| name | string | REQUIRED | Function name from request tools. |
| arguments | string | REQUIRED | JSON-encoded function arguments. |

### usage {#usage}

Reports token consumption for billing, cost tracking, and quota management.

| Field | Type | Required | Description |
|---|---|---|---|
| v | string | REQUIRED | "1.0" |
| seq | integer | REQUIRED | Sequence number. |
| type | string | REQUIRED | "usage" |
| input_tokens | integer | REQUIRED | Tokens in the request. |
| output_tokens | integer | REQUIRED | Tokens generated. |

MAY appear mid-stream or near stream end. At least one SHOULD be emitted per stream.

### error {#error}

Signals a generation error. After an error, the endpoint SHOULD send stream.end.

| Field | Type | Required | Description |
|---|---|---|---|
| v | string | REQUIRED | "1.0" |
| seq | integer | REQUIRED | Sequence number. |
| type | string | REQUIRED | "error" |
| code | string | REQUIRED | Machine-readable error code. |
| message | string | REQUIRED | Human-readable description. |

Error codes: "rate_limited", "context_length_exceeded", "content_filtered", "internal_error", "overloaded".

### stream.end {#stream-end}

MUST be the last event. After stream.end, no further events on this stream.

| Field | Type | Required | Description |
|---|---|---|---|
| v | string | REQUIRED | "1.0" |
| seq | integer | REQUIRED | Final sequence number. |
| type | string | REQUIRED | "stream.end" |
| request_id | string | OPTIONAL | Echoed for correlation. |

## Stream Lifecycle

1. The stream begins with exactly one stream.start (seq 0).
2. Zero or more content.delta events deliver text incrementally.
3. Zero or more tool.call events request tool invocations.
4. Zero or more usage events report token consumption.
5. Each content block is terminated by content.stop.
6. If an error occurs, an error event is emitted.
7. The stream ends with exactly one stream.end.

~~~
       POST request
            |
      stream.start (seq=0)
            |
       STREAMING
      /    |    \
content.delta tool.call usage
      \    |    /
    content.stop | error
            |
       stream.end
            |
         CLOSED
~~~

## Complete Example

~~~
HTTP/2 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

event: stream.start
data: {"v":"1.0","seq":0,
  "type":"stream.start",
  "request_id":"550e8400-...",
  "model":"example-model",
  "created":1749292800}

event: content.delta
data: {"v":"1.0","seq":1,
  "type":"content.delta",
  "index":0,"role":"assistant",
  "delta":{"text":"The"}}

event: content.delta
data: {"v":"1.0","seq":2,
  "type":"content.delta",
  "index":0,"role":"assistant",
  "delta":{"text":" weather"}}

event: content.delta
data: {"v":"1.0","seq":3,
  "type":"content.delta",
  "index":0,"role":"assistant",
  "delta":{"text":" in Vienna"}}

event: content.delta
data: {"v":"1.0","seq":4,
  "type":"content.delta",
  "index":0,"role":"assistant",
  "delta":{"text":" is 24C and sunny."}}

event: content.stop
data: {"v":"1.0","seq":5,
  "type":"content.stop",
  "index":0,"stop_reason":"end_turn"}

event: usage
data: {"v":"1.0","seq":6,
  "type":"usage",
  "input_tokens":42,"output_tokens":8}

event: stream.end
data: {"v":"1.0","seq":7,
  "type":"stream.end",
  "request_id":"550e8400-..."}
~~~

# Framing Design Trade-off {#framing-tradeoff}

## SSE Text-Based Framing (This Specification)

SSE delimits events with \n\n. The parser must scan the byte stream to locate each boundary. This is O(n) per event.

## Length-Prefixed Binary Framing (Alternative)

A binary approach (e.g., gRPC's {{gRPC}} 5-byte envelope) prefixes each message with a type byte and 4-byte length. The parser reads the header and jumps directly to the next message. This is O(1) per boundary.

## Rationale for SSE

1. Compatibility: SSE is supported natively by browser EventSource and every major HTTP library. Binary framing would require custom parsers.
2. Existing deployment: The five major SSE-based LLM providers would need minimal changes to adopt a standard JSON schema. Binary framing would require re-engineering their streaming pipeline.
3. Debuggability: SSE events are human-readable in browser developer tools, curl, and log files.
4. Incremental adoption: The payload schema is transport-independent. The same event types and JSON envelope can be carried over SSE or a future binary binding without changes.

A future companion document MAY define a binary transport binding, for example over WebTransport {{W3C-WEBTRANSPORT}}, that provides length-prefixed framing while reusing the same event types and JSON envelope.

# Backward Compatibility

## API Gateway Translation

An API gateway can accept vendor-specific requests, forward them to the upstream provider in their native format, and translate the response stream into the standard format before delivering it to the client.

## Content-Type Negotiation

If a client sends Content-Type: application/llm-request+json and the endpoint does not support the standard, it SHOULD return 415 Unsupported Media Type, allowing the client to fall back.

## Dual-Format Endpoints

A vendor MAY support both formats, selected by the request Content-Type or an Accept header:

~~~
Accept: text/event-stream; profile="llm-stream-1.0"
~~~

# Security Considerations

## Prompt Confidentiality

Prompts in application/llm-request+json requests may contain highly sensitive data. Any system that logs or processes these requests MUST apply appropriate confidentiality protections.

## Tool-Call Abuse

A compromised or adversarial endpoint could emit tool.call events invoking sensitive tools. Client implementations SHOULD validate tool.call events against the set of tools declared in the original request and SHOULD require user confirmation before executing tool invocations with side effects.

## Extension Field Safety

The extensions field allows vendor-specific data. Implementations SHOULD NOT interpret or execute extension values from untrusted sources.

## Sequence Exhaustion

The seq field is an integer. Implementations SHOULD use 64-bit integers. A stream exceeding 2^53 events (the safe integer limit in JSON/JavaScript) is pathological and SHOULD be terminated.

# IANA Considerations

## Media Type Registration: application/llm-request+json

Type name:
: application

Subtype name:
: llm-request+json

Required parameters:
: None

Optional parameters:
: version (default: "1.0")

Encoding considerations:
: 8bit (UTF-8 JSON)

Intended usage:
: COMMON

## SSE Event Type Registry

This document requests creation of an IANA registry for LLM-Stream SSE event types:

| Event Type | Reference |
|---|---|
| stream.start | {{stream-start}} |
| content.delta | {{content-delta}} |
| content.stop | {{content-stop}} |
| tool.call | {{tool-call}} |
| usage | {{usage}} |
| error | {{error}} |
| stream.end | {{stream-end}} |

New event types are registered via Specification Required {{RFC8126}}.

--- back

# JSON Schema for Event Envelope

~~~json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "urn:ietf:params:llm-stream:event-envelope:1.0",
  "type": "object",
  "required": ["v", "seq", "type"],
  "properties": {
    "v": {"type": "string", "const": "1.0"},
    "seq": {"type": "integer", "minimum": 0},
    "type": {
      "type": "string",
      "enum": ["stream.start", "content.delta",
               "content.stop", "tool.call", "usage",
               "error", "stream.end"]
    }
  },
  "additionalProperties": true
}
~~~

# Observed Protocol Landscape (June 2026)

| Provider | Transport | Framing | API Type |
|---|---|---|---|
| OpenAI | HTTP/2 + SSE | SSE data: lines | Public |
| Anthropic | HTTP/2 + SSE | SSE event: + data: | Public |
| Google Gemini | HTTP/2 + SSE | SSE data: lines | Public |
| Microsoft/Azure | WebSocket | SignalR 0x1E | Proprietary |
| Cursor/Windsurf | HTTP/2 | Connect binary | Proprietary |

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
