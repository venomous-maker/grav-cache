# Changelog

Every release of grav-cache, newest first. Each is a `vX.Y.Z` tag with a GitHub
release carrying its section; the `grav` line in `grav.toml` says which
compiler versions it builds with.

## Unreleased

## 0.1.0 (2026-11-19)

The first tagged release. Needs grav 1.7.0 or newer.

A cache for [Grav](https://github.com/venomous-maker/grav): `get`/`put`/ `remember` a value under a key, in memory, on a file, [Redis](https://github.com/venomous-maker/grav-redis), or a database table through [Orbit](https://github.com/venomous-maker/grav-orbit) — one of Laravel's Cache facade, in the manner of its array, file, Redis and database drivers — with atomic locks and a rate limiter on top of any store.
