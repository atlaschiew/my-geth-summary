GETH applies bidirectional linked-list in trie.

```sh  
                                  
                                 +---------+                       +-------------+
           +--db state.Database--|cachingDB|--+--db trie.Database--|trie.Database|
+-------+  |                     +---------+                       +-------------+
|StateDb|--+                                 
+-------+  |                   +----------+  
           +--trie state.Trie--|cachedTrie|--+--db *cachingDB
                               +----------+
```               

1) `db state.Database`. Formal blockchain uses new stateDb with `stateCache state.Database` as db, which `stateCache state.Database` is actually hold a `cachingDB` instance that created through `state.NewDatabaseWithConfig(...)`
2) `trie state.Trie`. is created through cachingDB.OpenTrie(rootHash, cachingDB.db), it is trie with cache feature.
3) both point number 1 and 2 are capable of cache feature.

long story short, let's dig deep into how trie's cachedNode is being created and used?

CachedNode is created after trie is commited. The flow is

1) trie.Commit(...)
2) h := newCommitter(), h.Commit(...)
3) db.insert(...)
4) lastly, db.dirties is filled with cachedNode

Next, this cachedNode must be inserted into flush-list to form a complete linked-list, and flush-list just used by trie.Database.Cap(...) to flush from memory to disk.

Features of flush-list (bidirectional linked-list)

1) `Database.oldest` head of linked-list.
2) `Database.newest` tail of linked-list.
3) `CachedNode` is element of linked-list.
4) `CacheNode.flushNext` & `CacheNode.flushPrev` are fields to form the linked-list.

Iteration of linked-list
```sh
func (db *Database) Cap(limit common.StorageSize) error {
	......
  
	oldest := db.oldest
	for size > limit && oldest != (common.Hash{}) {
		// Fetch the oldest referenced node and push into the batch
		node := db.dirties[oldest]
		rawdb.WriteTrieNode(batch, oldest, node.rlp())

		// If we exceeded the ideal batch size, commit and reset
		if batch.ValueSize() >= ethdb.IdealBatchSize {
			if err := batch.Write(); err != nil {
				log.Error("Failed to write flush list to disk", "err", err)
				return err
			}
			batch.Reset()
		}
		// Iterate to the next flush item, or abort if the size cap was achieved. Size
		// is the total size, including the useful cached data (hash -> blob), the
		// cache item metadata, as well as external children mappings.
		size -= common.StorageSize(common.HashLength + int(node.size) + cachedNodeSize)
		if node.children != nil {
			size -= common.StorageSize(cachedNodeChildrenSize + len(node.children)*(common.HashLength+2))
		}
		oldest = node.flushNext
	}
	// Flush out any remainder data from the last batch
	if err := batch.Write(); err != nil {
		log.Error("Failed to write flush list to disk", "err", err)
		return err
	}
	......
}
```

