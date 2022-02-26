GETH applies linked-list in trie's cached node. Precisely, it is map hash + bidirectional linked-list.

```sh
            +--db state.Database 
+-------+   |
|StateDb|---+
+-------+   |
            +-- trie state.Trie
```

1) `db state.Database` In formal blockchain running, this field uses to hold `stateCache state.Database`, which is actually a `cachingDB` instance that created through `state.NewDatabaseWithConfig(...)`
