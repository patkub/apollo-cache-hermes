# GraphQL Client Cache Performance

The performance of existing (normalizing) GraphQL caches [isn't great](http://convoy-scrubbed-graphql-client-benchmarks.s3-website-us-west-2.amazonaws.com), particularly for queries that select a large number of objects.

This document explores the reasons for the poor performance of existing implementations.  It also provides background for the [design decisions](./ARCHITECTURE.md) made by this new cache implementation.


## Terminology

**Scalar**: A [primitive value](http://facebook.github.io/graphql/#sec-Scalars), such as a number, string, boolean, etc.

**Node**: A [GraphQL object](http://facebook.github.io/graphql/#sec-Objects). May contain scalar values, or references to other nodes. _The term "node" ends up being clearer than "object" when used throughout this document._

**Entity**: A node that has special meaning in a particular schema; a business object.  May be composed of many nested nodes.  E.g. User, Comment, Post, etc.

**Edge**: A name used to reference the value of a node.

**Selection Set**: [An expression of edges](http://facebook.github.io/graphql/#sec-Selection-Sets) - often nested - describing the paths to all values that should be retrieved by a query.

**Observer**: A construct that watches the cache for a specific query, and emits callbacks whenever the values selected by that query have changed.


## Status Quo

The existing caches (Apollo, Relay, Cashay?) exhibit poor performance for _both_ reads and writes.  This behavior comes from a few key behaviors in their design:


### Flattening (& Normalization)

When processing a GraphQL response, _every node_ is transformed into a flat object that contains its scalar values, while references to child nodes are converted into explicit pointers.  These nodes are then inserted into (merged with) an identity map that maintains a normalized view of all nodes.

For example:

```json
{
  "id": 1,
  "title": "GraphQL Rocks!",
  "author": {
    "id": 2,
    "name": "Gouda"
  }
}
```

Would be flattened into an identity map that looks roughly like:

```json
{
  "1": {
    "id": 1,
    "title": "GraphQL Rocks!",
    "author": {
      "__ref": 2
    }
  },
  "2": {
    "id": 2,
    "name": "Gouda"
  }
}
```

Similarly, when performing queries against the cache, the process must be reversed.


### Exact Results

One of GraphQL's key selling points is the ability to select only the values that your application needs for a particular query.  This permits an application to select different values for the same node, across multiple queries - and it's great!

The existing caches provide two guarantees that are impacted by this behavior:

**All nodes are normalized**:  E.g. if you select `{ one two }` in a query, and `{ two three }` in another, you are guaranteed that the value of `two` will be the same for both queries when fetched from the cache (even if they initially disagreed).

**Queries yield exact results**: A caller is guaranteed that the results of a query will contain _at most_ the values expressed by the query, even if there are additional values in the cache.  This helps mitigate hard-to-catch bugs where a caller may reference a property of a node that they forgot to include in the query.


## The Cost

When combined, these design decisions end up incurring some significant costs:

1. Walking the response, converting each node to its flattened form, and merging that with the identity map consumes a fair bit of CPU time.

2. Executing a query against the cache requires walking its selection set, and constructing a hierarchy of matching values from the identity map also consumes a fair bit of CPU time.

3. Due to queries returning exact results, the CPU cost of (2) is magnified for each active observer.

4. Each observer also incurs increased memory overhead due to the duplication of result objects.

5. Due to the myriad of JavaScript objects being allocated and thrown away throughout this process, the JavaScript garbage collector will frequently run (particularly on mobile clients).