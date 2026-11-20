# What the typed cache layer costs

[`bench.grav`](bench.grav) runs `put`, `get` (a hit) and `remember` (a hit, of
a three-field map) 20,000 times each through a `CacheRepository`, over a
`MemoryCacheStore` and a `FileCacheStore`:

```bash
grav bench.grav --release -o cache_bench --emit bin && ./cache_bench 20000 /dev/shm/cache_bench
```

## Results

Best of 3, milliseconds for 20,000 operations (lower is better), on a 4-core
i5-8350U, the file store on `/dev/shm`.

| | memory store | file store |
|---|---:|---:|
| `put` of a number | 4 (was 7) | 325 (was 358) |
| `get` of a number | 4 (was 9) | 112 (was 124) |
| `remember` of a map | 49 (was 50) | 177 (was 176) |

The memory store's numbers are the layer itself: numbers, booleans, `null` and
strings without escapes are encoded and decoded without building a JSON tree,
which more than halved them. A map still goes through the standard library's
JSON parser and back into a `HashMap`. The file store's numbers are dominated
by its file system calls (a temporary file and a rename per `put`, one read
per `get`).
