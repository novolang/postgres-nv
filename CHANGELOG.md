# Changelog

All notable changes to postgres-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

- README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `pgmsg` — every backend and frontend message of the version 3
  protocol as two enums, the two framing shapes (startup messages have
  no type byte), a length check against the caller's ceiling, and the
  three predicates the protocol's rules live in.
- `pgtype` — the built-in OIDs, both wire formats, the server settings
  a text decode depends on, `numeric` as digits rather than as a
  float, and the timestamp epoch that is not 1970.
- `pgauth` — SCRAM-SHA-256 as a value with the nonce supplied by the
  caller, the three refusals, MD5 and cleartext, and Hi(), the proof
  and the signature published separately because the RFC publishes
  their values.
- `pgconn` — the socket, the TLS hook, the startup handshake, the read
  loop that routes asynchronous messages, the notification queue, and
  cancellation from a key with no connection.
- `pgquery` — the simple and extended protocols, prepared statements,
  both pipelines, cursors over portals, and the isolation levels.
- `pgcopy` — `COPY` both ways, the binary header and trailer, and rows
  drained from chunks that are not rows.
- `pgpool` — a pool as a value with a policy, the clock as an
  argument, and the reset that stops a borrower inheriting somebody
  else's transaction.
- `pgdriver` — `PgDatabase` as the prelude's `Database`, the borrow
  path over a pool, and the published SQLSTATE mapping.
- `pgerror` — the whole `ErrorResponse` field set, the SQLSTATE as the
  thing to branch on, and `is_retryable` / `is_fatal` as separate
  questions.

### Known

- **The load-bearing interface is `pgmsg.PgBackend`, and only
  `PgReadyForQuery` ends an exchange.**  After an `ErrorResponse` the
  server discards every message until `Sync`, so a pipelining client
  counting results waits for a statement the server skipped and the
  connection hangs rather than errors.  `pipeline` carries one `Sync`
  per statement; `pipeline_unsynced` is the other form, by name, and
  it answers how many statements were skipped.
- **Three messages belong to no exchange** and arrive mid-result-set.
- **The `Database` contract's effect row is the widening this lane
  found**: the trait declares `[io]`, this impl declares `[net, time]`,
  and a `dyn` call is charged the union over every impl in the
  program.  The fix is an effect parameter on the trait, which is a
  contract change.
- **TLS is a hook the caller supplies**, and `PgSslMode` has no
  default because a server answering `N` has not failed.
- **The SCRAM nonce is the caller's**, which is what makes RFC 7677's
  vectors reproducible with no randomness in the module.
- **Takes crypto-nv and calendar-nv; refuses uuid-nv**, which would
  put a random-number generator into every program that selects a UUID
  column.
- **The pool is a value and does not synchronise**, so a pool per cell
  with no sharing is the shape that runs.
- **Out of scope and said so**: replication, multi-dimensional arrays,
  composite and range types, and `async` entry points.
