# Binary storage building block
- Author(s): @WhitWaldo
- Significantly inspired by: @ItalyPaleAle (https://github.com/dapr/proposals/pull/18), @artursouza (https://github.com/dapr/dapr/issues/4804)
- Name inspired by @olitomlinson
- Updated October 14, 2026

> **Implementation status:** This proposal reflects the implementation in
> [dapr/dapr#10367](https://github.com/dapr/dapr/pull/10367),
> [dapr/components-contrib#4559](https://github.com/dapr/components-contrib/pull/4559), and
> [dapr/components-contrib#4637](https://github.com/dapr/components-contrib/pull/4637).

## Overview
This is a design proposal for building a new [specialized state store](https://github.com/dapr/dapr/issues/7339) for
managing named files in state providers. I intentionally use the word "file" as opposed to blob or object as to avoid 
any of the overloaded meaning associated with common object store or blob store providers. I think there's every 
possibility that it'll make sense to introduce a proper "blob store" or "object store" building block in the future. 
However, for now I think it's best to keep the scope of this proposal focused on files more generally and avoid the 
complexities of either of those specialty stores. Over time, the distinction between this and another Blob Store or 
Object Store block should be made clearer by the enhanced capabilities of either of those APIs (e.g., list by prefix 
or handling metadata). However, this proposal should be understood to be a bare-boned implementation designed to 
strictly store and retrieve (potentially) large serialized files.

Files are any contiguous sequences of unstructured data comprised of bytes that should be assumed to span lengths as 
few as a handful of bytes to several gigabytes. Examples include images, videos, documents, database backups, 
compressed archives and so on. This original version of this proposal referred to this being a `FileStore`, but after
talking about it with Oli, I've warmed up to instead calling it the `BinaryStore`. This is better than even `DataStore`
because the intent is that the data will already have been serialized and encoded so it's ready to persist somewhere. 
While this might be from a file, it doesn't necessarily have to be, so much as that it's just an array (or while in 
transit, a stream) of bytes. Referring to it as a `BinaryStore` also rules out any confusion about whether it's 
abstracting SMB or NFS.

## Background
Dapr currently implements a state store around a strictly key/value basis. This approach has led to component support
across many different component providers and works for many common use cases. However, decisions around the design
and implementation of the state store feature make it increasingly important to introduce a new generation of 
special-purpose state management building blocks instead of having a one-size-fits-all approach to managing state.

For example, files differ from values in key/value stores in that they can frequently span large amounts of data that
isn't generally compatible with the likes of Redis or Etcd. Because they span such large amounts of data, it's frequently
insufficient to simply store and retrieve them synchronously, mandating a stream-first approach to minimize
resources needed by the runtime to transfer the data. The goal is to allow users to store and retrieve binary blobs in a 
performant and scalable manner via a common runtime API and enable the Dapr SDKs to complement the implementation to
support additional provider-specific features on a case by case basis. A possible example of this will be given below.

## Component YAML
This component is expected to have similar attributes to existing state stores with some variation:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: binarystore
spec:
  type: binarystore.<providerName>
  version: v1
  metadata:
    - name: prefix
      value: my-prefix
    # Other properties necessary for component registration
```

The initial implementation includes the following component types:

- `binarystore.in-memory`
- `binarystore.aws.s3`
- `binarystore.azure.blobstorage`
- `binarystore.azure.datalake`
- `binarystore.gcp.bucket`
- `binarystore.oci.objectstorage`

All initial providers support an optional component-level `prefix` that is prepended to file names. Cloud providers
also expose an optional `partSize` metadata property, defaulting to `16Mi`, to balance request overhead, maximum
object size and memory used by each in-flight upload. The in-memory provider instead exposes `maxSize` (default
`4Mi`) and `maxTotalSize` (default `512Mi`) to bound both individual files and total memory consumption.

## Implementation Overview
As touched on above, there are a few guiding principles I want to carry through from previous iterations of this 
concept:
- All operations should be asynchronously performed with the sidecar
- When persisting or retrieving state, this should be done using an asynchronous stream to maximize SDK and runtime
performance while minimizing resource overhead and preventing memory exhaustion

The implemented component contract exposes `Init`, `Features`, `Set`, `Get`, `Delete`, `GetComponentMetadata` and
`Close`. `Set` receives an
`io.Reader`, while `Get` returns an `io.ReadCloser`, allowing data to be transferred without buffering the entire file
in either the runtime or component. `Set` and `Get` do not use runtime retries, circuit breakers or per-call resiliency
timeouts because their streams cannot be replayed and a timeout would bound the entire transfer. `Delete` continues
to use the configured component resiliency policy.

To prevent an endless or oversized upload from exhausting sidecar or component memory, each provider stops reading
when its configured or service-specific maximum is exceeded and returns a file-too-large error. The runtime maps this
to HTTP `413 Payload Too Large` or gRPC `RESOURCE_EXHAUSTED`. The in-memory provider additionally returns a store-full
error, mapped to HTTP `507 Insufficient Storage` or gRPC `RESOURCE_EXHAUSTED`, when accepting an upload would exceed
`maxTotalSize`. The HTTP upload endpoints opt out of Dapr's global request-body size limit (`SkipRequestBodySizeLimit`)
so legitimate large files can be streamed; the exemption is matched by route and only applies to endpoints permitted by
the API allowlist/denylist, so a denied endpoint cannot be used to bypass the limit. Provider limits remain in force.

Provider limits in the initial implementation are:

| Provider | Upload limit behavior |
| --- | --- |
| In-memory | Configurable `maxSize` per file and `maxTotalSize` across stored and in-flight data |
| AWS S3 | Lesser of 50 TB or 10,000 configured-size parts; `partSize` must be 5 MiB-5 GiB |
| Azure Blob Storage | 50,000 configured-size blocks; `partSize` must be 1 byte-4000 MiB |
| Azure Data Lake Storage Gen2 | 5 TB; `partSize` must be 1 byte-4000 MiB |
| Google Cloud Storage | 5 TiB; `partSize` must be a positive multiple of 256 KiB |
| OCI Object Storage | Lesser of 10 TiB or 10,000 configured-size parts; `partSize` must be 10 MiB-50 GiB |

### Make Runtime APIs Simpler
I'd like to make the runtime APIs as simple as possible so they're more readily used in a composable way alongside other
Dapr APIs by the SDKs, pluggable components or other developer-written utilities. With that in mind, I do not include
any support for persisting metadata with the files or listing the files in the store in this API. This ensures
broader compatibility with possible stores for file-based data and leaves the possibility of greater sophistication
to the SDKs to handle instead.

This approach allows for a leaner, more provider-agnostic API, takes work off the plate of the runtime developers and
better positions the SDKs to provide a more comprehensive experience leveraging existing building blocks, improving
and simplifying the developer experience.

> **NOTE** It would be important to call out in the developer documentation for this building block that the metadata 
> and listing information returned is not provided by the state provider. Rather, it is instead provided based on 
> values provided by the developer and are not necessarily reflective of the actual state of the store. This could 
> provide a key distinction between this and an actual object or blob store for those that need that level of detail 
> (with the tradeoff that fewer providers would be supported in the other implementation).


#### Example: C# SDK
For example, in the C# SDK, the developer might specify a key/value store of their choosing to persist metadata
during the dependency injection registration alongside the name of the file store component:

```csharp
services.AddDapr()
    .WithFileStore(opt => 
    {
        opt.BinaryStoreName = "my-file-store"; // Defines the binary store to use for this injected `DaprBinaryStore` type
        opt.MetadataStoreName = "my-metadata-store"; // Defines the metadata store to persist metadata to
    });
```

This would provide a simple approach that allows the developer to use the SDK to persist files that optionally include
metadata with a DI-registered type. This would avoid the current approach of having to pass in the store names each time.

```csharp
public sealed class MyClass(DaprBinaryStore binaryStore)
{
    public async Task SaveAsync(string name, byte[] data) 
    {
        await binaryStore.SetAsync(name, data);
    }
    
    public async Task SaveWithMetadataAsync(string name, byte[] data, IReadOnlyDictionary<string, string> metadata )
    {
        if (string.IsNullOrWhitespace(binaryStore.MetadataStoreName)) 
            throw new ArgumentException("Metadata store name must be set in the DaprFileStoreOptions during DI registration");
        
        await binaryStore.SetAsync(name, data, metadata);
    }
}
```

Further, the C# SDK could provide a trivial expansion on this idea to allow named types for saving data to different 
places throughout the application. This example uses values from a shared configuration while still relying on Dapr to 
manage the type lifecycle:

```csharp
services.AddDapr()
    .WithBinaryStore("storeA", (serviceProvider, options) => 
    {
        // Populate the file store and metadata store names from a configuration
        var configuration = serviceProvider.GetRequiredService<IConfiguration>();
        var binaryStoreName = configuration.GetValue<string>("BinaryStoreName");
        var metadataStoreName = configuration.GetValue<string>("MetadataStoreName");
        
        options.FileStoreName = fileStoreName;
        options.MetadataStoreName = metadataStoreName;        
    })
    .WithBinaryStore("storeB", options => 
    {
        // Use static values to populate the names in this named `DaprFileStore` client
        options.BinaryStoreName = "my-binary-store";
        options.MetadataStoreName = "my-metadata-store";
    });
```

Because I don't include list support in this API either, the SDK can again be relied on to abstract this away for the 
developer. Paired with the next-generation key/value store I [have proposed](https://github.com/dapr/dapr/issues/7338),
the developer might query the list of files in the store by instead querying the metadata key/value store for any
keys with the file name prefix they're interested in:

```csharp
public sealed class MyClass(DaprBinaryStore binaryStore)
{
    public async Task<IReadOnlyCollection<string>> GetFileNamesAsync(string prefix)
        => await binaryStore.ListFilesAsync(prefix);
}
```

### gRPC APIs
In the Dapr gRPC APIs, we'd either extend the `runtime.v1.Dapr` service to add new methods or create a separate service
specific to the file store. The benefit of taking the latter approach is that it need only be updated when the API
changes and otherwise won't trigger downstream operations monitoring changes to that file. However, this approach
generally falls outside the scope of this proposal and we'll assume that `runtime.v1.Dapr` is simply extended with the
following:

| Note: Consistent with current practice, APIs will have Alpha1 suffixed to the type names while in preview.

| Note: Any authentication behaviors are maintained in the component YAML configuration.

```proto
// Existing Dapr service
service Dapr {
    rpc SetBinaryFileAlpha1(stream SetBinaryFileRequestAlpha1) returns (SetBinaryFileResponseAlpha1) {}
    rpc GetBinaryFileAlpha1(GetBinaryFileRequestAlpha1) returns (stream GetBinaryFileResponseAlpha1) {}
    rpc DeleteBinaryFileAlpha1(DeleteBinaryFileRequestAlpha1) returns (DeleteBinaryFileResponseAlpha1) {}
}

message SetBinaryFileRequestAlpha1 {
    oneof set_binary_file_request_type {
        // Must be present in the first message only.
        SetBinaryFileRequestInitialAlpha1 initial_request = 1;
        // Chunk of data of arbitrary size.
        common.v1.StreamPayload payload = 2;
    }
}

message SetBinaryFileRequestInitialAlpha1 {
    string component_name = 1 [json_name="componentName"];
    string file_name = 2 [json_name="fileName"];
    optional bool overwrite = 3 [json_name="overwrite"];
}

message SetBinaryFileResponseAlpha1 {}

message GetBinaryFileRequestAlpha1 {
    string component_name = 1 [json_name="componentName"];
    string file_name = 2 [json_name="fileName"];
}

message GetBinaryFileResponseAlpha1 {
    common.v1.StreamPayload payload = 1;
}

message DeleteBinaryFileRequestAlpha1 {
    string component_name = 1 [json_name="componentName"];
    string file_name = 2 [json_name="fileName"];
}

message DeleteBinaryFileResponseAlpha1 {}
```

For `SetBinaryFileAlpha1`, the first message must contain `initial_request`; subsequent payload messages use
contiguous sequence numbers beginning at zero. The runtime waits up to five seconds for the first message and
rejects another initial request later in the stream. `overwrite` defaults to `false`, matching POST/create-only
semantics. Retrieval responses are emitted in 32 KiB payload chunks with contiguous sequence numbers.

File names may contain slash-delimited path segments, but must not be empty, begin or end with `/`, contain `//`, use
`.` or `..` as a segment, or contain `\`, `*`, `?`, `[`, `]` or `#`. HTTP callers must percent-encode a slash that is
part of the `{fileName}` route value.

### HTTP APIs

#### Persist the binary data whether it already exists or not
##### Request
PUT `http://localhost:{daprPort}/v1.0-alpha1/binarystore/{componentName}/{fileName}`

The request body contains arbitrary binary data and is streamed to the component.

##### Response
| Code | Description                                                                 |
| ---- |-----------------------------------------------------------------------------|
| 204 | Binary data successfully persisted                                          |
| 400 | Binary store component not found or file name is invalid                    |
| 413 | File exceeds the maximum size supported or configured by the provider       |
| 507 | Store does not have enough configured capacity for the file                 |
| 500 | Request formatted correctly, but error in Dapr code or underlying component | 


#### Persist the binary data only if it doesn't already exist with the given name
##### Request
POST `http://localhost:{daprPort}/v1.0-alpha1/binarystore/{componentName}/{fileName}`

The request body contains arbitrary binary data and is streamed to the component.

##### Response
| Code | Description                                                                 |
| ---- |-----------------------------------------------------------------------------|
| 204 | Binary data successfully persisted                                          |
| 400 | Binary store component not found or file name is invalid                    |
| 409 | File already exists                                                         |
| 413 | File exceeds the maximum size supported or configured by the provider       |
| 507 | Store does not have enough configured capacity for the file                 |
| 500 | Request formatted correctly, but error in Dapr code or underlying component |


#### Retrieve the binary data
##### Request
GET `http://localhost:{daprPort}/v1.0-alpha1/binarystore/{componentName}/{fileName}`

##### Response
The response to a retrieval request will have its content type header set to `application/octet-stream` as it
returns an array of bytes representing the binary file data.

| Code | Description                                                                 |
| ---- |-----------------------------------------------------------------------------|
| 200 | Binary data successfully retrieved                                          |
| 400 | Binary store component not found or file name is invalid                    |
| 404 | File not found                                                              |
| 500 | Request formatted correctly, but error in Dapr code or underlying component |


#### Delete the binary file
##### Request
DELETE `http://localhost:{daprPort}/v1.0-alpha1/binarystore/{componentName}/{fileName}`

##### Response
| Code | Description |
| ---- | ---- |
| 204 | Indicates the delete operation completed successfully |
| 400 | Binary store component not found or file name is invalid |
| 404 | File not found |
| 500 | Request formatted correctly, but error in Dapr code or underlying component |

## Final Thoughts
This section exists to address some of the concerns shared in previous iterations of this proposal:

### Why now?
This is a new building block intended to be used in conjunction with the existing state store building block.
It's not intended to replace them, but rather to complement them. The existing state management building block is
suitable only storing small amounts of data. Persisting large amounts of data puts large amounts of data in providers 
not intended for this purpose and can lead to performance and resource usage issues on both the sidecar and client.

By providing a new building block, we can provide a longer-term solution for storing large amounts of data while 
still allowing the developer to leverage the existing state store building blocks for managing small amounts of
data more suitable to the key/value stores it's built for.

Further, especially as more SDKs support agentic frameworks built atop Dapr, those tools will need a place to store
data that's larger in scope than a key/value store is necessarily designed to accommodate. Having this performant
alternative to our existing key/value store implementation available for persisting and retrieving that data would
be quite timely.

### How might it be immediately used in the SDKs?
Once we had support for this in the runtime, I'd write a proposed [helper tool](https://github.com/dapr/dotnet-sdk/issues/1533)
for Dapr Workflows in the .NET SDK. When activities yield any data, this utility would persist it to this file store 
and return a reference to it that can be returned to the calling workflow. Future activity invocations can then retrieve 
the data identified by that reference from the file store.

Similarly, such large data could be persisted by other operations and passed into workflows as inputs to be retrieved
by activities in later operations.

This will improve workflow performance by keeping large data out of the event source log in both the 
inputs and outputs of the workflows and their activities.

### Future Ideas
This block will be introduced as an alpha release. We can decide if there are some elements that should be added to it after
the release that warrant inclusion (though ideally not because of a nice-to-have, risking the current state block kitchen 
sink issues, but must-have). Here are a few such proposals:

1) Etags/Conditional writes

### Why doesn't it support [awesome-feature-X]?
This is intended as a provider-agnostic, lean building block that provides a simple mechanism to store binary data
in a streaming fashion that's generally too large for a key/value store.

I leave it to the architects of future even more specialized state store building blocks to decide whether they want to
support additional features beyond what's provided beyond this narrow implementation.

## Completion Checklist
- [x] Binary Store Runtime Implementation
- [x] Tests added (integration, unit and component conformance)
- [x] Implement initial Binary Store providers
- [ ] SDK Support
- [ ] Documentation (supported providers, purpose, set up documentation structure for speciality state stores)
