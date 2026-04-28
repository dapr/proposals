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

## Providers in scope

| Provider | Lexical Search | Vector Search |
|----------|---|---|
| Meilisearch | ✓ | ✓ |
| Elasticsearch | ✓ | ✓ |
| Chroma | ✓ | ✓ |

Other providers with existing component implementations may be leveraged.

## HTTP API / Protos

A HTTP api will be exposed mirroring the below.

### Search

```protobuf
rpc IndexDocumentsAlpha1(IndexDocumentsRequestAlpha1) returns (IndexDocumentsResponseAlpha1) {}
rpc DeleteDocumentsAlpha1(DeleteDocumentsRequestAlpha1) returns (google.protobuf.Empty) {}
rpc SearchAlpha1(SearchRequestAlpha1) returns (SearchResponseAlpha1) {}

message SearchDocument {
  string id = 1;
  bytes content = 2;
  map<string, string> metadata = 3;
}

message SearchHit {
  SearchDocument document = 1;
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
  INDEX_ACK_UNSPECIFIED = 0;
  INDEX_ACK_QUEUED = 1;
  INDEX_ACK_DURABLE = 2;
}

message IndexDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated SearchDocument documents = 3;
  map<string, string> metadata = 10;
}

message IndexDocumentsResponseAlpha1 {
  repeated string failed_ids = 1;
  IndexAck ack = 2;
}

message DeleteDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated string ids = 3;
  map<string, string> metadata = 10;
}

message SearchRequestAlpha1 {
  string store_name = 1;
  string index = 2;

  oneof query {
    string text = 3;
    google.protobuf.Struct native = 4;
  }

  google.protobuf.Struct filter = 5;
  uint32 top_k = 6;
  uint32 offset = 7;
  repeated string return_fields = 8;
  bool include_content = 9;
  map<string, string> metadata = 10;
  repeated string search_fields = 11;
  repeated SortClause sort = 12;
  repeated string highlight_fields = 13;

  reserved 14, 15, 16;
}

message SearchResponseAlpha1 {
  repeated SearchHit hits = 1;
  uint64 total_hits = 2;
  string continuation_token = 3;
}
```

### Vector

```protobuf
rpc UpsertVectorsAlpha1(UpsertVectorsRequestAlpha1) returns (UpsertVectorsResponseAlpha1) {}
rpc DeleteVectorsAlpha1(DeleteVectorsRequestAlpha1) returns (google.protobuf.Empty) {}
rpc GetVectorsAlpha1(GetVectorsRequestAlpha1) returns (GetVectorsResponseAlpha1) {}
rpc QueryVectorsAlpha1(QueryVectorsRequestAlpha1) returns (QueryVectorsResponseAlpha1) {}
rpc BatchQueryVectorsAlpha1(BatchQueryVectorsRequestAlpha1) returns (BatchQueryVectorsResponseAlpha1) {}

enum DistanceMetric {
  DISTANCE_METRIC_UNSPECIFIED = 0;
  DISTANCE_METRIC_COSINE = 1;
  DISTANCE_METRIC_DOT_PRODUCT = 2;
  DISTANCE_METRIC_EUCLIDEAN = 3;
}

message SparseVector {
  repeated uint32 indices = 1;
  repeated float values = 2;
}

message NamedVector {
  repeated float values = 1;
  SparseVector sparse_values = 2;
}

message VectorRecord {
  string id = 1;
  repeated float values = 2;
  bytes payload = 3;
  map<string, string> metadata = 4;
  SparseVector sparse_values = 5;
  map<string, NamedVector> named_vectors = 6;
}

message VectorMatch {
  VectorRecord record = 1;
  double score = 2;
}

message UpsertVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated VectorRecord records = 3;
  map<string, string> metadata = 10;
}

message UpsertVectorsResponseAlpha1 {
  repeated string failed_ids = 1;
}

message DeleteVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated string ids = 3;
  map<string, string> metadata = 10;
}

message GetVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated string ids = 3;
  bool include_values = 4;
  map<string, string> metadata = 10;
}

message GetVectorsResponseAlpha1 {
  repeated VectorRecord records = 1;
}

message QueryVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;

  oneof query {
    VectorRecord vector = 3;
    string by_id = 4;
  }

  uint32 top_k = 5;
  google.protobuf.Struct filter = 6;
  bool include_values = 7;
  bool include_payload = 8;
  DistanceMetric metric = 9;
  map<string, string> metadata = 10;
  SparseVector sparse_query = 11;
  optional float alpha = 12;
  optional string vector_name = 13;
  optional float score_threshold = 14;
}

message QueryVectorsResponseAlpha1 {
  repeated VectorMatch matches = 1;
}

message BatchQueryVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated QueryVectorsRequestAlpha1 queries = 3;
  map<string, string> metadata = 10;
}

message BatchQueryVectorsResponseAlpha1 {
  repeated QueryVectorsResponseAlpha1 results = 1;
}
```

## Consumption Examples (Go)

```go
// Search
hits, err := client.SearchAlpha1(ctx, &client.SearchRequestAlpha1{
    StoreName: "meili-products",
    Index:         "products",
    Query:         &client.SearchRequestAlpha1Text{Text: "wireless headphones"},
    SearchFields:  []string{"title", "description"},
    TopK:          10,
})

// Vector query
matches, err := client.QueryVectorsAlpha1(ctx, &client.QueryVectorsRequestAlpha1{
    StoreName: "meili-docs",
    Collection:    "manuals",
    Query: &client.QueryVectorsRequestAlpha1_Vector{
        Vector: &client.VectorRecord{Values: dense},
    },
    SparseQuery:    &client.SparseVector{Indices: sparseIdx, Values: sparseVal},
    Alpha:          proto.Float32(0.7),
    TopK:           5,
    IncludePayload: true,
    ScoreThreshold: proto.Float32(0.6),
})

// Batch query
batch, err := client.BatchQueryVectorsAlpha1(ctx, &client.BatchQueryVectorsRequestAlpha1{
    StoreName: "meili-docs",
    Collection:    "manuals",
    Queries: []*client.QueryVectorsRequestAlpha1{
        {Query: &client.QueryVectorsRequestAlpha1_Vector{Vector: &client.VectorRecord{Values: q1}}, TopK: 5},
        {Query: &client.QueryVectorsRequestAlpha1_Vector{Vector: &client.VectorRecord{Values: q2}}, TopK: 5},
    },
})
```