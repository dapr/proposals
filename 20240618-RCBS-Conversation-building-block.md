# Conversation building block

* Author(s): Loong Dai (@daixiang0)
* Updated: 2024-06-18

## Overview

This is a proposal for a new building block for Dapr to allow developers to leverage LLM services in a consistent way. Goal is to expose an API that allows developers to ask Dapr to do request.

## Background

Now there are many large language model servers or toolkits, which provides own APIs, like [OpenAI](https://openai.com/), [Hugging Face](https://huggingface.co/), [Kserve](https://kserve.github.io/website/latest/), [OpenVINO](https://docs.openvino.ai/) and so on.

For developers, it is hard to migrate from one to the other due to hardcode and API differences.

For startups and communities, they need to implement the popular APIs as soon as possible, or users maybe give up tring or adopting because of the cost of migration.

This is an area where Dapr can help. We can offer an abstraction layer on those APIs.

## Component YAML

A component can have it's own set of attributes, like in Dapr. For example:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: chatgpt4o
spec:
  type: conversation.chatgpt
  version: v1
  metadata:
    - name: key
      value: "bfnskdlgdhklhk53adfgsfnsgmtyqdghbid34891"
    - name: model
      value: "gpt-4o"
    - name: endpoints
      value: "us.api.openai.com,eu.api.openai.com"
    - name: loadBalancingPolicy
      value: "ROUNDROBIN"
```

## gRPC APIs

In the Dapr gRPC APIs, we are extending the `runtime.v1.Dapr` service to add new methods:

> Note: APIs will have "Alpha1" added while in preview

> Note: The API token is stored in the component

```proto
// (Existing Dapr service)
service Dapr {
  // Conversate.
  rpc Converse(stream ConversationRequest) returns (stream ConversationResponse);
}

// ConversationRequest is the request object for Conversation.
message ConversationRequest {
  // The name of Conversation component
  string name = 1;
  // Conversation context - the Id of an existing chat room (like in ChatGPT)
  optional string conversationContext = 2;
  // Inputs for the conversation, support multiple input in one time.
  repeated string inputs = 3;
  // Parameters for all custom fields.
  repeated google.protobuf.Any parameters = 4;
}

// ConversationResult is the result for one input.
message ConversationResult {
  // Result for the one conversation input.
  string result = 1;
  // Parameters for all custom fields.
  repeated google.protobuf.Any parameters = 2;
}

// ConversationResponse is the response for Conversation.
message ConversationResponse {
  // Conversation context - the Id of an existing or newly created chat room (like in ChatGPT)
  optional string conversationContext = 1;

  // An array of results.
  repeated ConversationResult outputs = 2;
}
```

## HTTP APIs

The HTTP APIs are same with the gRPC APIs：

`POST /v1.0/conversation/[component]/converse` -> Conversate

```json
REQUEST = {
  "conversationContext": "fb512b84-7a1a-4fb4-8bd2-ac7d2ec45984",
  "inputs": ["what is Dapr", "Why use Dapr"],
  "parameters": {},
}

RESPONSE  = {
  "conversationContext": "fb512b84-7a1a-4fb4-8bd2-ac7d2ec45984",
  "outputs": {
    {
       "result": "Dapr is distribution application runtime ...",
       "parameters": {},
    },
    {
       "result": "Dapr can help developers ...",
       "parameters": {},
    }

  },
}
```

> Note: URL will begin with `/v1.0-alpha1` while in preview
