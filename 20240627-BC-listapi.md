# List API for the Dapr state component

This proposal proposes implementing a List API in Dapr's state component. The List API will enable the retrieval of keys in a state store based on certain criteria, providing users with the necessary visibility into stored keys. List API results will **not** include the value. SDKs could provide the possibility to bulk get all the keys returned in a page.

The requirements for the API are:

- Ability to list all keys in a state store
- Ability to list keys in a state store with a certain prefix
- The results can be sorted
- The results can be paginated

As with the other state store APIs, the List API will also have the difficult job of finding a set of features that are supported across most state store components and filling in the gaps with reasonable behaviour when they aren’t. 

## API

### HTTP

Developers can list keys by issuing an HTTP API call to the Dapr sidecar:

```bash
GET /v1.0/state/:storeName/?prefix={prefix}&sorting={sorting}&page_limit={pageLimit}&page_token={pageToken}
```

The `sorting` query parameter can accept one of the following values:
- `default`
- `asc`
- `desc`


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
GET /v1.0/state/myStateStore?prefix=user&sorting=asc&page_limit=3&page_token=user3
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

  // Specifies if the result should be sorted
  optional Sort sort = 3;

  // Specifies the next pagination token
  // If this is empty, it indicates the start of a new listing.
  optional string page_token = 4;

  // Sorting order options
  enum Sort {
    DEFAULT = 0;
    ASCENDING = 1;
    DESCENDING = 2;
  }
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
- Sorting: “default”
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

Most often, offset-based pagination is not possible in no-sql databases, while it’s easy (even preferable) to implement in relational databases, so this proposal suggests using **token-based pagination** in the List API.

Based on this decision, listing items will only be available forwards, and not backwards. To list previous pages, the application would have to keep track of the page tokens.

## **Sorting**

Sorting is required for token-based pagination in relational databases, so we must have a default sorting order.

Some no-sql databases (ex. Azure blob store) don’t support sorting in descending order and others don’t support any sorting at all (ex. Redis). In these cases, we want to return an explicit error instead of failing silently.

This might be restricting for use cases where the underlying state store needs to be swapped though. For example a team could use Redis for local development, and Postgres in production, and they wouldn’t be able to use the same application code, because the sorting clause would error on Redis, but pass on Postgres. That’s why we’re introducing the `Default` sorting option which will sort in ascending order for all databases that support it, and leave results unsorted for the databases that don’t.

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

## Adendum

Here's a list of the relevant capabilities of all the stable state stores:

|   | Store | Cursor listing | Offset listing | Sorting | Number of Items per Page | Prefix Search | Comments |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | aws dynamodb | Yes | No | Yes, with a GSI | Yes | Yes, with an additional sortKey and a GSI | In order to be able to use prefix search, users will need to have a Global Search Index(GSI) where the partition key will be a single fixed string (for ex. the `_` character) and the sort key will be the key name. There are some drawbacks to this that can be discussed in detail elsewhere. |
| 2 | azure blob store | Yes (continuation token) | No | Always sorted in ASC order. Desc, or unsorted is not possible. | Yes | Yes | Results are always sorted by key name in ascending order. |
| 3 | azure cosmos db | Yes | Yes | Yes | Yes | Yes |   |
| 4 | azure table storage | Yes | No | Yes, just ASC | Yes, with $top | Yes, with range search | Partition key is the application id. |
| 5 | cassandraYes | No | No | No | No | Can’t prefix search and sort across all partitions. We could consider maintaining a new table containing all keys, and mirroring the original key’s ttl. |   |
| 6 | cockroachdbYes, if sorting is required | Yes | Yes | No | Yes | Need to create an index on the search column |   |
| 7 | gcp firestore | Yes |   |   |   |   |   |
| 8 | in-memory | No | No | No | No | No | We can implement all the features, but it’s not trivial to aggregate data across multiple instances |
| 9 | memcached | No | No | No | No | No |   |
| 10 | mongodbYes | Yes | Yes |   | Yes |   |   |
| 11 | mysqlYes | Yes |   |   | Yes | Need to create an index on the id column. MySql supports specialized prefix indices, but you would have to know the exact length of the prefix you’ll be searching on, also sorting will not use the index. |   |
| 12 | postgresqlYes | Yes | Yes | No | Yes | Need to create an index on the key column. We can use the varchar\_pattern\_ops operator class, optimised for prefix search. |   |
| 13 | redisYes | No | No | No | Yes | Number of record per page is not guaranteed, but best effort. |   |
| We could maintain our own sortedset that keeps the ttl. |   |   |   |   |   |   |   |
| 14 | sqliteYes, if sorting is required | Yes | Yes | No | Yes | Need to create an index on the key column. It’s a standard b-tree index.We could maintain an index of all keys in a hash |   |
| 15 | sqlserver | Yes, if sorting is required | Yes | Yes | Yes | Yes | need to create a non-clustered index on the “key” column |
