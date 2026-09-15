# postgres-nv

PostgreSQL is a relational database server. Clients talk to it over
the **version 3 frontend/backend protocol**, which is specified in the
[Frontend/Backend Protocol](https://www.postgresql.org/docs/current/protocol.html)
chapter of the PostgreSQL manual. This package speaks that protocol in
novo-lang with no `libpq` underneath it: the messages, the
authentication, the two query protocols, the value formats, `COPY`,
`LISTEN`/`NOTIFY`, cancellation, a connection pool and a `std.sql`
driver.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What the protocol is

A client opens a socket and sends a **startup message** naming the
user and the database. The server answers with an **authentication
request**, and once that is settled it sends a **ready for query**
message. From then on both ends exchange **messages**, each one a
1-byte type, a 4-byte length and a body.

The startup messages are the exception: `StartupMessage`,
`SSLRequest`, `GSSENCRequest` and `CancelRequest` have no type byte,
only a length and then a version or a request code.

A statement is sent in one of two protocols. The **simple query
protocol** is one message carrying SQL text. It can carry several
statements separated by semicolons, it always answers text, and it has
no parameters. The **extended query protocol** is `Parse`, `Bind`,
`Execute` and `Sync`: the statement is parsed once, the parameter
values are sent beside it rather than inside it, the result format is
chosen per column, and a parsed statement can be executed many times.

**Only `ReadyForQuery` ends an exchange.** The server's answer to a
statement has no length and no count: a parse acknowledgement, a bind
acknowledgement, a row description, some number of data rows, a
command completion, and then ready for query. A client that acted on
the command completion acted inside a transaction the server may have
already aborted, because the transaction status travels only on
`ReadyForQuery`.

**An error makes the server discard everything until it sees a
`Sync`.** Send two statements under one `Sync`, have the first fail,
and the second never runs. A client counting command completions waits
for a result that will never come, and the connection hangs rather
than fails.

**Three messages belong to no exchange.** `ParameterStatus`,
`NoticeResponse` and `NotificationResponse` arrive whenever the server
has something to say, including in the middle of a result set.

A value arrives in one of two **formats**. **Text** is the value
rendered as characters, and how it renders depends on session settings
the server announces in `ParameterStatus`. **Binary** is the type's
own layout, and it does not depend on any setting.

An error from the server is a **field set**, not a string: a severity,
a five-character **SQLSTATE**, a message, and up to fifteen more
fields including the schema, the table, the column and the constraint
that failed. The SQLSTATE is the part to branch on. The message is
localised, and `23505` is `unique_violation` on every server in every
locale.

The numbers worth knowing are these.

| Quantity | Value |
| --- | --- |
| Message header | 5 bytes: a 1-byte type and a 4-byte length |
| Startup message header | 4 bytes of length, then a 4-byte version or request code |
| Service port | 5432 |
| SQLSTATE | exactly 5 characters |
| Timestamp epoch | 2000-01-01T00:00:00Z, in microseconds |
| SCRAM iteration floor this package enforces | 4096 |
| Answer to an `SSLRequest` | one byte, `S` or `N`, and not a message |
| Binary `COPY` signature | 11 bytes, and the stream ends with a trailer of −1 |

## Install

```
novo pkg add postgres-nv
```

## Example

```novo
use std.list
use pgconn
use pgerror
use pgmsg
use pgquery
use pgtype

fn main() [io, net, time]
    // Where to connect and as whom. `plain_transport` is the standard
    // library's socket; a program that wants TLS supplies its own.
    let opts = pgconn.default_options("app")

    match pgconn.connect(opts, pgconn.plain_transport())
        Err(f) => println(pgerror.describe(f))
        Ok(c)  =>
            // The value goes as a parameter and never into the string.
            // `PgBinary` asks for the results in binary, which does not
            // depend on the server's date and float settings.
            match pgquery.query(c, "SELECT name FROM users WHERE id > $1",
                                [PgInt(v: 100)], PgBinary)
                Err(f) => println(pgerror.describe(f))
                Ok(e)  =>
                    // One row's first column, rendered as text.
                    match list.first(e.result.rows)
                        None      => println("no rows")
                        Some(row) =>
                            match list.first(row)
                                None    => println("no columns")
                                Some(v) => println(pgtype.show(v))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: postgres-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `pgmsg` | Every message the protocol has, in both directions, with the framing and the four predicates a read loop needs. |
| `pgtype` | The type OIDs, the value type, the two value formats, the session settings a text value depends on, and the encoders and decoders. |
| `pgauth` | SCRAM-SHA-256 as a four-message exchange over values, and the MD5 and cleartext methods. |
| `pgerror` | The server's field set, the SQLSTATE and its class, and every fault that happens before the server gets a chance to answer. |
| `pgconn` | The socket, the startup handshake, the TLS negotiation, the read loop, the notification queue and cancellation. |
| `pgquery` | Both query protocols, prepared statements, cursors, pipelining and the transaction wrappers. |
| `pgcopy` | `COPY` in both directions, in text, CSV and binary. |
| `pgpool` | A pool of connections as a value, and the three policy decisions a pool exists to make. |
| `pgdriver` | The `std.sql` `Database` implementation, so a program can hold a database without naming an engine. |

The first four modules perform no input or output at all: bytes in,
messages out, and messages in, bytes out. That is what lets the whole
protocol be tested against wire captures with no server running, and
what lets the same codec serve a client, a connection pooler and a
test double. The last five declare `[net]`, and `[time]` where a
deadline is consulted. No module here declares `[io]` or `[fs]`.

## How to choose an entry point

**`pgquery.query` is the ordinary call.** It runs one statement with
its parameters in the extended protocol and collects the rows. It uses
the extended protocol even when there are no parameters, so that the
parameterised path is the one every program exercises.

**`pgquery.simple` sends SQL text with no parameters.** It is the only
way to run `BEGIN; …; COMMIT` in one round trip and the only way to
run a script. It has no parameters at all, which is where SQL
injection lives.

**`pgquery.prepare` and `execute` keep the parsed statement**, for a
statement that runs many times.

**`pgquery.open_cursor` and `fetch` read a result in batches**, for a
query whose answer is larger than memory.

**`pgquery.pipeline` sends several statements in one round trip.** It
carries one `Sync` per statement. `pipeline_unsynced` is the form that
does not, and it answers which statements the server skipped.

**`pgcopy` moves bulk data.** A million rows as a million `INSERT`s is
a million round trips and a million plans; the same rows as one `COPY
FROM STDIN` is one statement and one stream.

**`pgdriver.PgDatabase` is for a program that must not name an
engine.** It implements the standard library's `Database` trait, so
the same code runs over this client and over sqlite-nv. The trait has
no row surface, so a caller that wants rows calls `pgdriver.query` and
has chosen PostgreSQL by then.

**`pgmsg`, `pgtype`, `pgauth` and `pgerror` together are the whole
protocol with no socket.** That is the path for a pooler, a proxy, a
capture reader and a test.

## The rules a user needs

1. **One socket read is not one message.** A read returns whatever
   arrived. `pgmsg.frame_length` says how much of a buffer is one
   message, and `pgconn.next_message` reassembles across reads.
2. **Bound the message size before the first read.**
   `pgmsg.frame_length` takes `max_bytes`. Without it, the length
   field on the wire is an allocation the wire asked for.
3. **Only `ReadyForQuery` ends an exchange, and only it carries the
   transaction status.** `pgmsg.tx_status` answers `Some` for that
   message and `None` for every other. `pgquery.in_failed_transaction`
   is the question to ask before sending more work.
4. **An error makes the server discard until `Sync`.**
   `pgmsg.discards_until_sync` is the predicate. Use
   `pgquery.pipeline`, which carries one `Sync` per statement.
   `pipeline_unsynced` answers a `skipped` list, and a caller has to
   read it.
5. **Route the three asynchronous messages first.**
   `pgmsg.is_asynchronous` is true for `ParameterStatus`,
   `NoticeResponse` and `NotificationResponse`. A read loop that does
   not route them first is one whose `SELECT` occasionally returns a
   notification where a row should be.
6. **A notice is not an error.** `NoticeResponse` carries the same
   field set as an error and arrives at any moment. A client that
   treats it as an error fails a query that succeeded, and one that
   drops it loses the output a `RAISE NOTICE` exists to produce.
7. **Branch on the SQLSTATE, never on the message text.** The message
   is localised. `pgerror.sqlstate` is the five characters, and
   `pgerror.sqlstate_class` is the first two, which is the useful
   grouping. There are about two hundred and fifty codes and
   extensions add their own, which is why it is a `Str` and not an
   enum.
8. **Keep the whole field set.** `PgFields` carries the schema, the
   table, the column, the data type and the constraint. A caller
   asking which unique index an insert violated needs
   `constraint_name`, and one underlining the offending token needs
   `position`.
9. **Ask for binary results.** A text value's meaning depends on the
   server's `DateStyle`, `IntervalStyle`, `TimeZone`,
   `client_encoding` and `integer_datetimes`. `pgtype.PgTextSettings`
   carries those, it is an argument to every text decode, and `pgconn`
   keeps it current from the `ParameterStatus` messages. Binary has no
   such dependency.
10. **A timestamp is microseconds since 2000-01-01, not 1970.**
    `PgTimestamp` and `PgTimestampTz` carry a calendar type for that
    reason, and `pgtype.postgres_epoch_day()` is public for a caller
    who wants the arithmetic.
11. **`timestamp` and `timestamptz` are different things.** A
    `timestamp` is a wall clock reading with no zone, and two of them
    from different sessions are not comparable. A `timestamptz` is an
    instant. They are separate variants, because collapsing them is
    how an application ends up an hour out twice a year.
12. **A NULL is not an empty string.** On the wire a NULL is a length
    of −1 and an empty text value is a length of 0. `PgNull` is a
    variant of `PgValue`, and no answer in `pgtype` is optional,
    because `?PgValue` would make a NULL and a missing column the same
    thing.
13. **`numeric` stays a `numeric`.** `pgtype.numeric_text` renders the
    digits. `numeric_to_float` exists and rounds, which for money is
    the wrong answer.
14. **`PgReadCommitted` does not mean what it means in other
    engines.** In PostgreSQL each statement sees a snapshot taken when
    that statement started, so two statements in one transaction can
    see different data. `PgRepeatableRead` is one snapshot for the
    whole transaction, and it also prevents phantom reads.
15. **`PgSerializable` needs a retry loop.** It is enforced by
    aborting transactions with SQLSTATE 40001.
    `pgerror.is_retryable` is what a caller branches on.
16. **Check the SCRAM server signature.** SCRAM is mutual
    authentication: the server's final message proves it knows the
    stored key, and a client that skips the check has reduced SCRAM to
    a password send. `pgauth.verify_server` does it with a
    constant-time comparison, because the value it compares against is
    one an attacker supplies.
17. **A cancel is a second connection.** `CancelRequest` is sent on a
    fresh socket carrying the process id and secret key from the
    startup handshake. `pgconn.cancel_key` publishes them, and
    `pgconn.cancel_by_key` sends the request from anywhere.
18. **A `CopyData` message is not a row.** The chunking is the
    server's: a row can span two messages and a message can carry
    twenty rows. `pgcopy.recv_rows` drains what is complete rather
    than what arrived.
19. **A binary `COPY` stream needs its trailer.**
    `pgcopy.binary_trailer` is it. A stream without one is accepted
    and then reported as truncated at the very end, after everything
    has been sent.
20. **A connection returned to the pool is checked and rolled back.**
    A connection returned in the middle of a transaction hands the
    next borrower an open transaction. `pgpool.release` reads the
    ready-for-query status rather than trusting the borrower.
21. **A pool's limit is not advice.** Past it, `acquire` waits or
    refuses, and `PgPoolPolicy.wait_ms` decides which. A PostgreSQL
    server's cost per connection is a process, so a pool that opens on
    demand turns a traffic spike into a connection storm.
22. **An idle connection can be closed by the server, by a firewall or
    by `idle_in_transaction_session_timeout`, and none of them tells
    the client.** A connection past `max_idle_ms` is closed rather
    than lent, and one that has served `max_uses` is recycled, which
    also bounds the memory a long-lived backend accumulates from
    prepared statements.

## Authentication

`pgauth` performs nothing: no socket, no clock and no randomness. The
client nonce is the caller's argument. RFC 7677 section 3 publishes a
complete SCRAM-SHA-256 exchange with every intermediate value, and a
module with no randomness in it reproduces that byte for byte. It is
the only way to know a signature is right, because a wrong one fails
to log in and every wrong one fails to log in identically.

Three things are refused, because in each case the plausible behaviour
is the unsafe one.

| Refused | Why |
| --- | --- |
| An iteration count below 4096 | `i=1` is a legal SCRAM message and makes the key derivation free, which turns a client into an offline cracking oracle for its own password |
| `SCRAM-SHA-256-PLUS` with no channel binding available | Negotiating down to plain SCRAM hands an attacker who terminates the TLS exactly what the binding was for |
| A nonce containing a comma | It is SCRAM's own field separator, so the server parses the message differently from the client |

**MD5 is implemented rather than refused**, because refusing it means
refusing to connect to a real server. It is weak: the stored verifier
is the password equivalent. **Cleartext is implemented**, and `pgconn`
refuses it over an unencrypted TCP socket unless the caller has set
`allow_cleartext`.

Channel binding needs the peer certificate's hash, which only the TLS
transport can produce. `PgChannelBinding` is declared so that a caller
can supply it.

## TLS

**TLS is a hook, not a dependency.** A driver that chose a TLS library
would choose it for every program that links the driver.
`pgconn.PgTransport` is the seam: named functions that move bytes, and
`pgconn.plain_transport` is the `std.net` pair.

`SSLRequest` is one eight-byte packet sent before anything else, and
the server answers a single byte, `S` or `N`. So
`pgconn.negotiate_tls` is a separate call with the caller's TLS
handshake between it and the startup message.

`PgSslMode` has five values and no default. **A server that answers
`N` has not failed.** It has said no, and a client that carried on has
downgraded a connection somebody asked to encrypt. `PgSslRequire` and
above refuse.

## What is not included

- **A TLS implementation.** `PgTransport` is where one goes.
- **Replication.** `CopyBothResponse` is in the message enum, because
  a decoder that did not know it would misframe a stream containing
  one. The logical and physical replication protocols on top of it are
  a package of their own.
- **Multi-dimensional arrays and arrays with a lower bound other than
  1.** A one-dimensional array is `PgArray`. Anything else arrives as
  `PgUnknown`, with its bytes intact, rather than as a wrong answer.
- **Composite types, range types and extension types.** `PgUnknown`
  carries the OID, the format and the bytes, so an extension's type
  reaches a caller who knows what to do with it.
- **A UUID type.** `PgUuid` carries the sixteen bytes in wire order
  and `pgtype.uuid_text` renders them. Naming another package's UUID
  type here would put a random-number generator into every program
  that selects a UUID column.
- **A JSON parser.** The wire format for `json` is text, and for
  `jsonb` it is a version byte and then the same text. `PgJson`
  carries it and `std.json` parses it.
- **Asynchronous calls.** Every call blocks its task. `std.net` has
  `recv_async` and `accept_async`, and this package does not use them
  yet.
- **A pool that synchronises.** `pgpool.PgPool` is a value, so two
  tasks sharing one share it the way they share any other value in
  this language. A pool per cell with no sharing is the shape that
  runs.

## Related packages

- [mysql-nv](https://novo-lang.org/packages/mysql-nv) is the same
  shape for MySQL and MariaDB. The two protocols differ in five places
  worth knowing about.

| | postgres-nv | mysql-nv |
| --- | --- | --- |
| Message shape | Fixed by the type byte | Negotiated: every decode takes the capability flags |
| Framing | A 4-byte length, one message per frame | A 3-byte length and a sequence id, split at 16 MB |
| End of exchange | A `ReadyForQuery` message | Nothing: the status rides every OK packet's flags |
| Parameters | The extended query protocol, either format | Only a prepared statement has parameters |
| Cancelling | An out-of-band request on a fresh socket | `KILL QUERY` on a second connection |

- [sqlite-nv](https://novo-lang.org/packages/sqlite-nv) reads and
  writes an SQLite database file directly. No server and no socket.
- [migrate](https://novo-lang.org/packages/migrate) plans schema
  migrations and produces their SQL. It runs nothing, so a caller
  hands its steps to this package. Its placeholder style for
  PostgreSQL is `$1`.
- [query-builder-nv](https://novo-lang.org/packages/query-builder-nv)
  builds statements as values and renders them for PostgreSQL's
  dialect, including its numbered placeholders and its `RETURNING`
  clause.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) is the
  SHA-256, HMAC, MD5 and constant-time comparison `pgauth` is built
  on. A server built this decade answers only `AuthenticationSASL`, so
  this dependency is not optional.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the
  civil date and date-time `PgDate`, `PgTimestamp` and `PgTimestampTz`
  carry.
- `std.sql` in the standard library opens an SQLite file by shelling
  out to the `sqlite3` binary. It is the `Database` contract this
  package implements, not a PostgreSQL client.

## Tests

```bash
novo test tests/pgmsg_tests.nv    #  8 tests: the framing and the four predicates
novo test tests/pgtype_tests.nv   #  9 tests: both value formats and the settings
novo test tests/pgauth_tests.nv   #  8 tests: SCRAM against RFC 7677's vectors
novo test tests/pghost_tests.nv   # 12 tests: the connection, the queries and the pool
```

The wire is PostgreSQL's own Frontend/Backend Protocol chapter, which
is the normative document. RFC 5802 and RFC 7677 are the SCRAM
specifications, and RFC 7677 section 3's published exchange is what
`pgauth_tests.nv` asserts against, intermediate value by intermediate
value. `tokio-postgres` and `psycopg` are the reference
implementations for the shape of the API.

No test opens a socket. The nonce, the salt and the server's bytes are
all arguments, so a handshake is a value the test writes out and the
same bytes produce the same proof on every run. The suite checks that
a message split across two reads is reassembled, that a length field
past the ceiling is refused, that an error message sets the
discard-until-sync state, that an asynchronous message arriving inside
a result set is routed and not counted as a row, that a text value
reads differently under `DateStyle = SQL, DMY`, that a timestamp is
read against the 2000 epoch, and that a SCRAM iteration count of 1 is
refused.

The tests compile today and fail at run, each on the
`not implemented: postgres-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green
one at a time as bodies land.

## Implementation status

Nothing is implemented. Every function below is a `todo()`.

| Module | Public surface |
| --- | --- |
| `pgmsg` | `protocol_version`, `ssl_request_code`, `cancel_request_code`, `frame_length`, `frame_kind`, `decode_backend`, `encode_frontend`, `encode_batch`, `discards_until_sync`, `ends_exchange`, `is_asynchronous`, `tx_status`, `tag_rows`, `tag_command` |
| `pgtype` | The sixteen type-OID accessors, `postgres_epoch_day`, `numeric`, `numeric_nan`, `default_settings`, `apply_parameter`, `decode`, `decode_row`, `encode`, `oid_for`, `uuid_text`, `numeric_to_float`, `numeric_text`, `show` |
| `pgauth` | `min_iterations`, `choose_mechanism`, `begin`, `client_first`, `server_first`, `client_final`, `verify_server`, `salted_password`, `client_proof`, `server_signature`, `saslprep`, `md5_password`, `cleartext_password` |
| `pgerror` | `describe`, `fields_new`, `set_field`, `sqlstate`, `sqlstate_class`, `is_retryable`, `is_fatal` |
| `pgconn` | `plain_transport`, `default_options`, `is_unix_socket`, `connect`, `negotiate_tls`, `close`, `next_message`, `send`, `send_batch`, `drain_to_ready`, `take_notifications`, `wait_notification`, `parameter`, `cancel`, `cancel_key`, `cancel_by_key` |
| `pgquery` | `simple`, `prepare`, `close_statement`, `execute`, `query`, `execute_count`, `pipeline`, `pipeline_unsynced`, `open_cursor`, `fetch`, `close_cursor`, `begin`, `commit`, `rollback`, `in_failed_transaction` |
| `pgcopy` | `begin_in`, `send_raw`, `send_row`, `finish_in`, `fail_in`, `begin_out`, `recv_raw`, `recv_rows`, `finish_out`, `binary_header`, `binary_trailer`, `encode_binary_row`, `encode_text_row` |
| `pgpool` | `default_policy`, `new_pool`, `warm`, `acquire`, `release`, `discard`, `reap`, `close_pool`, `stats`, `has_idle` |
| `pgdriver` | `open`, `of_conn`, `borrow`, `give_back`, `last_fault`, `conn_of`, `query`, `to_db_error`, and the `Database` members `close`, `exec` and `query_count` |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
