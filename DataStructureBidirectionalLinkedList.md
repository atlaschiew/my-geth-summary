GETH applies bidirectional linked-list in trie.

```sh  
                                  
                                 +---------+                       +-------------+
           +--db state.Database--|cachingDB|--+--db trie.Database--|trie.Database|--+--diskdb ethdb.KeyValueStore
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

Iterator
```sh
// Cap iteratively flushes old but still referenced trie nodes until the total
// memory usage goes below the given threshold.
//
// Note, this method is a non-synchronized mutator. It is unsafe to call this
// concurrently with other mutators.
func (db *Database) Cap(limit common.StorageSize) error {
	......
  
	oldest := db.oldest
	for size > limit && oldest != (common.Hash{}) {
		// Fetch the oldest referenced node and push into the batch
		node := db.dirties[oldest]
		rawdb.WriteTrieNode(batch, oldest, node.rlp())

		......
		oldest = node.flushNext
	}
	
	......
}
```

Appendation
```sh
// insert inserts a collapsed trie node into the memory database.
// The blob size must be specified to allow proper size tracking.
// All nodes inserted by this function will be reference tracked
// and in theory should only used for **trie nodes** insertion.
func (db *Database) insert(hash common.Hash, size int, node node) {
	......
	
	// Update the flush-list endpoints
	if db.oldest == (common.Hash{}) {
		db.oldest, db.newest = hash, hash
	} else {
		db.dirties[db.newest].flushNext, db.newest = hash, hash
	}
	......
}

```

Deletion
```sh
// dereference is the private locked version of Dereference.
func (db *Database) dereference(child common.Hash, parent common.Hash) {
	......
	if node.parents == 0 {
		// Remove the node from the flush-list
		switch child {
			case db.oldest:
				db.oldest = node.flushNext
				db.dirties[node.flushNext].flushPrev = common.Hash{}
			case db.newest:
				db.newest = node.flushPrev
				db.dirties[node.flushPrev].flushNext = common.Hash{}
			default:
				db.dirties[node.flushPrev].flushNext = node.flushNext
				db.dirties[node.flushNext].flushPrev = node.flushPrev
		}
		......
	}
}
```
