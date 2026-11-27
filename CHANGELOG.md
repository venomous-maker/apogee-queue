# Changelog

Every release of apogee-queue, newest first. Each is a `vX.Y.Z` tag with a GitHub
release carrying its section; the `grav` line in `grav.toml` says which
compiler versions it builds with.

## Unreleased

## 0.3.3 (2026-11-28)

### Fixes

- A Redis connection shared by several threads (the cache, session and queue-dashboard stores, used by a multi-threaded server or queue worker) no longer mixes up replies or stalls: every command is one locked round trip (grav-redis 0.2.2). Uses the new releases of grav-cache, grav-redis.

## 0.3.2 (2026-11-27)

### Fixes

- Database connections that the server dropped while idle (MySQL's `wait_timeout`, a restart, a proxy's idle limit) reconnect on their next use instead of failing with "server has gone away" (grav-db 0.2.1). Uses the new releases of grav-cache, grav-db.

## 0.3.1 (2026-11-25)

### Dependencies

- Uses grav-cache 0.2.2 (from their GitHub tags).

## 0.3.0 (2026-11-20)

### Added

- Job events (`JobQueued`, `JobProcessing`, `JobProcessed`, `JobReleased`,
  `JobFailed`) carry the job's `payload` (its data, tags, tries, pushed time),
  for dashboards and monitoring; `JobQueued` from `bulk()` has the job's id.
- Workers sharing a cache can be paused and continued together:
  `Worker.signalPause(cache)`, `Worker.signalContinue(cache)`,
  `Worker.isPausedIn(cache)` (checked about once a second).

## 0.2.1 (2026-11-20)

### Requirements

- Uses the 0.2.1 releases of grav-cache and grav-redis.

## 0.2.0 (2026-11-20)

### Requirements

- Needs grav 1.9.0 or later, and uses the 0.2.0 releases of the packages it
  depends on.

## 0.1.1 (2026-11-19)

- Uses the newer releases of grav-cache, grav-orbit, grav-db (faster row loading and query building in grav-orbit and grav-db).

## 0.1.0 (2026-11-19)

The first tagged release. Needs grav 1.7.0 or newer.

Queues for [Apogee](https://github.com/venomous-maker): jobs that run later, on workers.
