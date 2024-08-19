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

## Metadata

```yaml

key: # string
model: # string
endpoints: # []string
loadBalancingPolicy: # string, support "ROUNDROBIN"

```

## gRPC APIs

In the Dapr gRPC APIs, we are extending the `runtime.v1.Dapr` service to add new methods:

> Note: APIs will have "Alpha1" added while in preview

> Note: The API token is stored in context

```proto
// (Existing Dapr service)
service Dapr {
  // Conversate.
  rpc Converse(stream ConversationRequest) returns (stream ConversationResponse);
}

// ConversationRequest is the request object for Conversation.
message ConversationRequest {
  // The name of Coverstaion component
  string name = 1;
  // Inputs for the conversation, support multiple input in one time.
  repeated string inputs = 2;
  // Parameters for all custom fields.
  repeated google.protobuf.Any parameters = 3;
  // The metadata passing to converstion components
  //
  // metadata property:
  // - key : the key of the message.
  map<string, string> metadata = 4;
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
  // An array of results.
  repeated ConversationResult outputs = 1;
}
```

## HTTP APIs

The HTTP APIs are same with the gRPC APIs：

`POST /v1.0/conversation/[component]/converse` -> Conversate

```json
REQUEST = {
    "metadata": {
      "model": "gpt-4o",
      "endpoint": "api.openai.com",
      "key": "token-key",
      "policy": "ROUNDROBIN",
  },

  "inputs": ["what is Dapr", "Why use Dapr"],
  "parameters": {},

}

RESPONSE  = {
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
