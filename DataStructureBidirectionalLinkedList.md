GETH applies linked-list in trie's cached node. Precisely, it is map hash + bidirectional linked-list.

```sh
            +--db state.Database 
+-------+   |
|StateDb|---+
+-------+   |
            +-- trie state.Trie
```

1) `db state.Database` In formal blockchain applies new stateDb with `stateCache state.Database` as db, which `stateCache state.Database` is actually a `cachingDB` instance that created through `state.NewDatabaseWithConfig(...)`
