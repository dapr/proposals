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