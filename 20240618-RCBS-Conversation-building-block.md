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

## Components

Components in dapr/components-contrib are to be placed in the `conversation` folder and must implement the `Conversation` interface:

```go
// Converse offers an interface to perform low-level conversational operations
type Conversation interface {
    // Converse one conversation
    Converse(
        // Context that can be used to cancel the running operation
        ctx context.Context,
        // Endpoint for model service
        endpoint string,
        // Name of the model
        model_name string,
        // Inputs for the conversation, support multiple input in one time
        inputs []byte,
        // Parameters for all custom fields
        parameters map[string]string,
        // Key of API token
        key string,
    ) (
        // An array of conversation results.
        []map[string]string outputs
        // Error
        err error,
    )
}
```

## gRPC APIs

In the Dapr gRPC APIs, we are extending the `runtime.v1.Dapr` service to add new methods:

> Note: APIs will have "Alpha1" added while in preview

> Note: The API token is stored in context

```proto
// (Existing Dapr service)
service Dapr {
  // Conversate.
  rpc Converse(ConversationRequest) returns (ConversationResponse);
}

// ConversationRequest is the request object for Conversation.
message ConversationRequest {
  enum LoadBalancingPolicy {
    // Round robin policy.
    ROUNDROBIN = 0;
  }

  // Endpoints for the model service, co-work with load balanace policy.
  repeated string endpoints = 1;
  // Name of the model.
  string model_name = 2;
  // Inputs for the conversation, support multiple input in one time.
  repeated string inputs = 3;
  // Parameters for all custom fields.
  repeated google.protobuf.Any parameters = 4;
  // Key of API token
  string key = 5;
  // Load balancing policy for endpoints.
  LoadBalancingPolicy policy = 6;
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

> Note: URL will begin with `/v1.0-alpha1` while in preview
