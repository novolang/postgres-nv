# postgres-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

A PostgreSQL client, in novo-lang, with no libpq.

The version 3 wire protocol is a documented, stable, thirty-year-old
message format, and every language that talks to PostgreSQL without C
has ported it — Go, Rust, Java, Erlang, and PostgreSQL's own JDBC
driver.  This is that port: the messages, the authentication, the two
query protocols, the type formats, `COPY`, `LISTEN`/`NOTIFY`,
cancellation and a pool.

The `libpq-sys` row on [the bindings shelf](https://novo-lang.org/docs/orbit-map.html)
stays there as the escape hatch; after this package lands, nothing on
the grid needs it.

Nine modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **messages** | `pgmsg` | anything. Start here |
| the **values** | `pgtype` | you are reading or binding a column |
| the **login** | `pgauth` | you are debugging a handshake |
| the **connection** | `pgconn` | you want the socket, or a cancel |
| the **queries** | `pgquery` | you want rows |
| the **bulk** | `pgcopy` | you are moving a million rows |
| the **pool** | `pgpool` | you have more than one request at a time |
| the **contract** | `pgdriver` | you want a `dyn Database` |
| the **faults** | `pgerror` | something went wrong |

## Adding it, and checking it

```bash
novo pkg add postgres-nv         # into your novo.toml
novo pkg build                   # type- and effect-check the package
novo test --isolate tests/pgmsg_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: postgres-nv.<module>.<fn>`.
They turn green one at a time as bodies land.

## The one example that will work

```novo
use pgconn
use pgquery
use pgmsg
use pgtype

// Ask a parameterised question, in one round trip, in binary.
fn count_after(host: Str, id: Int) -> Int [net, time]
    match pgconn.connect(pgconn.default_options("app"), pgconn.plain_transport())
        Err(f) => -1
        Ok(c) =>
            match pgquery.query(c, "SELECT count(*) FROM t WHERE id > $1",
                                [pgtype.PgInt(v: id)], pgmsg.PgBinary)
                Err(f) => -1
                Ok(e)  => list.len(e.result.rows)
```

The value goes as a parameter, not into the string.  That is not a
style preference: it is what makes SQL injection structurally
impossible rather than a matter of escaping.

## The layer, and why

`host`, and the split inside it is the point.

`pgmsg`, `pgtype`, `pgauth` and `pgerror` are `[]` throughout — bytes
in, messages out, messages in, bytes out, and not one socket.  That is
what lets the whole protocol be tested against wire captures with no
server running, and it is what lets the same codec serve a client, a
connection pooler and a test double.

`pgconn`, `pgquery`, `pgcopy`, `pgpool` and `pgdriver` connect.  They
declare `[net]`, and `[time]` where a deadline is consulted, and
nothing else.  `[io]` is not owed: `std.net`'s free functions —
`net.connect`, `net.send_bytes`, `net.recv_bytes` — declare `[net]`
alone, and no module here prints.

**The `core` half is a MODULE split rather than a package split**, and
that is a decision rather than an omission.  A codec whose only
consumer is the client beside it is a second package a reader has to
assemble, with no second caller to pay for it.  The row that would
change that is a PostgreSQL wire PROXY — a pooler, a protocol-aware
load balancer, a query router — and if one is written, `pgmsg` and
`pgtype` move out as `pg-codec-nv` with no signature change.

## The load-bearing interface

`pgmsg.PgBackend`, and the rule that **only `PgReadyForQuery` ends an
exchange.**

```novo norun:pseudo
pub enum PgBackend
    PgAuthentication(request: PgAuthRequest)
    PgReadyForQuery(status: PgTxStatus)
    PgDataRow(values: [Bytes], nulls: [Bool])
    PgErrorResponse(fields: pgerror.PgFields)
    // … twenty more
```

Every byte a PostgreSQL server sends becomes one of these.  What makes
the enum load-bearing is not that it exists — every client has one —
but the three rules that come with it, each of which is a public
predicate here because each one is a silent bug elsewhere.

**One: an error makes the server discard until `Sync`.**  This is the
one that hangs a connection instead of failing it.  Send
Parse/Bind/Execute for two statements under one `Sync`, have the first
fail, and the second NEVER RUNS — the server skipped it — but a client
counting `CommandComplete`s is still waiting for a result that will
never come.  `pgmsg.discards_until_sync` is the predicate;
`pgquery.pipeline` carries one `Sync` per statement, and
`pgquery.pipeline_unsynced` is the name of the dangerous form so that
using it is a decision.  It answers `skipped`, and a caller has to look
at it.

**Two: three messages belong to no exchange.**  `ParameterStatus`,
`NoticeResponse` and `NotificationResponse` arrive whenever the server
feels like it, including in the middle of a result set.
`pgmsg.is_asynchronous` is the predicate and `pgconn.next_message`
routes on it first — a read loop that did not is one whose `SELECT`
occasionally returns a notification where a row should be.

**Three: the transaction status only travels on `ReadyForQuery`.**  A
client that acted on a `CommandComplete` acted inside a transaction the
server may have already aborted.  `pgmsg.tx_status` answers `Some` for
that message and `None` for every other one, and `pgquery.in_failed_transaction`
is how a caller asks the question it has to ask before sending more
work.

## What the wire gets wrong quietly, and where each one has a name

| the mistake | what it costs | where it is named |
| --- | --- | --- |
| one socket read treated as one message | works on localhost, fails under load | `pgmsg.frame_length` |
| the length field trusted | an allocation the wire asked for | `frame_length`'s `max_bytes` |
| pipelining under one `Sync` | a hang, not an error | `pgmsg.discards_until_sync` |
| `ParameterStatus` ignored | dates read a month out | `pgtype.PgTextSettings` |
| the timestamp epoch taken as 1970 | dates thirty years early | `pgtype.postgres_epoch_day` |
| a NULL read as an empty string | two different values collapsed | `pgtype.PgNull` |
| `numeric` converted to `Float` | money, rounded | `pgtype.numeric_text` |
| the SCRAM server signature unchecked | mutual auth downgraded to a password send | `pgauth.verify_server` |
| a SCRAM iteration count of 1 accepted | an offline cracking oracle | `pgauth.min_iterations` |
| a connection returned mid-transaction | the next borrower inside somebody else's | `pgpool.release` |
| the binary `COPY` trailer omitted | truncated, reported at the very end | `pgcopy.binary_trailer` |

## Authentication

`pgauth` is `[]` — no socket, no clock, no randomness — and the nonce
is the CALLER's argument.  That is not a workaround for the effect
budget; it is what makes the handshake testable.  RFC 7677 § 3
publishes a complete SCRAM-SHA-256 exchange with every intermediate
value, and a module with no randomness in it reproduces that byte for
byte.  It is the only way to know a signature is right: a wrong one
fails to log in, and every wrong one fails to log in identically.

Three refusals, each because the plausible behaviour is the insecure
one:

- **an iteration count below 4096** — `i=1` is a legal SCRAM message
  and makes Hi() free, which is what a hostile server sends to turn a
  client into an offline cracking oracle for its own password;
- **`SCRAM-SHA-256-PLUS` offered with no channel binding available** —
  negotiating down to plain SCRAM hands an attacker who terminates the
  TLS exactly what the binding was for;
- **a nonce containing a comma** — SCRAM's own field separator, so the
  server parses the message differently from the client.

MD5 is implemented rather than refused, because refusing it means
refusing to connect to a real server; it is weak (the stored verifier
IS the password equivalent) and the module says so where it is
declared.  Cleartext is implemented and `pgconn` refuses it over an
unencrypted TCP socket unless the caller has set `allow_cleartext`.

## TLS is a hook, not a dependency

A driver that chose a TLS implementation would choose it for every
program that links the driver.  `pgconn.PgTransport` is the seam: the
caller supplies three named functions that move bytes, and
`pgconn.plain_transport` is the `std.net` pair.  `SSLRequest` is one
eight-byte packet before any of it — the server answers a single BYTE,
`S` or `N` — so `pgconn.negotiate_tls` is a separate call with the
caller's handshake between it and the startup packet.

`pgconn.PgSslMode` has five values and no default, because **a server
that answers `N` has not failed.**  It has said "no TLS", and a client
that carried on has silently downgraded a connection the user asked to
encrypt.  `PgSslRequire` and above refuse.

The `tls-nv` row on the grid is planned and is not a dependency of
this package either way.

## The `std.sql` driver

`pgdriver.PgDatabase` implements the prelude's `Database` trait, so a
program written against `dyn Database` runs over this client and over
sqlite-nv with nothing in it naming either.

**What the effect row costs, stated rather than discovered.**  The
trait's members declare `[io]`; this impl declares `[net, time]`,
which is legal — a trait with no effect parameter does not pin its
impls' rows, and the standard library's own `SqliteDb` already
declares `[fs]` against the same `[io]`.  The bill is that **a `dyn
Database` call is charged the UNION over every impl in the program**
(SPEC § 5.6), so a program that links this package makes every `dyn
Database` call in it cost `[net, time]` — including the ones that only
ever hold a file.  Concrete receivers are charged their own rows, so
the union is only paid where the engine really is unknown.

This is the same obstruction `Connection` records in its own header,
and it has the same fix: an effect parameter on the trait, so each
impl supplies what it costs.  It is a **contract change** and belongs
in a feature file; this package names it rather than working around
it, and it is the widening this lane found.

## Two dependencies, and two refusals

**crypto-nv**, and it is not optional: a server built this decade
answers `AuthenticationSASL` and nothing else, so a client without
SCRAM-SHA-256 cannot log in.  `pgauth` uses `hmac_sha256`, `sha256`,
`md5` and `digest.ct_eq` — the last for the server-signature check,
which compares against a value an attacker controls and is therefore
the one comparison here that may not be `==`.

**calendar-nv**, for one type, because the type is where the trap is.
A PostgreSQL timestamp is microseconds since **2000-01-01**, and a
driver that answered a bare integer would be handing back the trap with
a plausible name on it.  calendar-nv is `core` with no dependencies of
its own.

**Not uuid-nv.**  It depends on rand-nv, so naming `uuid.Uuid` in a
signature would put a random-number generator into every program that
selects a UUID column — and a driver has no business shipping a v4
generator.  `pgtype.PgUuid` carries the sixteen bytes in wire order and
`pgtype.uuid_text` renders them; a caller that wants uuid-nv's type
calls `uuid.from_bytes`, in one line, having chosen to.

**Not a json package.**  The wire format for `json` IS text, and for
`jsonb` it is a version byte and then the same text.  `PgJson` carries
it and `std.json` parses it.

## What is out of scope, out loud

**Replication.**  `CopyBothResponse` is in the message enum because a
decoder that did not know it would misframe a stream that contained
one, but the logical and physical replication protocols on top of it
are a package of their own.

**Multi-dimensional arrays and non-1 lower bounds.**  A one-dimensional
array is `PgArray`; anything else arrives as `PgUnknown` with its bytes
intact rather than as a wrong answer.

**Composite and range types, and every extension type.**  `PgUnknown`
carries the OID, the format and the bytes, so an extension's type
reaches a caller who knows what to do with it.

**`async`.**  Every call here blocks its task.  `std.net` has
`recv_async` and `accept_async` and this package does not use them yet;
the row that wants it is a server handling many connections per cell,
and the change is an effect row and a second set of entry points
rather than a redesign.

**A pool that synchronises.**  `pgpool.PgPool` is a VALUE, so it does
not synchronise anything and cannot: two tasks sharing one share it the
way they share any other value in this language.  `cell.pool` is how
novo-lang programs own shared state, and a pool per cell with no
sharing is the shape that actually runs.

## The reference implementations

tokio-postgres and psycopg, for the API shape; PostgreSQL's own
[Frontend/Backend Protocol](https://www.postgresql.org/docs/current/protocol.html)
chapter for the wire, which is the normative document and what the
module headers transcribe; RFC 5802 and RFC 7677 for SCRAM, whose
published vectors `tests/pgauth_tests.nv` is written against.

The implementation lane's gate is a real server: `initdb`, a socket, a
corpus of statements, and the same queries through `psql` for
comparison.

## Status

Interface only.  Nine modules, 125 public functions and three trait
members, every body a `todo()`.

- `novo pkg build` — clean, 9 modules checked.
- `novo test` — four suites, all red, every failure `not implemented`.
- `scripts/shard_audit.sh --strict` — `effect-budget`, `dep-layer`,
  `no-discharge-in-core`, `doc-examples` and `docs-pub` green; `test`
  red by design.
