# Search and Vector Building Blocks

Author: Mike Nguyen - @mikeee (hey@mike.ee)

## Overview

This is a proposal surrounding the implementation of two new specialised state stores without bringing into
scope the deep integrations planned on implementation of these building blocks. Whilst the two building 
blocks share almost the same envelope, the method of requesting the records can vary significantly and should
be kept distinct shapes.

## Background

The current implementations of state stores do not accomodate the necessity to access data not only by 
key-value but an extended querying layer. New 'specialised' building blocks facilitate rapid development.

There is a specific need from AI workloads to not only query by key but also by 'relevant' records returned
through lexical/structured matching or geometric proximity for example.

### Scope boundary

This proposal is limited to storing, retrieving, and querying index-ready documents and vectors. It does not
include data-source connectors or application-level data transformation such as extraction, OCR, chunking,
embedding generation, or ingestion-pipeline orchestration. Callers must perform those steps before invoking these
building blocks; provider-internal indexing remains an implementation detail of the component.

[OmniVec](https://github.com/AzureCosmosDB/OmniVec) illustrates why this boundary is important: turning arbitrary
data sources into searchable vectors requires a purpose-built pipeline spanning source integration, content
processing, model execution, and destination management. Absorbing those concerns here would turn the building
blocks into a kitchen-sink search and vector platform rather than a focused, provider-neutral data access API.

### Alpha scope

The alpha surface is deliberately small: dense vectors only, a portable filter subset, and a single
acknowledgement model shared by every write. Features that no in-scope provider can exercise today (sparse and
named vectors, hybrid fusion, vector query pagination) are reserved rather than shipped, so that the alpha wire
contract does not carry surface that is `UNIMPLEMENTED` everywhere. See [Deferred to beta](#deferred-to-beta).

## Providers in scope

| Provider | Lexical Search | Vector Search |
|----------|---|---|
| Meilisearch | ✓ | ✓ |
| Elasticsearch | ✓ | ✓ |
| Chroma | ✓ | ✓ |

Other providers with existing component implementations may be leveraged.

Providers differ in the acknowledgement boundary a write reaches before returning. Meilisearch accepts writes into an
asynchronous [task queue](https://www.meilisearch.com/docs/capabilities/indexing/tasks_and_batches/async_operations)
and can therefore return `INDEX_ACK_QUEUED`; providers such as Elasticsearch that expose only a final write result
return `INDEX_ACK_COMPLETED`. The boundary is reported on every write response through `ack` rather than declared
per provider, so callers never need a capability table; see
[Write failure and acknowledgement semantics](#write-failure-and-acknowledgement-semantics).

### Waiting for Meilisearch task completion

A wait-for-completion write against Meilisearch enqueues the write, then waits for the resulting task to reach a
terminal status. The component supports two mechanisms and selects between them at runtime:

- **Task status polling (default).** The component polls the
  [task status API](https://www.meilisearch.com/docs/reference/api/tasks#get-one-task) with exponential backoff
  until the task is terminal or the wait expires. This works against every supported Meilisearch version with no
  additional configuration and is the guaranteed path.
- **Streamed task changes (experimental optimisation).** When Meilisearch's experimental `tasksStreamingRoute`
  setting is enabled and the component credentials carry `tasks.get`, each Dapr sidecar instance maintains one
  long-lived [task-change stream](https://www.meilisearch.com/docs/reference/api/async-task-management/stream-tasks-changes)
  per initialised Meilisearch component. The stream is filtered to document addition, update and deletion tasks
  in terminal statuses. The component dispatches matching task IDs to waiting requests; requests never open their
  own stream. If Meilisearch reports that the route is disabled, the component falls back to polling for that and
  subsequent waits without failing the request.

The task stream is a notification channel rather than the source of truth. The component reconnects with backoff,
keeps pending waiters across reconnects, and reconciles their task IDs using the task status API after registration,
after a reconnect, and immediately before applying a wait-timeout action. A time-bounded cache of terminal changes
also covers the enqueue-to-registration race. While waiters exist, a stream liveness timeout forces a reconnect and
reconciliation if the connection becomes silent or half-open. Waiters are removed on a terminal change, wait
timeout, or request cancellation.

Neither mechanism is advertised as a component feature; the choice is invisible to callers and both honour the
same `IndexingOptionsAlpha1` semantics.

## HTTP API / Protos

The HTTP API mirrors the protobuf methods below. Path parameters provide `store_name` and, where applicable,
`index` or `collection`; the remaining request fields are supplied in the body.

### HTTP API

#### Search endpoints

| Method | Endpoint | Operation |
|--------|----------|-----------|
| `POST` | `/v1.0-alpha1/search/{storeName}/indexes/{index}` | Create an index |
| `GET` | `/v1.0-alpha1/search/{storeName}/indexes/{index}` | Get an index |
| `GET` | `/v1.0-alpha1/search/{storeName}/indexes` | List indexes |
| `DELETE` | `/v1.0-alpha1/search/{storeName}/indexes/{index}` | Delete an index |
| `POST` | `/v1.0-alpha1/search/{storeName}/indexes/{index}/documents` | Index documents |
| `POST` | `/v1.0-alpha1/search/{storeName}/indexes/{index}/documents/get` | Get documents by ID |
| `POST` | `/v1.0-alpha1/search/{storeName}/indexes/{index}/documents/delete` | Delete documents by ID |
| `POST` | `/v1.0-alpha1/search/{storeName}/indexes/{index}/query` | Search an index |

#### Vector endpoints

| Method | Endpoint | Operation |
|--------|----------|-----------|
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}` | Create a collection |
| `GET` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}` | Get a collection |
| `GET` | `/v1.0-alpha1/vector/{storeName}/collections` | List collections |
| `DELETE` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}` | Delete a collection |
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/upsert` | Upsert vectors |
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/get` | Get vectors |
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/vectors/delete` | Delete vectors by ID |
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/query` | Query vectors |
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/batch-query` | Batch query vectors |

The document and vector get and delete-by-ID endpoints use `POST` because they are batch operations whose repeated
IDs and options are supplied in the request body. A `GET` or `DELETE` request body is not consistently supported by
HTTP clients, proxies, or caches, while encoding large ID lists in a query string imposes practical URL-length
limits. The get operations remain read-only. Index and collection deletion take no body and use `DELETE`.

### Search

```protobuf
rpc CreateIndexAlpha1(CreateIndexRequestAlpha1) returns (google.protobuf.Empty) {}
rpc GetIndexAlpha1(GetIndexRequestAlpha1) returns (GetIndexResponseAlpha1) {}
rpc ListIndexesAlpha1(ListIndexesRequestAlpha1) returns (ListIndexesResponseAlpha1) {}
rpc DeleteIndexAlpha1(DeleteIndexRequestAlpha1) returns (google.protobuf.Empty) {}
rpc IndexDocumentsAlpha1(IndexDocumentsRequestAlpha1) returns (IndexDocumentsResponseAlpha1) {}
rpc GetDocumentsAlpha1(GetDocumentsRequestAlpha1) returns (GetDocumentsResponseAlpha1) {}
rpc DeleteDocumentsAlpha1(DeleteDocumentsRequestAlpha1) returns (DeleteDocumentsResponseAlpha1) {}
rpc SearchAlpha1(SearchRequestAlpha1) returns (SearchResponseAlpha1) {}

message SearchDocument {
  // Caller-supplied identifier. Required for writes and unique within an
  // index.
  string id = 1;
  // UTF-8 encoded JSON object. Field-level operations (search_fields,
  // return_fields, highlight_fields, sort, filter) address top-level and
  // dotted nested keys of this object.
  bytes content = 2;
  // Opaque caller metadata stored with the document and returned unchanged.
  // It is not indexed, not filterable and not addressable by field-level
  // operations.
  map<string, string> metadata = 3;

  // Reserved for a future content_type field. Alpha content is always a JSON
  // object.
  reserved 4;
  reserved "content_type";
}

message SearchHit {
  SearchDocument document = 1;
  // Unnormalized provider-specific relevance score. Higher values indicate a
  // more relevant match.
  double score = 2;
  map<string, string> highlights = 3;
}

enum SortOrder {
  SORT_ORDER_UNSPECIFIED = 0;
  SORT_ORDER_ASC = 1;
  SORT_ORDER_DESC = 2;
}

message SortClause {
  string field = 1;
  SortOrder order = 2;
}

enum IndexAck {
  // Never returned by a successful write call.
  INDEX_ACK_UNSPECIFIED = 0;
  // The provider accepted the request for asynchronous processing. This does
  // not indicate that any item has been applied.
  INDEX_ACK_QUEUED = 1;
  // The provider completed the write. failed_items contains every
  // item-specific failure for this request.
  INDEX_ACK_COMPLETED = 2;
}

enum IndexingMode {
  // Behaves exactly as INDEXING_MODE_RETURN_ON_ACCEPTANCE. Callers that need
  // a final result must select INDEXING_MODE_WAIT_FOR_COMPLETION explicitly,
  // because waiting requires an explicit timeout policy.
  INDEXING_MODE_UNSPECIFIED = 0;
  // Wait for a final provider result.
  INDEXING_MODE_WAIT_FOR_COMPLETION = 1;
  // Return after the provider durably accepts the write for background
  // processing. No eventual result is exposed by this API.
  INDEXING_MODE_RETURN_ON_ACCEPTANCE = 2;
}

enum IndexingWaitTimeoutAction {
  INDEXING_WAIT_TIMEOUT_ACTION_UNSPECIFIED = 0;
  // Return INDEX_ACK_QUEUED and allow a durably queued provider task to
  // continue. Invalid when the provider has no queued acknowledgement.
  INDEXING_WAIT_TIMEOUT_ACTION_CONTINUE_ASYNC = 1;
  // Return DEADLINE_EXCEEDED. This does not guarantee cancellation of
  // provider-side work.
  INDEXING_WAIT_TIMEOUT_ACTION_FAIL_REQUEST = 2;
}

// IndexingOptionsAlpha1 is shared by every write: document indexing and
// deletion, vector upsert and deletion.
message IndexingOptionsAlpha1 {
  IndexingMode mode = 1;
  // Required positive limit for waiting on provider completion. Valid only with
  // INDEXING_MODE_WAIT_FOR_COMPLETION and must be shorter than the remaining
  // RPC context deadline when one is set.
  google.protobuf.Duration wait_timeout = 2;
  // Required with INDEXING_MODE_WAIT_FOR_COMPLETION.
  IndexingWaitTimeoutAction on_wait_timeout = 3;
}

// FailedItem is shared by document indexing and vector upserts.
message FailedItem {
  string id = 1;
  // error.code uses a canonical google.rpc.Code and must not be OK.
  // Provider-specific codes may be included in google.rpc.ErrorInfo details.
  google.rpc.Status error = 2;
}

message CreateIndexRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  // Component-specific index settings.
  map<string, string> metadata = 3;
}

message GetIndexRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  map<string, string> metadata = 3;
}

message GetIndexResponseAlpha1 {
  string index = 1;
  // Approximate document count. Providers that cannot supply this value
  // efficiently may return 0.
  uint64 document_count = 2;
  // Component-specific index properties.
  map<string, string> properties = 3;
}

message ListIndexesRequestAlpha1 {
  string store_name = 1;
  map<string, string> metadata = 2;
}

message ListIndexesResponseAlpha1 {
  repeated string indexes = 1;
}

message DeleteIndexRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  map<string, string> metadata = 3;
}

message IndexDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated SearchDocument documents = 3;
  map<string, string> metadata = 4;
  IndexingOptionsAlpha1 options = 5;
}

message IndexDocumentsResponseAlpha1 {
  // Item-specific failures known at the acknowledgement boundary.
  repeated FailedItem failed_items = 1;
  // Always set to QUEUED or COMPLETED on a successful RPC.
  IndexAck ack = 2;
}

message GetDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated string ids = 3;
  bool include_content = 4;
  map<string, string> metadata = 5;
}

message GetDocumentsResponseAlpha1 {
  // Found documents in request order. IDs that do not exist are omitted.
  repeated SearchDocument documents = 1;
}

message DeleteDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated string ids = 3;
  map<string, string> metadata = 4;
  IndexingOptionsAlpha1 options = 5;
}

message DeleteDocumentsResponseAlpha1 {
  // Always set to QUEUED or COMPLETED on a successful RPC.
  IndexAck ack = 1;
}

message SearchRequestAlpha1 {
  string store_name = 1;
  string index = 2;

  oneof query {
    string text = 3;
    google.protobuf.Struct native = 4;
  }

  google.protobuf.Struct filter = 5;
  // Maximum number of hits to return in this page.
  uint32 top_k = 6;
  // Opaque token returned by the previous SearchResponseAlpha1. Empty for the
  // first page.
  string continuation_token = 7;
  repeated string return_fields = 8;
  bool include_content = 9;
  repeated string search_fields = 10;
  repeated SortClause sort = 11;
  repeated string highlight_fields = 12;

  map<string, string> metadata = 13;
}

enum TotalHitsRelation {
  TOTAL_HITS_RELATION_UNSPECIFIED = 0;
  TOTAL_HITS_RELATION_EXACT = 1;
  TOTAL_HITS_RELATION_LOWER_BOUND = 2;
  TOTAL_HITS_RELATION_ESTIMATE = 3;
}

message SearchResponseAlpha1 {
  repeated SearchHit hits = 1;
  // Best-effort total. Omitted when the provider cannot supply one.
  optional uint64 total_hits = 2;
  // Opaque token for the next page. Empty when there are no more results.
  string continuation_token = 3;
  // Describes the accuracy of total_hits. UNSPECIFIED when total_hits is
  // omitted.
  TotalHitsRelation total_hits_relation = 4;
}
```

#### Document content and metadata

`SearchDocument.content` is a UTF-8 encoded JSON object. This is normative for alpha: every field-level feature of
the search API (`search_fields`, `return_fields`, `highlight_fields`, `sort`, and the filter DSL) addresses keys of
that object using dotted paths, so the runtime and components must be able to parse it. A document whose content is
not a JSON object is reported in `failed_items` with `INVALID_ARGUMENT` and is never sent to the provider; the
remaining documents in the request are processed normally. A `content_type` field number is reserved so that other
encodings can be introduced additively later.

`SearchDocument.metadata` is opaque caller metadata. Components store it with the document and return it unchanged
from `GetDocumentsAlpha1` and `SearchAlpha1`; it is not indexed and cannot be filtered, sorted or projected. Callers
that need filterable attributes place them in `content`.

#### Filter DSL

Filters are a `google.protobuf.Struct` in a JSON-like document shape. Components translate the portable subset to
provider-native queries and reject unsupported operators with `INVALID_ARGUMENT`:

| Class | Operators | Operand types |
|-------|-----------|---------------|
| Comparison | `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte` | string, number, bool |
| Set | `$in`, `$nin` | array of string, number or bool |
| Existence | `$exists` | bool |
| Logical | `$and`, `$or`, `$not` | nested filter expressions |

A bare `"field": value` is shorthand for `"field": {"$eq": value}`. Field paths use dotted notation. For search,
paths address `SearchDocument.content`; for vector, paths address `VectorRecord.metadata`. Operand values are
compared with their JSON type: a number compares numerically, a string lexically, and a mismatch between the operand
type and the stored value is provider-defined and never coerced by the runtime.

```json
{
  "category": { "$in": ["news", "blog"] },
  "price": { "$gte": 10, "$lt": 100 },
  "$or": [
    { "tenant": "acme" },
    { "public": true }
  ]
}
```

#### Index lifecycle semantics

`CreateIndexAlpha1` returns `ALREADY_EXISTS` when the index exists. It does not compare or reconcile settings, so
callers that want "ensure exists" semantics treat `ALREADY_EXISTS` as success. `DeleteIndexAlpha1` and
`GetIndexAlpha1` return `NOT_FOUND` for a missing index. `DeleteDocumentsAlpha1` succeeds for IDs that do not exist
and is therefore safe to retry.

#### Search score semantics

`SearchHit.score` is an unnormalized provider-specific relevance score where higher values indicate a better match.
Components translate any distance-oriented native representation to preserve this higher-is-better contract. Scores
are comparable only among hits from the same query, provider, and index configuration. When an explicit sort is
requested, result order follows that sort and does not necessarily follow score order.

#### Search pagination semantics

The first search request omits `continuation_token`. When more results are available, the response returns an opaque
token that the caller supplies unchanged in the next request. An empty response token means that the provider
definitively reports no further results. Callers must not parse, construct, or modify tokens and must tolerate a
non-empty token leading to a final empty page when provider exhaustion can only be detected by reading the next page.

The component translates the token to the provider's strongest resumable pagination primitive, such as
`search_after`, a point-in-time handle, a provider-managed cursor, or an encoded offset for providers that expose only
offset pagination.

A continuation token is bound to the component, store, index, query, filter, sort, page size, projection, highlighting,
and component-declared result-affecting metadata. Transport metadata such as trace or request IDs is not bound. A
subsequent request repeats those fields unchanged and sets the token from the previous response. A malformed or
mismatched token returns `INVALID_ARGUMENT`. An expired provider cursor returns `FAILED_PRECONDITION` with
`google.rpc.ErrorInfo.reason` set to `SEARCH_CONTINUATION_EXPIRED`; callers restart pagination from the first page.

Pagination requires deterministic ordering. Components append the document ID as a stable tie-breaker when the
requested sort or provider relevance order is not unique. Providers that only sort on declared attributes (for
example Meilisearch's `sortableAttributes`) declare the document ID sortable when the index is created, so that the
tie-breaker is always available. A provider-native query must not embed its own offset, cursor, page size, or
conflicting sort; such a request returns `INVALID_ARGUMENT`.

`total_hits` is best-effort and may be omitted. `total_hits_relation` distinguishes an exact total, a lower bound, and
an estimate. Providers may return it only on the first page, and concurrent writes can change a total reported on
later pages.

#### Write failure and acknowledgement semantics

These semantics apply to every write: `IndexDocumentsAlpha1`, `DeleteDocumentsAlpha1`, `UpsertVectorsAlpha1` and
`DeleteVectorsAlpha1`. All four accept `IndexingOptionsAlpha1` and report an `IndexAck`, so that a caller can
observe when a deletion has been applied on a queued provider in the same way as an insert.

`failed_items` is reserved for failures that the component can attribute to individual input IDs. Authentication,
transport, missing-index, malformed-request, and other request-wide failures are returned as the non-OK status of the
RPC rather than duplicated for every item. Components map native provider errors to canonical `google.rpc.Code`
values. Messages are developer-facing, must not be used as machine-readable values, and must not expose document
content or other sensitive provider details. Deletions report no `failed_items`: a missing ID is not a failure.

`IndexDocumentsAlpha1` and `UpsertVectorsAlpha1` are keyed upserts. Every input must have a non-empty,
caller-supplied ID, and IDs must be unique within a request. The RPC returns `INVALID_ARGUMENT` before invoking the
provider when an ID is empty or duplicated. Retrying the same request is therefore idempotent, including after an
indeterminate timeout.

`INDEXING_MODE_UNSPECIFIED` behaves exactly as `INDEXING_MODE_RETURN_ON_ACCEPTANCE`; the default is explicit and
will not change without a new API version. Both return at the earliest durable acknowledgement boundary offered by
the provider. A provider with a native asynchronous queue returns `INDEX_ACK_QUEUED` after accepting the request for
background processing. The response's `failed_items` contains only item failures known at that boundary, and an
empty list does not indicate that every document will eventually be indexed. The API intentionally provides no
operation ID or later diagnostics; callers receiving `INDEX_ACK_QUEUED` accept that eventual provider failures are
not observable through Dapr. Callers that require consistent final-result semantics across providers must
explicitly select `INDEXING_MODE_WAIT_FOR_COMPLETION`.

If the provider exposes only a final result, or completes the write before returning, the component returns
`INDEX_ACK_COMPLETED` with final `failed_items`. Return-on-acceptance is therefore not guaranteed to be non-blocking
across providers. Components must not create a process-local background queue to manufacture an earlier
acknowledgement.

`INDEXING_MODE_WAIT_FOR_COMPLETION` explicitly waits for provider processing to finish. For Meilisearch, the component
enqueues the write and waits for a terminal task status using polling or, when available, the shared task-change
stream:

- `succeeded` returns `INDEX_ACK_COMPLETED`; the IDs in `failed_items` failed and every requested ID not listed
  succeeded.
- `failed` returns a non-OK RPC status mapped from the task error.
- `canceled` returns a non-OK RPC status with canonical code `ABORTED`; `CANCELLED` remains reserved for cancellation
  of the Dapr RPC itself.

A provider failure that cannot be attributed to individual documents is returned as the non-OK status of the RPC.
Meilisearch task completion is atomic: a succeeded task has no eventual `failed_items`, while a failed task fails the
whole RPC. For Meilisearch, `failed_items` can therefore contain only failures identified before the task is enqueued.

When a provider reports a batch-level failure but cannot establish which items were applied, the non-OK RPC status
includes `google.rpc.ErrorInfo` with reason `INDEXING_OUTCOME_UNKNOWN`. A non-OK response does not by itself guarantee
that no write occurred; callers can safely retry the keyed upsert.

`wait_timeout` and `on_wait_timeout` are required with an explicit `INDEXING_MODE_WAIT_FOR_COMPLETION`. The duration
must be positive. If the RPC context has a deadline, the remaining deadline must be longer than `wait_timeout` so the
selected timeout action can be returned to the caller. Missing fields, an insufficient context deadline, or either
field used with another mode returns `INVALID_ARGUMENT` before records are enqueued. Immediately before the timeout
action, the component performs one task-status reconciliation so a missed notification cannot produce a false
timeout.

If provider work is still not terminal when `wait_timeout` expires:

- `INDEXING_WAIT_TIMEOUT_ACTION_CONTINUE_ASYNC` returns `INDEX_ACK_QUEUED` and the provider task continues. This action
  requires a native durable queued acknowledgement; providers without one return `INVALID_ARGUMENT` before invoking
  the provider.
- `INDEXING_WAIT_TIMEOUT_ACTION_FAIL_REQUEST` returns `DEADLINE_EXCEEDED`. Provider-side work is not guaranteed to be
  canceled, so its final outcome may be unknown to the caller.

Client cancellation or an RPC context ending unexpectedly still takes precedence over the wait option. Components
must remove the request waiter promptly; provider-side work may continue.

The protobuf options message is the portable wire contract. SDKs may expose it using language-idiomatic functional
options; for example, a Go SDK can provide `WithReturnOnIndexAcceptance()` and
`WithWaitForIndexingCompletion(timeout, onTimeout)`.

`UpdateIndexAlpha1` is intentionally omitted. Providers differ in which index settings can be changed after creation,
and some changes require rebuilding the index. A future update operation should define patch semantics, distinguish
mutable and immutable settings, and return an error rather than silently recreating an index or ignoring unsupported
changes.

### Vector

```protobuf
rpc CreateCollectionAlpha1(CreateCollectionRequestAlpha1) returns (google.protobuf.Empty) {}
rpc GetCollectionAlpha1(GetCollectionRequestAlpha1) returns (GetCollectionResponseAlpha1) {}
rpc ListCollectionsAlpha1(ListCollectionsRequestAlpha1) returns (ListCollectionsResponseAlpha1) {}
rpc DeleteCollectionAlpha1(DeleteCollectionRequestAlpha1) returns (google.protobuf.Empty) {}
rpc UpsertVectorsAlpha1(UpsertVectorsRequestAlpha1) returns (UpsertVectorsResponseAlpha1) {}
rpc DeleteVectorsAlpha1(DeleteVectorsRequestAlpha1) returns (DeleteVectorsResponseAlpha1) {}
rpc GetVectorsAlpha1(GetVectorsRequestAlpha1) returns (GetVectorsResponseAlpha1) {}
rpc QueryVectorsAlpha1(QueryVectorsRequestAlpha1) returns (QueryVectorsResponseAlpha1) {}
rpc BatchQueryVectorsAlpha1(BatchQueryVectorsRequestAlpha1) returns (BatchQueryVectorsResponseAlpha1) {}

enum DistanceMetric {
  // On CreateCollection: the component's documented default metric. On
  // QueryVectors: the metric configured for the collection.
  DISTANCE_METRIC_UNSPECIFIED = 0;
  // Cosine similarity. Higher scores indicate closer matches.
  DISTANCE_METRIC_COSINE = 1;
  // Dot-product similarity. Higher scores indicate closer matches.
  DISTANCE_METRIC_DOT_PRODUCT = 2;
  // Euclidean distance. Lower scores indicate closer matches.
  DISTANCE_METRIC_EUCLIDEAN = 3;
}

message CreateCollectionRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  // Component-specific collection settings, such as index parameters.
  map<string, string> metadata = 3;
  // Required. Length of every dense vector stored in the collection.
  uint32 dimensions = 4;
  // Distance metric for the collection. UNSPECIFIED selects the component's
  // documented default. A metric the component cannot provide returns
  // INVALID_ARGUMENT.
  DistanceMetric metric = 5;
}

message GetCollectionRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  map<string, string> metadata = 3;
}

message GetCollectionResponseAlpha1 {
  string collection = 1;
  // Approximate number of records. Providers that cannot supply this value
  // efficiently may return 0.
  uint64 record_count = 2;
  // Component-specific collection properties.
  map<string, string> properties = 3;
  uint32 dimensions = 4;
  // Effective metric. Always a concrete value.
  DistanceMetric metric = 5;
}

message ListCollectionsRequestAlpha1 {
  string store_name = 1;
  map<string, string> metadata = 2;
}

message ListCollectionsResponseAlpha1 {
  repeated string collections = 1;
}

message DeleteCollectionRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  map<string, string> metadata = 3;
}

message VectorRecord {
  string id = 1;
  // Dense vector. Length must equal the collection's dimensions.
  repeated float values = 2;
  // Opaque caller payload stored with the record and returned unchanged. Not
  // filterable.
  bytes payload = 3;
  // Structured, filterable attributes addressed by the filter DSL.
  google.protobuf.Struct metadata = 6;

  // Reserved for sparse and named vectors (deferred to beta).
  reserved 4, 5;
  reserved "sparse_values", "named_vectors";
}

message VectorMatch {
  VectorRecord record = 1;
  // Unnormalized value of the effective metric. Higher is better for COSINE
  // and DOT_PRODUCT; lower is better for EUCLIDEAN.
  double score = 2;
}

message UpsertVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated VectorRecord records = 3;
  map<string, string> metadata = 4;
  IndexingOptionsAlpha1 options = 5;
}

message UpsertVectorsResponseAlpha1 {
  // Item-specific failures known at the acknowledgement boundary.
  repeated FailedItem failed_items = 1;
  // Always set to QUEUED or COMPLETED on a successful RPC.
  IndexAck ack = 2;
}

message DeleteVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated string ids = 3;
  map<string, string> metadata = 4;
  IndexingOptionsAlpha1 options = 5;
}

message DeleteVectorsResponseAlpha1 {
  // Always set to QUEUED or COMPLETED on a successful RPC.
  IndexAck ack = 1;
}

message GetVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated string ids = 3;
  bool include_values = 4;
  map<string, string> metadata = 5;
}

message GetVectorsResponseAlpha1 {
  // Found records in request order. IDs that do not exist are omitted.
  repeated VectorRecord records = 1;
}

message QueryVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;

  oneof query {
    // Only values is read; the remaining VectorRecord fields are ignored.
    VectorRecord vector = 3;
    string by_id = 4;
  }

  uint32 top_k = 5;
  google.protobuf.Struct filter = 6;
  bool include_values = 7;
  bool include_payload = 8;
  // Determines how score and score_threshold are interpreted. UNSPECIFIED
  // uses the metric configured for the collection. A metric the component
  // cannot evaluate for this collection returns INVALID_ARGUMENT.
  DistanceMetric metric = 9;
  // Inclusive cutoff. A match is retained when its score is greater than or
  // equal to this value for COSINE and DOT_PRODUCT, or less than or equal to
  // this value for EUCLIDEAN. The value is not normalized.
  optional double score_threshold = 13;
  map<string, string> metadata = 14;

  // Reserved for hybrid and named-vector queries (deferred to beta).
  reserved 10, 11, 12;
  reserved "sparse_query", "alpha", "vector_name";
}

message QueryVectorsResponseAlpha1 {
  repeated VectorMatch matches = 1;
  // Effective metric used for scores. This is set to a concrete value even
  // when the request uses DISTANCE_METRIC_UNSPECIFIED.
  DistanceMetric metric = 2;
}

message BatchQueryVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated QueryVectorsRequestAlpha1 queries = 3;
  map<string, string> metadata = 4;
}

// BatchQueryResultAlpha1 is the outcome of one query in a batch.
message BatchQueryResultAlpha1 {
  oneof result {
    QueryVectorsResponseAlpha1 response = 1;
    // error.code uses a canonical google.rpc.Code and must not be OK.
    google.rpc.Status error = 2;
  }
}

message BatchQueryVectorsResponseAlpha1 {
  // One result per request query, in request order.
  repeated BatchQueryResultAlpha1 results = 1;
}
```

#### Collection lifecycle semantics

`CreateCollectionAlpha1` requires `dimensions` greater than zero and returns `INVALID_ARGUMENT` otherwise before
invoking the provider. `metric` is a first-class field so that the metric a collection was created with is portable
across providers; `metadata` carries only provider-specific tuning such as index parameters. `CreateCollectionAlpha1`
returns `ALREADY_EXISTS` when the collection exists and does not reconcile settings. `GetCollectionAlpha1` reports the
effective `dimensions` and `metric`. `DeleteVectorsAlpha1` succeeds for IDs that do not exist.

`UpdateCollectionAlpha1` is intentionally omitted. Providers differ in which collection settings can be changed after
creation, and changes to settings such as vector dimensions or distance metric generally require rebuilding the
collection. A future update operation should define patch semantics, distinguish mutable and immutable settings, and
return an error rather than silently recreating a collection or ignoring unsupported changes.

#### Vector score semantics

`VectorMatch.score` and `score_threshold` use the metric's unnormalized value rather than a provider-specific ranking
score or a value normalized onto a common range. Components translate their provider's native score representation to
the following contract:

| Metric | Score | Better match | Inclusive threshold |
|--------|-------|--------------|---------------------|
| Cosine | `dot(a, b) / (norm(a) * norm(b))`, in `[-1, 1]` | Higher | `score >= score_threshold` |
| Dot product | `dot(a, b)`, unbounded | Higher | `score >= score_threshold` |
| Euclidean | `sqrt(sum((a_i - b_i)^2))`, in `[0, +inf)` | Lower | `score <= score_threshold` |

Scores are only comparable when they use the same metric and embedding model. If `metric` is
`DISTANCE_METRIC_UNSPECIFIED`, the collection's configured metric determines these semantics and the response's
`metric` field reports that effective metric.

#### Batch query semantics

`BatchQueryVectorsAlpha1` evaluates each query independenly and returns one `BatchQueryResultAlpha1` per query in
request order. A query that fails validation or is rejected by the provider produces an `error` result with a
canonical code; the remaining queries still run. Only request-wide failures (unknown store, unknown collection,
authentication, transport) fail the RPC as a whole. A query's `store_name` and `collection` are taken from the batch
request; values set on individual queries are ignored.

## Runtime and component contract

The runtime (`dapr/dapr`) owns transport, store resolution and portable validation; components
(`dapr/components-contrib`) own provider translation. The split determines which errors carry Dapr error codes and
which carry provider-mapped canonical codes.

### Validation performed by the runtime

The runtime rejects the following before invoking the component, with `INVALID_ARGUMENT`, HTTP 400, a
`google.rpc.BadRequest` field violation naming the offending field, and the Dapr error code
`ERR_SEARCH_INVALID_REQUEST` or `ERR_VECTOR_INVALID_REQUEST`:

- an empty or duplicated ID in a keyed upsert (`documents.id`, `records.id`);
- an `IndexingOptionsAlpha1` combination that this proposal forbids (`options`, `options.wait_timeout`);
- a vector query that does not set exactly one of `vector` / `by_id` (`query`) — inside a batch this is a per-query
  `error` result rather than an RPC failure;
- a collection created without `dimensions` (`dimensions`).

A document whose `content` is not a JSON object is reported in `failed_items` rather than failing the request, as
described in [Document content and metadata](#document-content-and-metadata).

A store name that does not resolve returns `NOT_FOUND` with `ERR_SEARCH_STORE_NOT_FOUND` /
`ERR_VECTOR_STORE_NOT_FOUND` when other stores of that kind are configured, and `FAILED_PRECONDITION` with
`ERR_SEARCH_STORE_NOT_CONFIGURED` / `ERR_VECTOR_STORE_NOT_CONFIGURED` when none are.

### Component error contract

Components return errors created with `google.golang.org/grpc/status` carrying a canonical `codes.Code`. The runtime
passes a status error through unchanged, including any `google.rpc.ErrorInfo` details such as the
`SEARCH_CONTINUATION_EXPIRED` and `INDEXING_OUTCOME_UNKNOWN` reasons (domain `dapr.io`). An error that is not a status
error is reported as `INTERNAL`. The runtime never re-wraps a component error with a Dapr error code, so that the
canonical code and details the caller relies on survive.

The `search` package in components-contrib provides the shared validators (`ValidateWriteIDs`,
`ValidateIndexingOptions`) and error helpers (`ContinuationExpiredError`, `IndexingOutcomeUnknownError`) so that
components and the runtime enforce identical rules.

### Component interfaces

```go
// github.com/dapr/components-contrib/search
type Search interface {
    metadata.ComponentWithMetadata
    Init(ctx context.Context, meta Metadata) error
    CreateIndex(ctx context.Context, req *CreateIndexRequest) error
    GetIndex(ctx context.Context, req *GetIndexRequest) (*GetIndexResponse, error)
    ListIndexes(ctx context.Context, req *ListIndexesRequest) (*ListIndexesResponse, error)
    DeleteIndex(ctx context.Context, req *DeleteIndexRequest) error
    IndexDocuments(ctx context.Context, req *IndexDocumentsRequest) (*IndexDocumentsResponse, error)
    GetDocuments(ctx context.Context, req *GetDocumentsRequest) (*GetDocumentsResponse, error)
    DeleteDocuments(ctx context.Context, req *DeleteDocumentsRequest) (*DeleteDocumentsResponse, error)
    Search(ctx context.Context, req *SearchRequest) (*SearchResponse, error)
    io.Closer
}

// github.com/dapr/components-contrib/vector
type Vector interface {
    metadata.ComponentWithMetadata
    Init(ctx context.Context, meta Metadata) error
    CreateCollection(ctx context.Context, req *CreateCollectionRequest) error
    GetCollection(ctx context.Context, req *GetCollectionRequest) (*GetCollectionResponse, error)
    ListCollections(ctx context.Context, req *ListCollectionsRequest) (*ListCollectionsResponse, error)
    DeleteCollection(ctx context.Context, req *DeleteCollectionRequest) error
    Upsert(ctx context.Context, req *UpsertRequest) (*UpsertResponse, error)
    Get(ctx context.Context, req *GetRequest) (*GetResponse, error)
    Delete(ctx context.Context, req *DeleteRequest) (*DeleteResponse, error)
    Query(ctx context.Context, req *QueryRequest) (*QueryResponse, error)
    BatchQuery(ctx context.Context, req *BatchQueryRequest) (*BatchQueryResponse, error)
    io.Closer
}
```

Request and response types mirror the protobuf messages using Go-native types (`[]byte` content,
`map[string]any` metadata and filters, `time.Duration` timeouts) so that no protobuf types leak into components.
Conformance tests in `tests/conformance/search` and `tests/conformance/vector` are the executable form of the
semantics in this document and are the contract a new component is written against.

### Tenancy

The `(store, index)` and `(store, collection)` tuples are the scoping mechanism. Per-tenant isolation within an
index or collection is the caller's responsibility: place a tenant identifier in `content` (search) or `metadata`
(vector) and apply it as a filter on every query. Components must not silently widen a filtered query.

## Deferred to beta

The following are explicitly out of the alpha contract. Each is additive to the wire format above, and the relevant
field numbers are reserved where a shape has already been sketched.

- **Sparse and named vectors, hybrid fusion (`sparse_values`, `named_vectors`, `sparse_query`, `alpha`,
  `vector_name`).** No in-scope provider can exercise them today, and `alpha` needs a defined fusion contract
  before it can be portable.
- **Vector query pagination.** Deep pagination over approximate-nearest-neighbour results is not uniformly
  supported; a continuation token will be added once a provider with a resumable primitive is in scope.
- **Capability discovery.** Whether a store offers a queued acknowledgement or a given metric is currently learned
  by failing; a metadata-endpoint capability listing will follow.
- **`UpdateIndexAlpha1` / `UpdateCollectionAlpha1`.** See the rationale in each section.
- **Request limits.** Maximum batch size, `top_k` ceiling and payload size are component-enforced in alpha and
  will be surfaced as configurable runtime limits.
- **Observability.** Span attributes and metrics for search and vector calls follow the existing building-block
  conventions and will be defined with the beta API.
- **Continuation-token lifetime and `total_hits` opt-out.**

## Alpha exit criteria

- Two providers implement each building block and pass the conformance suite.
- The Go SDK exposes both building blocks; at least one further SDK follows the same proto without wire changes.
- No `reserved` field is reintroduced with different semantics from those sketched here.

## Consumption Examples (Go)

```go
import (
    "context"
    "time"

    "google.golang.org/protobuf/proto"
    "google.golang.org/protobuf/types/known/durationpb"
    "google.golang.org/protobuf/types/known/structpb"

    pb "github.com/dapr/dapr/pkg/proto/runtime/v1"
)

func examples(ctx context.Context, client pb.DaprClient, document, anotherDocument, payload []byte, dense, q1, q2 []float32) error {
    // Wait up to five seconds for a terminal task status. If the wait expires,
    // return QUEUED and allow the provider task to continue.
    indexing, err := client.IndexDocumentsAlpha1(ctx, &pb.IndexDocumentsRequestAlpha1{
        StoreName: "meili-products",
        Index:     "products",
        Documents: []*pb.SearchDocument{
            {Id: "headphones-123", Content: document},
        },
        Options: &pb.IndexingOptionsAlpha1{
            Mode:          pb.IndexingMode_INDEXING_MODE_WAIT_FOR_COMPLETION,
            WaitTimeout:   durationpb.New(5 * time.Second),
            OnWaitTimeout: pb.IndexingWaitTimeoutAction_INDEXING_WAIT_TIMEOUT_ACTION_CONTINUE_ASYNC,
        },
    })
    if err != nil {
        return err
    }
    _ = indexing.GetAck() // INDEX_ACK_COMPLETED or, after the wait expired, INDEX_ACK_QUEUED

    // Fire-and-forget indexing. SDKs may expose this request option as a
    // language-idiomatic functional option such as WithReturnOnIndexAcceptance().
    if _, err = client.IndexDocumentsAlpha1(ctx, &pb.IndexDocumentsRequestAlpha1{
        StoreName: "meili-products",
        Index:     "products",
        Documents: []*pb.SearchDocument{
            {Id: "product-123", Content: anotherDocument},
        },
        Options: &pb.IndexingOptionsAlpha1{
            Mode: pb.IndexingMode_INDEXING_MODE_RETURN_ON_ACCEPTANCE,
        },
    }); err != nil {
        return err
    }

    // Search with a portable filter.
    filter, _ := structpb.NewStruct(map[string]any{
        "category": map[string]any{"$in": []any{"audio"}},
        "price":    map[string]any{"$lt": 200},
    })
    firstPage, err := client.SearchAlpha1(ctx, &pb.SearchRequestAlpha1{
        StoreName:    "meili-products",
        Index:        "products",
        Query:        &pb.SearchRequestAlpha1_Text{Text: "wireless headphones"},
        Filter:       filter,
        SearchFields: []string{"title", "description"},
        TopK:         10,
    })
    if err != nil {
        return err
    }

    // Request the next page by repeating the same search and passing the opaque
    // token unchanged.
    if firstPage.GetContinuationToken() != "" {
        if _, err = client.SearchAlpha1(ctx, &pb.SearchRequestAlpha1{
            StoreName:         "meili-products",
            Index:             "products",
            Query:             &pb.SearchRequestAlpha1_Text{Text: "wireless headphones"},
            Filter:            filter,
            SearchFields:      []string{"title", "description"},
            TopK:              10,
            ContinuationToken: firstPage.GetContinuationToken(),
        }); err != nil {
            return err
        }
    }

    // Collections declare dimensions and metric as first-class fields.
    if _, err = client.CreateCollectionAlpha1(ctx, &pb.CreateCollectionRequestAlpha1{
        StoreName:  "meili-docs",
        Collection: "manuals",
        Dimensions: uint32(len(dense)),
        Metric:     pb.DistanceMetric_DISTANCE_METRIC_COSINE,
    }); err != nil {
        return err
    }

    // Vector upserts use the same acknowledgement and wait options. Structured
    // metadata is filterable; payload is opaque.
    metadata, _ := structpb.NewStruct(map[string]any{"tenant": "acme", "revision": 3})
    if _, err = client.UpsertVectorsAlpha1(ctx, &pb.UpsertVectorsRequestAlpha1{
        StoreName:  "meili-docs",
        Collection: "manuals",
        Records: []*pb.VectorRecord{
            {Id: "manual-123", Values: dense, Payload: payload, Metadata: metadata},
        },
        Options: &pb.IndexingOptionsAlpha1{
            Mode:          pb.IndexingMode_INDEXING_MODE_WAIT_FOR_COMPLETION,
            WaitTimeout:   durationpb.New(5 * time.Second),
            OnWaitTimeout: pb.IndexingWaitTimeoutAction_INDEXING_WAIT_TIMEOUT_ACTION_FAIL_REQUEST,
        },
    }); err != nil {
        return err
    }

    // Dense vector query with an inclusive cosine threshold.
    tenantFilter, _ := structpb.NewStruct(map[string]any{"tenant": "acme"})
    matches, err := client.QueryVectorsAlpha1(ctx, &pb.QueryVectorsRequestAlpha1{
        StoreName:      "meili-docs",
        Collection:     "manuals",
        Query:          &pb.QueryVectorsRequestAlpha1_Vector{Vector: &pb.VectorRecord{Values: dense}},
        Filter:         tenantFilter,
        Metric:         pb.DistanceMetric_DISTANCE_METRIC_COSINE,
        TopK:           5,
        IncludePayload: true,
        ScoreThreshold: proto.Float64(0.6),
    })
    if err != nil {
        return err
    }
    _ = matches.GetMetric() // the effective metric, always concrete

    // Batch query: one result per query, each succeeding or failing on its own.
    batch, err := client.BatchQueryVectorsAlpha1(ctx, &pb.BatchQueryVectorsRequestAlpha1{
        StoreName:  "meili-docs",
        Collection: "manuals",
        Queries: []*pb.QueryVectorsRequestAlpha1{
            {Query: &pb.QueryVectorsRequestAlpha1_Vector{Vector: &pb.VectorRecord{Values: q1}}, TopK: 5},
            {Query: &pb.QueryVectorsRequestAlpha1_Vector{Vector: &pb.VectorRecord{Values: q2}}, TopK: 5},
        },
    })
    if err != nil {
        return err
    }
    for _, result := range batch.GetResults() {
        if st := result.GetError(); st != nil {
            continue // this query failed; st.Code is canonical
        }
        _ = result.GetResponse().GetMatches()
    }

    // Deletions share the acknowledgement model, so a caller can wait for a
    // queued provider to apply them.
    deleted, err := client.DeleteVectorsAlpha1(ctx, &pb.DeleteVectorsRequestAlpha1{
        StoreName:  "meili-docs",
        Collection: "manuals",
        Ids:        []string{"manual-123"},
        Options: &pb.IndexingOptionsAlpha1{
            Mode:          pb.IndexingMode_INDEXING_MODE_WAIT_FOR_COMPLETION,
            WaitTimeout:   durationpb.New(5 * time.Second),
            OnWaitTimeout: pb.IndexingWaitTimeoutAction_INDEXING_WAIT_TIMEOUT_ACTION_FAIL_REQUEST,
        },
    })
    if err != nil {
        return err
    }
    _ = deleted.GetAck()
    return nil
}
```
