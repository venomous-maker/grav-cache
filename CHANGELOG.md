# Changelog

Every release of grav-cache, newest first. Each is a `vX.Y.Z` tag with a GitHub
release carrying its section; the `grav` line in `grav.toml` says which
compiler versions it builds with.

## Unreleased

## 0.2.4 (2026-11-28)

### Fixes

- A Redis connection shared by several threads (the cache, session and queue-dashboard stores, used by a multi-threaded server or queue worker) no longer mixes up replies or stalls: every command is one locked round trip (grav-redis 0.2.2). Uses the new releases of grav-redis.

## 0.2.3 (2026-11-27)

### Fixes

- Database connections that the server dropped while idle (MySQL's `wait_timeout`, a restart, a proxy's idle limit) reconnect on their next use instead of failing with "server has gone away" (grav-db 0.2.1). Uses the new releases of grav-orbit.

## 0.2.2 (2026-11-25)

### Dependencies

- Uses grav-orbit 0.3.0 (from their GitHub tags).

## 0.2.1 (2026-11-20)

### Requirements

- Uses the 0.2.1 release of grav-redis.

### Performance

- Numbers, booleans, `null` and strings without escapes are stored and read
  back without building a JSON tree: a memory-store `get` of a number is more
  than twice as fast, and `put` nearly so ([bench/README.md](bench/README.md),
  new). A map is read by walking the parsed object's members instead of
  looking up each key, and `putMany` encodes with a loop instead of a closure.
- `FileCacheStore` names a key's file (its FNV-1a hash in hex) directly
  instead of formatting and padding two halves.

## 0.2.0 (2026-11-20)

### Requirements

- Needs grav 1.9.0 or later, and uses the 0.2.0 releases of the packages it
  depends on.

## 0.1.1 (2026-11-19)

- Uses the newer releases of grav-orbit, grav-db (faster row loading and query building in grav-orbit and grav-db).

## 0.1.0 (2026-11-19)

The first tagged release. Needs grav 1.7.0 or newer.

A cache for [Grav](https://github.com/venomous-maker/grav): `get`/`put`/ `remember` a value under a key, in memory, on a file, [Redis](https://github.com/venomous-maker/grav-redis), or a database table through [Orbit](https://github.com/venomous-maker/grav-orbit) — one of Laravel's Cache facade, in the manner of its array, file, Redis and database drivers — with atomic locks and a rate limiter on top of any store.
