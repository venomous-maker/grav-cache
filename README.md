# grav-cache

A cache for [Grav](https://github.com/venomous-maker/grav): `get`/`put`/
`remember` a value under a key, on a file, [Redis](https://github.com/venomous-maker/grav-redis),
or a database table through [Orbit](https://github.com/venomous-maker/grav-orbit) —
one of Laravel's Cache facade, in the manner of its file, Redis and database
drivers.

```grav
import "std"
import "grav-cache"

fn main() {
    Cache.store(new FileCacheStore("/var/cache/app"))

    Cache.put("greeting", "hello", 60)             // expires in 60 seconds
    print(Cache.get("greeting"))                   // "hello"

    let user = Cache.remember("user:1", 300, fn() -> any {
        return loadUserSomehow(1)                  // only runs once per 5 minutes
    })
}
```

## Install

```bash
grav add github.com/venomous-maker/grav-cache
```

```grav
import "grav-cache"
use cache.*
```

`grav-cache` depends on both `grav-redis` and `grav-orbit` (for
`RedisCacheStore` and `OrbitCacheStore`); `FileCacheStore` needs neither. If a
program only ever uses the file store, those two dependencies still resolve
at build time — they just cost nothing at run time until something reaches
for a `RedisCacheStore` or `OrbitCacheStore`.

## Values

A cached value is kept as JSON text: `null`, `bool`, a whole or fractional
number, a string, `DateTime`, `Decimal`, and `Array<any>`/`HashMap<string,
any>` of those — the same plain values Orbit's `@Cast("json")` round-trips.
Anything else (your own class) is a `ValueError`: turn it into a map first
(`u.toMap()` on an Orbit model, say).

## The facade

```grav
Cache.store(new FileCacheStore("/var/cache/app"))          // the default store
Cache.store(new RedisCacheStore(redisConn), "sessions")     // a named one

Cache.put("key", value, 60)        // ttlSeconds; 0 (the default) never expires
Cache.forever("key", value)
Cache.get("key")                   // the value, or null
Cache.get("key", "fallback")
Cache.has("key")
Cache.forget("key")                // removes it; whether there was one
Cache.pull("key")                  // reads it, then forgets it
Cache.remember("key", 60, fn() -> any { return expensive() })      // cached after the first call
Cache.rememberForever("key", fn() -> any { return expensive() })
Cache.increment("hits"); Cache.decrement("hits", 5)
Cache.many(["a", "b"])             // HashMap<string, any>, missing keys left out
Cache.putMany(values, 60)
Cache.flush()

Cache.repository("sessions").get("key")     // a named store, by its own CacheRepository
```

Each of these is also a method on `CacheRepository`, for code that was
handed one explicitly instead of going through the facade:
`new CacheRepository(store).get("key")`.

## Stores

**File** — one file per key, under a directory; no dependency beyond the
standard library.

```grav
Cache.store(new FileCacheStore("/var/cache/app"))
```

Each write goes to a temp file, then an atomic rename, so a reader never
sees a half-written entry. `increment()` is not atomic across processes (a
plain file has no server-side add); fine within one process.

**Redis** — through [grav-redis](https://github.com/venomous-maker/grav-redis).

```grav
Cache.store(new RedisCacheStore(RedisConnection.open("redis://localhost:6379/0")))
Cache.store(new RedisCacheStore(conn, "myapp:cache:"), "sessions")   // a key prefix (default "cache:")
```

Expiry is Redis's own (`SET ... EX`); `increment()` is Redis's `INCRBY`
(atomic on the server); `flush()` deletes only this store's prefix (`SCAN`,
never `FLUSHDB` — the database may hold other things).

**Orbit** — a table, through [grav-orbit](https://github.com/venomous-maker/grav-orbit).

```grav
DB.connect(Sqlite.open("app.db"))        // or MySql/Postgres, as usual for Orbit
CacheEntry.createTable()
Cache.store(new OrbitCacheStore())
Cache.store(new OrbitCacheStore("reports"), "reports_cache")   // a named DB connection
```

`CacheEntry` is an ordinary Orbit model (`cacheKey`, `cacheValue`,
`expiresAt`) — `CacheEntry.createTable()` makes its table, and it migrates
and queries like any other model. `increment()` runs inside a transaction
(read, add, write) so two readers cannot both apply the same addition.

## Writing your own store

Implement `CacheStore` (`get`, `put`, `forget`, `increment`, `flush`;
`has` and `forever` already have sensible defaults) and give it to
`Cache.store(...)` — `CacheRepository` adds the JSON encoding, `remember`,
`many`/`putMany` and the rest on top of any store the same way.

## Tests

```bash
REDIS_URL="redis://127.0.0.1:6379/9" grav test tests/cache.grav
```

Needs a real Redis server (database 9, flushed by the tests — do not point
this at one with data you want) for `RedisCacheStore`'s test; the file and
Orbit (SQLite in memory) stores need nothing extra.

## License

MIT. See [LICENSE](LICENSE).
