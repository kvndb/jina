# Incremental Indexing in Jina

## Generating Documents Ids:

In Jina, the document ID is by default generated deterministically by computing a hash the document content. This avoids repeating the doc id while reindexing by using a Random or Incremental counter. The user can `override` document ids with Jina client and add `tags` to map to their unique concepts.

## Incremental Indexing Design Pattern:

The rationale behind Incremental indexing is to avoid re-indexing previously indexed documents, thus saving time and compute. This is done by persisting the document identifiers of indexed docs and using them as a DocIdCache.
While indexing a document, it’s identifier is checked against the Cache for a ‘hit’ - if the document id exists,it’s already been indexed. We skip re-indexing it. If it’s a ‘miss’, we index it, adding the identifier to the Cache.
1. DocIdCache stores doc ids in a int64 set and persists it to a numpy array
2. UniqueVectorIndexer is a design pattern for combining VectorIndexer with DocIdCache
3. UniquePbIndexer is a common design pattern for combining KVIndexer with DocIdCache

The difference between a cache and a KVIndexer is the handler_mutex is released in cache. This allows a user to query-while-indexing. This is because we can support querying already indexed doc ids in the cache, since they are not re-indexed on hit. In the meantime, non-indexed doc ids get added to the cache during indexing.

## Uses-before and Uses-after for both single and multiple shards

The Pod context manager now allows --uses-before and --uses-after with single shard --parallel = 1. Previously uses-before and uses-after were only respected when parallel > 1.
With this, users can build a sequential CompoundExecutor ```(before -> exec -> after)``` easily via:
```f = Flow().add(uses_before='before.yml', uses_after='after.yml', uses='exec.yml')```
Comparing the old and new ways:
Old way: CompoundExecutor option without `_uses_before` or `_uses_after` has only one process
New Way: Can spin up 1, 2 or 3 processes at a time

To avoid the differences between single-shard and multi-shard while using DuplicateChecker, it can now be implemented as:
```f = Flow().add(uses=os.path.join(cur_dir, 'docindexer.yml'), uses_before='_unique')```
In this case there is only HeadPea but no TailPea.

