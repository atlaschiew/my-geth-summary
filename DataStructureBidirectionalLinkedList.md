GETH applies linked-list in trie's cachedNode. Precisely, it is map hash + bidirectional linked-list.

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

Cached node is created after trie is commited. The flow is

1) trie.Commit(...)
2) h := newCommitter(), h.Commit(...)
3) db.insert(...)
4) lastly, db.dirties is filled with cachedNode

next this cachedNode must be inserted into flush-list to form a complete linked-list, and flush-list just used by trie.Database.Cap(...) to flush from memory to disk.

