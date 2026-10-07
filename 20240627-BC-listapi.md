# List API for the Dapr state component

This proposal proposes implementing a List API in Dapr's state component. The List API will enable the retrieval of keys in a state store based on certain criteria, providing users with the necessary visibility into stored keys. List API results will **not** include the value. SDKs could provide the possibility to bulk get all the keys returned in a page.

The requirements for the API are:

- Ability to list all keys in a state store
- Ability to list keys in a state store with a certain prefix
- The results can be paginated

Not required:
- Ability to sort keys
- Ability to return a paginated result of a snapshot of the state store at a certain point in time

As with the other state store APIs, the List API will also have the difficult job of finding a set of features that are supported across most state store components and filling in the gaps with reasonable behaviour when they aren’t. 

## API

### HTTP

Developers can list keys by issuing an HTTP API call to the Dapr sidecar:

```bash
GET /v1.0/state/:storeName/?prefix={prefix}&page_limit={pageLimit}&page_token={pageToken}
```


The response will be a JSON object with the following structure:
```json
{
    "keys": ["key1", "key2", "key3", "...", "keyN"],
    "next_page_token": "nextTokenString"
}
```

For example:  
Request:
```cURL
GET /v1.0/state/myStateStore?prefix=user&page_limit=3&page_token=user3
```
Response:
```json
{
    "keys": ["user4", "user5", "user6"],
    "next_page_token": "user6"
}
```

### gRPC

Developers can also list keys by issuing a unary gRPC call

```bash
service Dapr {
    ...
	rpc ListState(ListRequest) returns (ListResponse) {}
	...
}

message ListStateRequest {
  // The prefix that should be used for listing.
  optional string prefix = 1;
  
  // The maximum number of items that should be returned per page
  optional uint32 page_limit = 2;

  // Specifies the next pagination token
  // If this is empty, it indicates the start of a new listing.
  optional string page_token = 4;

}

message ListStateResponse {
  // The items that were listed
  repeated string keys = 1;

  // The next pagination token that should be sent for the next page
  // If this is empty, it indicates that there are no more pages.
  optional string next_page_token = 2;
}
```

### Default values

- Prefix: “”
- Page limit: 50
- Next token: “”

## **Pagination**

The two most common pagination strategies are token and offset-based pagination.

**Offset-based pagination** 
Uses a fixed offset and limit to retrieve a subset of results from a larger dataset. This method is common in relational databases and is implemented with the `LIMIT` and `OFFSET` clauses. 

It’s not common in no-sql databases, but it is very common in SQL databases. It relies on a table scan and skipping results until it reaches the offset value.

**Token-based pagination** 

Relies on a token usually equal to, or derived from the last element in the last returned page. 

Very common in no-sql databases that do a scan across the keyspace. 

In relational databases this method relies on an indexed column, such as a timestamp or an ID, to ensure efficient sorting and querying. For example:

```bash
SELECT * FROM items WHERE key > last_key_id ORDER BY key;
```

---

Most often, offset-based pagination is not possible in no-sql databases, while token-based pagination is easy (even preferable) to implement in relational databases, so this proposal suggests using **token-based pagination** in the List API.

Based on this decision, listing items will only be available forwards, and not backwards. To list previous pages, the application would have to keep track of the page tokens.

## SDKs

All supported SDKs should be updated to implement the List API. SDKs should offer the option to fetch batch values of the returned keys.

## Default behaviour for state stores with missing features

Some of the state stores Dapr supports don’t provide the necessary capabilities for implementing the list API. For example, Memcached doesn’t provide a way to list keys,  Azure table storage can’t sort keys in descending order and so on. For those cases the list API will do a best effort to provide the closest functionality to the one defined in the API. The functionality will be specific to the data store and will be implemented on the component level.

List API requests on state stores that don’t support the List API will result in errors.

## Impact of the List API on Dapr state store components
From the moment this proposal is accepted, all state store components will be required to implement the List API in order to get the "Stable" certification level.
Components that are currently stable and for which the underlying state store does not support listing will not lose their stable status.

## Performance and pricing implications
Listing keys in big data sets, specially for partitioned databases, can be expensive in terms of both performance and cost. Often it would incur creating an index which will impact write performance, storage cost and sometimes even read performance.
For the databases where this is a concern, we should offer an option to disable the List API on the component level.

## Definitions:

- **Listing**: The ability to retrieve a collection of items.
- **Sorting**: The ability to sort the results based on one or more fields.
- **Prefix Search**: The ability to search for items that start with a given prefix.
- **Pagination**: The ability to paginate through items, typically using skip/limit or similar mechanisms.

## Addendum

Here's a list of the relevant capabilities of all the stable state stores:

| Store | Cursor listing | Offset listing | Sorting | Number of Items per Page | Prefix Search | Comments |
| --- | --- | --- | --- | --- | --- | --- |
| **aws dynamodb** | Yes | No | Yes, with a GSI | Yes | Yes, with an additional sortKey and a GSI | In order to be able to use prefix search, users will need to have a Global Search Index(GSI) where the partition key will be a single fixed string (for ex. the `_` character) and the sort key will be the key name. There are some drawbacks to this that can be discussed in detail elsewhere. |
| **azure blob store** | Yes | No | Always sorted in ASC order. Desc, or unsorted is not possible. | Yes | Yes | Results are always sorted by key name in ascending order. |
| **azure cosmos db** | Yes | Yes | Yes | Yes | Yes |   |
| **azure table storage** | Yes | No | Yes, just ASC | Yes, with $top | Yes, with range search | Partition key is the application id. |
| **cassandra** | Yes | No | No | Yes | Yes, with `ALLOW FILTERING` but it does a full table scan. | Prefix search with `ALLOW FILTERING` is prohibitively slow. Sorting doesn't happen across all partitions. We could consider maintaining a new index-like non-partitioned table containing all keys, and mirroring the original key’s ttl. |
| **cockroachdb** | Yes, if sorting is required | Yes | Yes | Yes | Yes | Need to create an index on the search column |
| **gcp firestore** | Yes |   |   |   |   |   |
| **in-memory** | No | No | No | No | No | All features can be implemented |
| **memcached** | No | No | No | No | No |   |
| **mongodb** | Yes | Yes | Yes | Yes | Yes |   |
| **mysql** | Yes | Yes | Yes | Yes | Yes | Need to create an index on the id column. MySql supports specialized prefix indices, but you would have to know the exact length of the prefix you’ll be searching on, also sorting will not use the index. |
| **postgresql** | Yes, if sorting is required | Yes | Yes | Yes | Yes | Need to create an index on the key column. We can use the varchar\_pattern\_ops operator class, optimised for prefix search. |
| **redis** | Yes | No | No | Yes (Best effort) | Yes | Number of record per page is not guaranteed, but best effort. |
| **sqlite** | Yes, if sorting is required | Yes | Yes | Yes | Yes | Need to create an index on the key column. It’s a standard b-tree index.We could maintain an index of all keys in a hash |
| **sqlserver** | Yes, if sorting is required | Yes | Yes | Yes | Yes | need to create a non-clustered index on the “key” column |
