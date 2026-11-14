# grav-cache

A cache for [Grav](https://github.com/venomous-maker/grav): `get`/`put`/
`remember` a value under a key, in memory, on a file, [Redis](https://github.com/venomous-maker/grav-redis),
or a database table through [Orbit](https://github.com/venomous-maker/grav-orbit) —
one of Laravel's Cache facade, in the manner of its array, file, Redis and
database drivers — with atomic locks and a rate limiter on top of any store.

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

## Locks

`Cache.lock(name, seconds)` is a lock that processes and threads share through
the store: one owner at a time, lapsing after `seconds` (so a crashed holder
cannot keep it), released only by its owner.

```grav
let lock = Cache.lock("reports:daily", 120)
if (lock.get()) { try { build() } finally { lock.release() } }
lock.run(fn() { build() })                   // takes it, runs, lets go (false when taken)
lock.blockRun(5, fn() { build() })           // waits up to 5 s (LockTimeoutException after)
Cache.restoreLock("reports:daily", ownerToken).release()   // from the process handed the token
```

Taking it is `add` (store only when missing) and releasing is
`forgetIfValue` (delete only while it holds this owner's token); both are
atomic on Redis (`SET NX`, and a compare-and-delete script) and in memory.
The file and Orbit stores check and then write, which suits one process.

## Rate limiting

`Cache.limiter()` counts attempts in a window, as Laravel's `RateLimiter`:

```grav
let limiter = Cache.limiter()
if (limiter.tooManyAttempts("login:" + ip, 5)) { return tooMany(limiter.availableIn("login:" + ip)) }
limiter.hit("login:" + ip, 60)                       // a 60-second window from the first hit
limiter.attempt("sms:" + userId, 3, fn() { send() }, 3600)   // false once the limit is reached
limiter.remaining(key, 5)  limiter.attempts(key)  limiter.clear(key)
```

## Stores

**Memory** — this process's memory: tests, one-process apps, a fast first
level in front of a shared store. Thread-safe; `add`, `increment` and
`forgetIfValue` are atomic.

```grav
Cache.store(new MemoryCacheStore())
Cache.store(new MemoryCacheStore(10000), "local")   // at most 10,000 keys: expired, then oldest, go first
```

**File** — one file per key, under a directory; no dependency beyond the
standard library.

```grav
Cache.store(new FileCacheStore("/var/cache/app"))
```

Each write goes to a temp file, then an atomic rename, so a reader never
sees a half-written entry. `increment()` keeps the key's expiry, but is not
atomic across processes (a plain file has no server-side add); fine within
one process.

**Redis** — through [grav-redis](https://github.com/venomous-maker/grav-redis).

```grav
Cache.store(new RedisCacheStore(RedisConnection.open("redis://localhost:6379/0")))
Cache.store(new RedisCacheStore(conn, "myapp:cache:"), "sessions")   // a key prefix (default "cache:")
```

Expiry is Redis's own (`SET ... EX`); `increment()` is Redis's `INCRBY`
(atomic on the server); `flush()` deletes only this store's prefix (`SCAN`,
never `FLUSHDB` — the database may hold other things). `many()` is one `MGET`
and `putMany()` one pipelined round trip, however many keys they cover.

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
`has`, `forever`, `add`, `forgetIfValue` and `ttl` already have defaults,
which a store that can do them atomically overrides) and give it to
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
