# File storage building block
- Author(s): @WhitWaldo
- Significantly inspired by: @ItalyPaleAle (https://github.com/dapr/proposals/pull/18), @artursouza (https://github.com/dapr/dapr/issues/4804)
- Updated August 23, 2025

## Overview
This is a design proposal for building a new specialized state store for managing named files in state providers. I
intentionally use the work file as opposed to blob or object as to avoid any of the overloaded meaning associated with
common object store or blob store providers. I think there's every possibility that it'll make sense to introduce
a proper "blob store" or "object store" building block in the future. However, for now I think it's best to keep the
scope of this proposal focused on files more generally and avoid the complexities of either of those specialty stores.
Over time, the distinction between this and another Blob Store or Object Store blobk should be made clearer by the
enhanced capabilities of either of those APIs (e.g., list by prefix or handling metadata). However, this proposal should
be understood to be a bare-boned implementation designed to strictly store and retrieve (potentially) large serialized
files.

Files are any contiguous sequences of unstructured data comprised of bytes that should be assumed to span lengths as 
few as a handful of bytes to several gigabytes. Examples include images, videos, documents, database backups, 
compressed archives and so on.

Finally, while I also considered calling this the even more ambiguous `DataStore`, I want to be clear that this is 
intended to persist data that's been serialized and not some store for arbitrary in-memory data that's not trivially 
persisted externally. It should be called out in the documentation that this is not intended to be a replacement for 
something like SMB or NFS and is just a general-purpose storage implementation that doesn't get into the Object/Blob 
details.

## Background
Dapr currently implements a state store around a strictly key/value basis. This approach has led to component support
across many different component providers and works for many common use cases. However, decisions around the design
and implementation of the state store feature make it increasingly important to introduce a new generation of 
special-purpose state management building blocks instead of having a one-size-fits-all approach to managing state.

For example, files differ from values in key/value stores in that they can frequently span large amounts of data that
isn't generally compatible with the likes of Redis or Etcd. Because they span such large amounts of data, it's frequently
insufficient to simply store and retrieve them synchronously, mandating a stream-first approach to minimize
resources needed by the runtime to transfer the data. The goal is to allow users to store and retrieve files in a 
performant and scalable manner via a common runtime API and enable the Dapr SDKs to complement the implementation to
support additional provider-specific features on a case by case basis. A possible example of this will be given below.

## Component YAML
This component is expected to have similar attributes to existing state stores with some variation:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: filestore
spec:
  type: state.filestore.<providerName>
  version: v1.0 # Following the API versioning convention proposed in https://github.com/dapr/proposals/pull/87
  metadata:
    # Various properties necessary for component registration
```

## Implementation Overview
As touched on above, there are a few guiding principles I want to carry through from previous iterations of this 
concept:
- All operations should be asynchronously performed with the sidecar
- When persisting or retrieving state, this should be done using an asynchronous stream to maximize SDK and runtime
performance while minimizing resource overhead and preventing memory exhaustion
- While SDKs should allow metadata to be stored alongside the data, if it's not supported by the provider, the 
documentation should clearly state this lack of support (e.g., that the runtime will not otherwise persist it).

### Make Runtime APIs Simpler
I'd like to make the runtime APIs as simple as possible so they're more readily used in a composable way alongside other
Dapr APIs by the SDKs, pluggable components or other developer-written utilities. With that in mind, I do not include
any support for persisting metadata with the files or listing the files in the store in this API. This ensures
broader compatibility with possible stores for file-based data and leaves the possibility of greater sophistication
to the SDKs to handle instead.

This approach allows for a leaner, more provider-agnostic API, takes work off the plate of the runtime developers and
better positions the SDKs to provide a more comprehensive experience leveraging existing building blocks, improving
and simplifying the developer experience.

> **NOTE** It would be imperative to call out in the developer documentation for this building block that the metadata 
> and listing information returned is not provided by the state provider and is instead provided based on values provided 
> by the developer and are not necessarily reflective of the actual state of the store. This could provide a key
> distinction between this and an actual object or blob store for those that need that level of detail (with the tradeoff
> that fewer providers would be supported in the other implementation).


#### Example: C# SDK
For example, in the C# SDK, the developer might specify a key/value store of their choosing to persist file metadata
during the dependency injection registration alongside the name of the file store component:

```csharp
services.AddDapr()
    .WithFileStore(opt => 
    {
        opt.FileStoreName = "my-file-store"; // Defines the file store to use for this injected `DaprFileStore` type
        opt.MetadataStoreName = "my-metadata-store"; // Defines the metadata store to persist metadata to
    });
```

This would provide a simple approach that allows the developer to use the SDK to persist files that optionally include
metadata with a DI-registered type. This would avoid the current approach of having to pass in the store names each time.

```csharp
public sealed class MyClass(DaprFileStore fileStore)
{
    public async Task SaveDataAsync(string name, byte[] data) 
    {
        await fileStore.SetAsync(name, data);
    }
    
    public async Task SaveDataWithMetadataAsync(string name, byte[] data, IReadOnlyDictionary<string, string> metadata )
    {
        await fileStore.SetAsync(name, data, metadata);
    }
}
```

Further, the C# SDK could provide a trivial expansion on this idea to allow named types for saving data to different 
places throughout the application. This example uses values from a shared configuration while still relying on Dapr to 
manage the type lifecycle:

```csharp
services.AddDapr()
    .WithFileStore("fileStoreA", (serviceProvider, options) => 
    {
        // Populate the file store and metadata store names from a configuration
        var configuration = serviceProvider.GetRequiredService<IConfiguration>();
        var fileStoreName = configuration.GetValue<string>("FileStoreName");
        var metadataStoreName = configuration.GetValue<string>("MetadataStoreName");
        
        options.FileStoreName = fileStoreName;
        options.MetadataStoreName = metadataStoreName;        
    })
    .WithFileStore("fileStoreB", options => 
    {
        // Use static values to populate the names in this named `DaprFileStore` client
        options.FileStoreName = "my-file-store";
        options.MetadataStoreName = "my-metadata-store";
    });
```

Because I don't include list support in this API either, the SDK can again be relied on to abstract this away for the 
developer. Paired with the next-generation key/value store I [have proposed](https://github.com/dapr/dapr/issues/7338),
the developer might query the list of files in the store by instead querying the metadata key/value store for any
keys with the file name prefix they're interested in:

```csharp
public sealed class MyClass(DaprFileStore fileStore)
{
    public async Task<IReadOnlyCollection<string>> GetFileNamesAsync(string prefix)
        => await fileStore.ListFilesAsync(prefix);
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
    rpc SetFileAlpha1(stream SetFileRequest) returns (SetFileResponse) {};
    rpc GetFileAlpha1(GetFileRequest) returns (stream GetFileResponse);
    rpc DeleteFileAlpha1(DeleteFileRequest) returns (DeleteFileResponse) {};
}

message SetFileRequest {
    // Request details. Must be present in the first message only.
    SetFileRequestOptions options = 1;
    // Chunk of data of arbitrary size
    common.v1.StreamPayload payload = 2;
}

message SetFileRequestOptions {
    string component_name = 1 [json_name="componentName"];
    string file_name = 2 [json_name="fileName"];
    bool overwrite = 3 [json_name="overwrite"];
}

message SetFileResponse {}

message GetFileRequest {
    string component_name = 1 [json_name="componentName"];
    string file_name = 2 [json_name="fileName"];
}

// The response for GetFileRequest
message GetFileResponse {
    // Chunk of data
    common.v1.StreamPayload payload = 1;
}

message DeleteFileRequest {
    string component_name = 1 [json_name="componentName"];
    string file_name = 2 [json_name="fileName"];
}

message DeleteFileResponse {}
```

### HTTP APIs

#### Persist the file data whether it already exists or not
##### Request
PUT `http://localhost:{daprPort}/v1.0-alpha1/state/filestore/{componentName}/{fileName}`

The request body should contain an array of (nominally UTF-8 encoded) bytes.

##### Response
| Code | Description |
| ---- | ---- |
| 200 | File data successfully persisted |
| 400 | File store provider not found |
| 500 | Request formatted correctly, but error in Dapr code or underlying component | 


#### Persist the file data only if it doesn't already exist with the given name
##### Request
POST `http://localhost:{daprPort}/v1.0-alpha1/state/filestore/{componentName}/{fileName}`

The request body should contain an array of (nominally UTF-8 encoded) bytes.

##### Response
| Code | Description |
| ---- | ---- |
| 200 | File data successfully persisted |
| 400 | File store provider not found |
| 409 | File already exists |
| 500 | Request formatted correctly, but error in Dapr code or underlying component |


#### Retrieve the file data
##### Request
GET `http://localhost:{daprPort}/v1.0-alpha1/state/filestore/{componentName}/{fileName}`

##### Response
The response to a decryption request will have its content type header set to `application/octet-stream` as it
returns an array of bytes representing the file data.

| Code | Description |
| ---- | ---- |
| 200 | File data successfully retrieved |
| 400 | File store provider not found |
| 404 | File not found |
| 500 | Request formatted correctly, but error in Dapr code or underlying component |


#### Delete the file
##### Request
DELETE `http://localhost:{daprPort}/v1.0-alpha1/state/filestore/{componentName}/{fileName}`

##### Response
If the file does not exist, returns:
| Code | Description |
| ---- | ---- |
| 204 | Indicates the delete operation completed successfully |
| 400 | File store provider not found |
| 404 | File not found |
| 500 | Request formatted correctly, but error in Dapr code or underlying component |

## Final Thoughts
This section exists to address some of the concerns shared in previous iterations of this proposal:

### Why now?
This is a new building block intended to be used in conjunction with the existing state store building blocks.
It's not intended to replace them, but rather to complement them. The existing state management building block is
suitable only storing small amounts of data. Persisting large amounts of data puts large amounts of data in providers 
not intended for this purposes and can lead to performance and resource usage issues on both the sidecar and client.

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

This would improve workflow performance by keeping large data out of the event source log in both the inputs and outputs
of the workflows and their activities.

### Why doesn't it support [awesome-feature-X]?
This is intended as a provider-agnostic, lean building block that provides a simple mechanism to store data
in a streaming fashion that's generally too large for a key/value store.

I leave it to the architects of future even more specialized state store building blocks to decide whether they want to
support additional features beyond what's provided beyond this narrow implementation. 


## Completion Checklist
- [ ] File Store Runtime Implementation
- [ ] Tests added (e2e, unit)
- [ ] Implement File Store providers
- [ ] SDK Support
- [ ] Documentation (supported providers, purpose, set up documentation structure for speciality state stores)
