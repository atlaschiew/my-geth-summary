GETH applies linked-list in trie's cached node. Precisely, it is map hash + bidirectional linked-list.

```sh  
                                  
                                 +---------+                       +-------------+
           +--db state.Database--|cachingDB|--+--db trie.Database--|trie.Database|
+-------+  |                     +---------+                       +-------------+
|StateDb|--+                                 
+-------+  |                   +----------+  
           +--trie state.Trie--|cachedTrie|--+--db *cachingDB
                               +----------+
```               

1) `db state.Database`. Formal blockchain uses new stateDb with `stateCache state.Database` as db, which `stateCache state.Database` is actually a `cachingDB` instance that created through `state.NewDatabaseWithConfig(...)`
2) `trie state.Trie`. is created through cachingDB.OpenTrie(rootHash, cachingDB.db), it is trie with cache feature.


