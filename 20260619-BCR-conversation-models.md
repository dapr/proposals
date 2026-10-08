# Remove `model` from the Conversation building block specification

- Author: Whit Waldo @whitwaldo
- State: Proposed
- Introduced: 2026-06-19

## Overview
The Conversation building block is in an Alpha state and subject to breaking changes; this is the time to make one. When the infrastructure team defines a Conversation component today, they provide the component name, block type/version/provider, connection details (endpoint, auth), the provider configuration, Dapr functionality configuration, and the model name.

I propose removing this last item. The `model` property leaves the component spec and becomes a required property on the conversation request. This is one conceptual change with the follow-through it implies: proto, runtime (including caching), SDKs, and documentation. 

Downstream enhancements are deferred to separate proposals.

## Background

### Motivation
The Conversation building block landed with the model named in the component, which was a sensible choice when the supported providers each exposed only a handful of models. As the provider set has grown, the most common request I've seen (both in Discord and in person) has been a way to choose the model without standing up a near-duplicate component for each one. That pressure is only increasing: Ollama already supports 100+ models and a provider like OpenRouter (400+) would make a component-per-model approach impractical for an infrastructure team to reasonably maintain.

One natural response is to let the request override the component's model. I don't think this is the right direction. Overrides raise their own questions about which properties are overridable, who owns a connection whose values can change at runtime, and how Dapr functionality layered on top (caching, for example) behaves when they do. Rather than answer those, I think the cleaner move is to reconsider where the model belongs in the first place.

### Model is a runtime concern
Dapr components have consistently separated connectivity from operation: the component defines how to reach a provider and the scope of access, and the developer supplies the operational specifics at runtime. The model sits squarely on the operational side of that line; it is neither connectivity nor a scope boundary. Removing it from the component and making it a required property on the request puts it where Dapr already puts comparable choices, and it does so without introducing an override mechanism at all, since there's no component-owned model left to override.

This pattern holds across the existing building blocks:
- State: The component fixes block type, version, connection and provider options; the developer freely chooses the state capabilities to use and keys at runtime.
- PubSub: The component defines connectivity and provider options; the developer chooses the topic at runtime, within the component's scope.
- Cryptography: The component defines connectivity to the vault and the provider options; the developer names the key and selects the algorithm per operation.

Crypto also speaks to the obvious follow-up: "What if the developer asks for a model the provider can't serve?" Dapr already accepts that a component guarantees connectivity and scope, not operational correctness. Name a key that isn't in the vault, or request an operation incompatible with the key (an asymmetric operation against a symmetric-only key, say) and the call fails at runtime, by design. This is documented and presumes the developer knows something about the infrastructure even if they don't own the connection to it. An unsupported model (by provider or for purpose) fails the same way, for the same reason - no new failure mode is introduced.

## Implementation Details
1. Remove `model` from the Conversation provider component specs that support it. This likely touches the corresponding implementations in components-contrib.
2. Add `Alpha3` request/response types to the protos. `ConversationRequestAlpha3` is identical to `ConversationRequestAlpha2` except `model` is treated as required.

```protobuf
message ConversationRequestAlpha3 {
  // The name of Conversation component
  string name = 1;

  // The name of the large language model to engage for the request.
  // NOTE: Validated as non-empty by the SDK/runtime
  string model = 2;

  // The ID of an existing chat (like in ChatGPT)
  optional string context_id = 3;

  // Inputs for the conversation.
  repeated ConversationInputAlpha3 inputs = 4;

  // Parameters for all custom fields.
  map<string, google.protobuf.Any> parameters = 5;

  // Set of 16 key-value pairs that can be attached to the conversation. 
  // This can be useful for storing additional information about the object in a structured format, 
  // and querying for objects via API or the dashboard.
  // Keys are strings with a maximum length of 64 characters.
  // Values are strings with a maximum length of 512 characters.
  // NOTE: In the next iteration of this API, this will be within the HTTP/gRPC headers instead.
  map<string, string> metadata = 6;
  
  // Scrub PII data that comes back from the LLM
  optional bool scrub_pii = 7;

  // Temperature for the LLM to optimize for creativity or predictability
  optional double temperature = 8;

  // Tools register the tools available to be used by the LLM during the conversation.
  // These are sent on a per request basis.
  // The tools available during the first round of the conversation
  // may be different than tools specified later on.
  repeated ConversationTools tools = 9;

  // Controls which (if any) tool is called by the model. 
  // `none` means the model will not call any tool and instead generates a message. 
  // `auto` means the model can pick between generating a message or calling one or more tools.
  // Alternatively, a specific tool name may be used here, and casing/syntax must match on tool name.
  // `none` is the default when no tools are present.
  // `auto` is the default if tools are present.
  // `required` requires one or more functions to be called.
  // ref: https://github.com/openai/openai-go/blob/main/chatcompletion.go#L1976
  // ref: https://python.langchain.com/docs/how_to/tool_choice/
  optional string tool_choice = 10;

  // Structured outputs described using a JSON Schema object.
  // Use this when you want strict, typed structured output.
  // This corresponds to OpenAI's:
  //   { "type": "json_schema", "json_schema": { ... } }
  //
  // The schema must be provided as a parsed JSON object.
  // Note: This is currently only supported by OpenAI components.
  // This is only supported by Deepseek, GoogleAI, HuggingFace, OpenAI, and Anthropic.
  // inspired by openai.ResponseFormat
  // ref: https://github.com/openai/openai-go/blob/main/chatcompletion.go#L3111
  optional google.protobuf.Struct response_format = 11;

  // Retention policy for the prompt cache.
  // If using OpenAI with this value set to `24h` it enables extended prompt caching,
  // which keeps cached prefixes active for longer, up to a maximum of 24 hours.
	// [Learn more](https://platform.openai.com/docs/guides/prompt-caching#prompt-cache-retention).
  // inspired by openai.ChatCompletionMessageParamUnion.PromptCacheRetention
  // ref: https://github.com/openai/openai-go/blob/main/chatcompletion.go#L3030
  optional google.protobuf.Duration prompt_cache_retention = 12;
}
```

Making the model required is a deliberate trade-off: it buys uniform, early validation across every provider at the cost of a "use the provider's default model" affordance. The only provider that offers that default today is [DeepSeek](https://v1-18.docs.dapr.io/reference/components-reference/supported-conversation/deepseek/), and only as an accident of history (I suspect): its component shipped when DeepSeek exposed a single model, so omitting the model made sense at the time. DeepSeek now exposes [multiple models](https://api-docs.deepseek.com/) (soon to be two), so that default is stale regardless. Under this change, DeepSeek's model becomes a required request property like every other provider's, which alleviates the impact of the change for that component; the broader multi-model cleanup for DeepSeek is a separate work item.

3. Key Conversation caching on a hash of (component name, request model) rather than component name alone. This is strictly more correct than today: the moment model can vary per request, a component-name-only key would conflate distinct models. This change fixes this latent conflation rather than introducing new behavior.
4. Update the SDKs - this is an easy change per SDK. The model joins the component name as a required request parameter. In .NET for example, it becomes a required property on `ConversationOptions`:
```csharp
var response = await conversationClient.ConverseAsync(
    [
        new ConversationInput(
            new List<IConversationMessage>
            {
                new UserMessage
                {
                    Name = "Test User",
                    Content =
                    [
                        new MessageContent(
                            "Please write a witty haiku about the Dapr distributed programming framework at dapr.io")
                    ]
                }
            }
        )
    ],
    new ConversationOptions("conversation", "kimi-k2.7-code")
);
```

### Interaction with `context_id`
With model now per-request, a follow-up request that reuses an existing `context_id` may carry a different model than the request that opened the context. Whether switching models within a single context is meaningful is provider-defined; the runtime makes no guarantee here, consistent with the Crypto stance above. The model binds per request.

## Backwards Compatibility
This is a breaking change, but the Conversation block is in alpha, so this is the time to do this.

## Acceptance Criteria
- [ ] New `Alpha3` conversation prototypes are created in the dapr/dapr repository
- [ ] Runtime: implement the changes to conversation support including caching
- [ ] SDK: Go
- [ ] SDK: .NET
- [ ] SDK: Rust
- [ ] SDK: Java
- [ ] SDK: Python
- [ ] SDK: Node
- [ ] Update documentation to reflect required new `model` property on requests, removed from component

## Summary
This change moves the model from the component to the request, aligning the Conversation block with Dapr's established separation of connectivity from operation: the component dictates how to connect to a provider and bounds the scope of access, while the developer drives operational use - here, the specific model - at runtime. It answers a frequent and reasonable developer request, requires only a small and largely straightforward set of changes, and keeps Dapr nimble in a fast-moving agentic space.

## Future Work
An optional `allowlist`/`denylist` on the component, with pattern matching, would let infrastructure teams explicitly permit or bar specific models through a given provider. This is an additive capability for organizations that want to constrain developer choice, deferred here as an enhancement rather than a requirement. Runtime recently worked through similar allow/deny semantics for another feature, so the concept could likely be adapted here rather than reinvented with different rules.  