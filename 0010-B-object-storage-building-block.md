# Object storage building block

* Author(s): @ItalyPaleAle
* State: Draft
* Updated: 2022-12-14
* Original proposals / discussions: dapr/dapr#4808, dapr/dapr#4934

## Overview

This is a design proposal for a new "object storage" building block which allows Dapr users to store and retrieve unstructured data of arbitrary size.

We define **objects** as sequences of unstructured data, that should be assumed to be possibly large (many MBs or even GBs in size) and binary. Examples include images and videos, Office documents, etc.

## Implementation Details

### Building block interface

The proposal involves creating a new `objectstore` building block with the following interface:

```go
type ObjectStore interface {
    // GetObject retrieves an object with name key and writes it to the out stream.
    // If the object doesn't exist, err will be ErrNotFound.
    // Return value tagOut contains an etag or a similar concurrency-control tag.
    GetObject(ctx context.Context, key string, out io.Writer) (md ObjectMetadata, tagOut any, err error)
    // SetObject stores an object with name key, reading data from the in stream.
    // Parameter tag is optional and can be used to pass an etag or any similar concurrency-control tag.
    // In case of conflict (e.g. etag mismatch), the method returns ErrConflict.
    SetObject(ctx context.Context, key string, in io.Reader, tag any, md ObjectMetadata) (tagOut any, err error)
    // DeleteObject deletes an object with name key.
    // Parameter tag is optional and can be used to pass an etag or any similar concurrency-control tag.
    // In case of conflict (e.g. etag mismatch), the method returns ErrConflict.
    DeleteObject(ctx context.Context, key string, tag any) (err error)
}

// Metadata associated with an object.
type ObjectMetadata map[string]string

// Error constants:
var ErrNotFound = errors.New("...")
var ErrConflict = errors.New("...")
```

The `key` parameter is the name of the object. Unlike with state stores, there are no key prefixes when operating with objects: applications have full access to all data in the storage service.

Each object can have metadata associated to it, which is normally passed to clients as headers (more on that below). In particular, `Content-Type` is yet another metadata key, treated no differently from anything else.

> Option: we could support auto-detection of the content type if users pass `"auto"` as value, using a library such as [ItalyPaleAle/file-type-stream-go](https://github.com/ItalyPaleAle/file-type-stream-go) (which I'd be very happy to transfer to the Dapr org).

The only exception is the `tag`, which is passed separately from the metadata object. The reason is that in the `SetObject` method, the tag is both an (optional) parameter and a return value. Note however that support for tags / ETags, is not a requirement if the underlying storage service doesn't support it.

When invoking `GetObject` or `SetObject`, the output and input streams (respectively) are created outside of the state store and are owned by the caller. The `GetObject` and `SetObject` methods synchronously return only after all data has been written or read (respectively), and after that the caller can call `Close()` on the streams (if necessary/appropriate).

Notes:

- This building block will be stream-first. It will be working with input and output data as binary streams, and will not perform any transformation or encoding on the data. Thanks to using streams, there are no limits on the size of the input/output data, bypassing `MaxBodySize`.
- Dapr only supports the "last-write-wins" concurrency pattern for object storage (unlike with state stores). This makes sense considered the intended usage of object storage, which is storing documents that usually do not change. When documents do change, using last-write-wins is consistent with how regular filesystems work, for example.
- Dapr will not calculate and store the checksum of objects. This is because checksums must be transmitted as header, but Dapr stores data in the state store in a streaming way, so it can't compute the full checksum until the end of the stream. Apps that want to store checksums (such as the `Content-MD5` header) should compute it beforehand and submit it as metadata header.

### HTTP APIs

Applications that interact with object storage using HTTP can leverage these APIs, that follow a more REST-like pattern than the typical state store APIs.

> These APIs have been modeled after the APIs supported by AWS S3 and Azure Blob Storage.

#### Retrieving an object

To retrieve an object, clients make a GET request:

```text
GET http://localhost:<port>/v1.0-alpha1/objects/<component>/<key>
```

Where `<component>` is the name of the object store component, and `<key>` is the key (name) of the object.

A successful response will be:

- Status: `200 OK`
- Headers: each metadata key is a header; see below for a comment on headers. Additionally, the `ETag` is passed as response header too.
- Body: the body of the response is the raw bytes of the object, which are sent to the client in a streamed way, as soon as they are retrieved from the state store (that is: Dapr does not buffer the data in-memory before sending it to the client).

An error response will have:

- Status:
  - `404 Not Found` for objects not found
  - `400 Bad Request` if the state store doesn't exist or the key parameter is missing
  - `500 Internal Server Error` for all other errors
- Headers: response will have `Content-Type: application/json`
- Body: a JSON-encoded payload including more details on the error

#### Storing an object

To store an object, clients make a PUT request:

```text
PUT http://localhost:<port>/v1.0-alpha1/objects/<component>/<key>
```

Where `<component>` is the name of the object store component, and `<key>` is the key (name) of the object.

The request body contains the raw bytes of the object, which are read by Dapr in a streamed way and are sent to the object store as soon as they are available (once again, Dapr does not buffer the data in-memory before storing it).

Additionally, metadata keys are passed as header values; see below for a comment on headers. An `ETag` can be passed as request header too for concurrency control, and it's ignored if the object doesn't exist already. If an ETag is not specified, the write will always succeed (unless other errors occur).

A response will be:

- Status:
  - `200 OK` for successful operations
  - `400 Bad Request` if the state store doesn't exist or the key parameter is missing
  - `409 Conflict` if the object already exists and there's an ETag mismatch
  - `500 Internal Server Error` for all other errors
- Headers: response will have `Content-Type: application/json`
- Body:
  - For successful responses: a JSON-encoded payload containing the ETag and the number of bytes stored.
  - In case of errors: a JSON-encoded payload including more details on the error

#### Deleting an object

To delete an object, clients make a DELETE request:

```text
DELETE http://localhost:<port>/v1.0-alpha1/objects/<component>/<key>
```

Where `<component>` is the name of the object store component, and `<key>` is the key (name) of the object.

An `ETag` can be passed as request header too for concurrency control. If an ETag is is specified, the operation will fail if the ETag doesn't match; the ETag is optional and when it's missing there's no concurrency control.

A response will be:

- Status:
  - `204 No Content` for successful operations
  - `400 Bad Request` if the state store doesn't exist or the key parameter is missing
  - `409 Conflict` if the object already exists and there's an ETag mismatch
  - `500 Internal Server Error` for all other errors
- Headers: error responses will have `Content-Type: application/json`
- Body:
  - For successful responses, the response body is empty
  - In case of errors: a JSON-encoded payload including more details on the error

### gRPC APIs

For applications using gRPC to interact with Dapr, the `Dapr` service is expanded to include:

```proto3
// The "Dapr" service already exists
service Dapr {
  // GetObject retrieves an object.
  rpc GetObject(GetObjectRequest) returns (stream GetObjectResponse) {}

  // SetObject stores an object with name key.
  rpc SetObject(stream SetObjectRequest) returns (SetObjectResponse) {}

  // DeleteObject an object with name key.
  rpc DeleteObject(DeleteObjectRequest) returns (DeleteObjectResponse) {}
}

// GetObjectRequest is the message to get an object from specific object store.
message GetObjectRequest {
  // The name of object store.
  string store_name = 1;

  // The key of the desired object.
  string key = 2;
}

// GetObjectResponse contains the retrieved object (this is used in a streamed response).
message GetObjectResponse {
  // The tag for concurrency control.
  string tag = 1;

  // The metadata stored with the object.
  map<string, string> metadata = 2;

  // Chunk of data.
  StreamPayload payload = 10;
}

// SetObjectRequest is the message to store a object in a specific object store (this is used in a streamed request).
message SetObjectRequest {
  // The name of object store.
  string store_name = 1;

  // The key of the desired object.
  string key = 2;

  // The tag for concurrency control.
  string tag = 3;

  // The metadata which will be stored with the object.
  map<string, string> metadata = 4;

  // Chunk of data.
  StreamPayload payload = 10;
}

// SetObjectResponse contains the result of storing the object.
message SetObjectResponse {
  // The updated tag for concurrency control.
  string tag = 1;

  // The number of bytes written.
  int bytes = 2;
}

// DeleteObjectRequest is the message to delete an object in specific object store.
message DeleteObjectRequest {
  // The name of object store.
  string store_name = 1;

  // The key of the desired object.
  string key = 2;
}

// DeleteObjectResponse contains the result of deleting the object.
message DeleteObjectResponse {
    // Currently empty but allowing for future expansion.
}
```

#### Handling streams

`StreamPayload` is first introduced with dapr/dapr#4903 and corresponds to:

```proto3
// Chunk of data sent in a streaming request or response.
message StreamPayload {
  // Data sent in the chunk.
  google.protobuf.Any data = 1;

  // Set to true if this is the last chunk.
  bool complete = 2;
}
```

Data is sent from the application to Dapr (in `SetObject` RPCs) or from Dapr to the application (in `GetObject` RPCs) in a stream. Each message in the stream contains a `GetObjectResponse` or `SetObjectRequest` where:

- The first message in the stream MUST contain all other required keys.
- The first message in the stream MAY contain a `payload`, but that is not required.
- Subsequent messages (any message except the first in the stream) MUST contain a `payload` and MUST NOT contain any other property.
- The last message in the stream MUST contain a `payload` with `complete` set to `true`. That message is assumed to be the last one from the sender and no more messages are to be sent in the stream after that.

The amount of data contained in `payload.data` is variable and it's up to the discretion of the sender. In service invocation calls, as implemented by dapr/dapr#4903, chunks are at most 4KB in size, although senders may send smaller chunks if they wish. Receivers must not assume that messages will contain any specific number of bytes in the payload.

Note that it's possible for senders to send a single message in the stream. If the data is small and could fit in a single chunk, senders MAY choose to include a `payload` with `complete=true` in the first message. Receivers should assume that single message to be the entire communication from the sender.

### Metadata and headers

Metadata properties are passed between the client and server as headers.

There can be two types of headers:

- Custom ones have the `x-dapr-` prefix in requests sent between Dapr and the app. When they are stored, they generally do not have the `x-dapr-` prefix, which is removed (although this could be dependent on the implementation of the state store).
- Certain common headers are exchanged between apps and Dapr in their canonical form. The way these are stored in the state store depends on the service and implementation:
  - `Last-Modified`
  - `Content-Length`
  - `Content-Type`
  - `Content-MD5`
  - `Content-Encoding`
  - `Content-Language`
  - `Cache-Control`
  - `Origin`
  - `Range`
  - Additionally, the `ETag` header is exchanged in the request/response headers, although it's not included in the metadata object.
