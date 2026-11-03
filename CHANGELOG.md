# Changelog

All notable changes to postgres-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-03

### Fixed

- A backend message whose option, column, parameter or format count runs past the end of its body is refused, where a count of 2147483647 in a NegotiateProtocolVersion message used to loop without end.
- A data row, row description, parameter description, COPY response or function call result that declares more than its body holds is refused instead of decoding as zero-filled values.
- A SCRAM iteration count above 10000000 is refused, so a server cannot hold the client in key derivation.

### Changed

- Error propagation reads `!` where it was a `match` returning the `Err` unchanged, the array-element reader, the text-format escape table and the private searches (`parameter_index`, `nul_at`, `first_number_at`, `zone_start`, `element_end`) use `match` and `?Int` rather than `-1`, and byte codes are character literals; no public signature or behaviour changes.

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-13

The bodies land.  Every signature the 0.0.1 release declared now
performs what it says: the codec, both authentication exchanges, the
connection with its read loop and its notification queue, the simple
and extended query protocols, `COPY` both ways, the pool and the
`std.sql` driver.  `stability` moves to `experimental`, and 166 tests
across thirteen suites pass.

### Added

- The four codec modules.  `pgmsg` decodes and encodes every version 3
  message in both directions, including the two framing shapes (a
  startup message's length is not preceded by a type byte) and the
  check against the caller's byte ceiling; `pgtype` reads and writes
  both wire formats, keeps `numeric` as digits rather than as a float,
  and reads a timestamp against PostgreSQL's 2000-01-01 epoch rather
  than the Unix one; `pgauth` runs SCRAM-SHA-256 as a value with the
  nonce supplied by the caller, so RFC 7677's published exchange is
  reproducible, and carries MD5 and cleartext beside it; `pgerror`
  models the whole `ErrorResponse` field set and the SQLSTATE classes
  a retry decision is made on.
- The five connecting modules.  `pgconn` opens the socket, negotiates
  `SSLRequest`, runs the startup handshake, routes the three messages
  that belong to no exchange, queues notifications, and cancels from a
  key with no connection; `pgquery` implements the simple and extended
  protocols, prepared statements, cursors, both pipeline forms and the
  transaction wrappers; `pgcopy` streams `COPY` in text, CSV and
  binary, with the binary header and trailer; `pgpool` is a pool as a
  value with a policy and the reset that stops a borrower inheriting
  somebody else's transaction; `pgdriver` implements the standard
  library's `Database`.
- Four suites beyond the interface release's four.
  `pgauth_vectors_tests.nv` pins the published SCRAM exchange value by
  value and then the same exchange as PostgreSQL spells it, byte for
  byte; `pgtype_paths_tests.nv` walks every decoder path in both
  formats and every refusal; `pgwire_edge_tests.nv` pins every length
  field and the messages this codec refuses, with no socket anywhere;
  `pgwire_server_tests.nv` runs the real client against a scripted
  PostgreSQL on an ephemeral loopback port.
- Five more suites, each driving a path the others leave dark, and
  between them every instrumented line under `src/` that is not
  excused by a `cov: skip` marker is reached by an assertion rather
  than by a call that happened to run it.
  `pgcover_codec_tests.nv` (35) takes the refusals and the encoders of
  `pgmsg`, `pgtype` and `pgerror`; `pgcover_pool_tests.nv` (22) drives
  the pool's three decisions against a caller-supplied clock and a
  scripted `PgConn`; `pgcover_api_tests.nv` (16) puts a whole exchange
  in a connection's inbox and checks the answer;
  `pgcover_conn_tests.nv` (13) is the file descriptor as a transport,
  including the four framing refusals and a `pgconn.connect` against a
  loopback listener nothing accepts on; `pgcover_driver_tests.nv` (8)
  pins the `std.sql` driver's class table and both answers to
  `close`.

### Changed

- `redundant-bool` sites rewritten to the `not` spelling: 17
  pre-existing ones across the three shipped suites
  (`pghost_tests.nv` six, `pgmsg_tests.nv` six, `pgtype_tests.nv`
  five), which were the whole of the interface's `== false` sites.
  The linter's own suggestion, and the predicate each assertion
  demands is unchanged.  `grep -rn '== false' src/ tests/` now answers
  nothing.
- `pgauth`'s five byte comparisons written as character literals:
  `printable_nonce`'s `c < '!' or c > '~' or c == ','` and
  `all_digits`'s `c < '0' or c > '9'`, where the hex `0x21`/`0x7E`/
  `0x2C`/`0x30`/`0x39` stood.  Also the linter's own `char-literal`
  suggestion, and what the language guide's example prescribes
  (`return b >= '0' and b <= '9'` … "not `b >= 0x30 and b <= 0x39`");
  a character literal is its byte value as an `Int`, so neither
  predicate changes and `novo.lint` now reports all nine modules
  clean.
- A TLS refusal names the mode it was made under.  `PgTlsRefused`'s
  detail is a sentence about the decision, not about the byte:
  `the server refused TLS, and PgSslRequire does not continue in the
  clear`.  A caller holding one mode among five cannot act on "TLS was
  refused" alone.

### Known

- **No PostgreSQL server was reachable on the machine this was
  written on**: no `postgres`, `psql`, `pg_isready` or `initdb` binary
  anywhere on the filesystem, nothing listening on 5432 or 5433, no
  `/var/run/postgresql` directory and no `/tmp/.s.PGSQL.*` socket, and
  no passwordless root to install one with.  The socket suites
  therefore pin the CLIENT'S half of the protocol — every byte it
  sends, against a fixture built out of raw bytes — and say nothing
  about a real server's behaviour.  The README says the same.
- **One suite exits 1 with every verdict green, and the object is the
  runtime's.**  `novo test --isolate
  tests/pgcover_conn_tests.nv` prints `13 passed, 0 failed`, then
  `novo test: the run leaked (exit 23 above)` and exits 1; the plain
  run's table is
  `tag 0 — 161199 allocs, 161198 reclaims, 1 leaked`.  The one object
  belongs to `novo_io_read_fd`: it allocates the Bytes the `read(2)`
  fills and answers `NULL` when the read fails without releasing it.
  `test_a_descriptor_that_refuses_the_write_or_the_read_is_a_lost_connection`
  drives exactly that arm on purpose — `pgconn.fd_recv` turns
  `io.read_fd`'s `None` into `Err(PgConnectionLost)` — so the leak is
  the test's whole subject and not a side effect of it.  Filed as
  `stdlib/io-read-fd-leaks-one-object-per-failed-read`, `fixed_in:
  0.20.3`; this release is built against 0.20.2, where it reproduces.
  The isolated run's line is itself filed — it points "above" at a
  report `--isolate` discards and cites a tag that run never prints:
  `test-infra/isolate-discards-the-leak-report-and-points-above-at-nothing`.
  Measured with `NOVO_CFLAGS=-DNOVO_RT_LEAK_SITES` and
  `NOVO_LEAK_SITE_TAG=0` on a binary this lane kept: one site,
  `rc==1` (a missing release), whose three frames resolve to
  `novo_io_read_fd` / `_bytes_alloc` / `novo_alloc_atomic`.  There is
  no workaround in this package's reach — `io.read_fd` is the only byte
  reader in `std.io`, and the readiness-first workaround the filing
  records cannot be used from `fd_recv` without turning a blocking read
  into a poll that a reset connection then leaks through anyway.  It is
  the process that fails, not a verdict.  The other twelve suites are
  leak-clean in the per-file isolated runs of the verification battery,
  and `pgwire_server_tests.nv` was run five more times isolated and once
  plain, clean every time.
- **The SCRAM nonce is the caller's.**  `fresh_nonce` is a clock
  reading, which is unique in practice and is not unpredictable; a
  caller who needs unpredictability passes one in
  `PgConnectOptions.nonce`.  The field's comment says why, and the
  reason is the effect row.
- **SASLprep is not performed.**  An ASCII password goes out exactly
  as typed, which is PostgreSQL's own first move; a non-ASCII one goes
  out as the bytes the user typed, without RFC 4013's mapping or NFKC
  steps.  A password the SERVER normalised will not match one sent
  from here.  `pgauth.saslprep`'s comment says which half this is.
- **`float4` keeps its value, not its width.**  A binary `float4` is
  widened to `Float` on decode (4 bytes of IEEE 754 binary32 are a
  width this package's `Float` is not) and a value is written back as
  `float8`; the round trip through this package is value-exact and not
  bit-exact.
- **Workarounds for six toolchain defects, each filed and cited at the
  site.**  Five are `fixed_in: 0.20.3` and reproduce on the 0.20.2 this
  release is built against; the sixth is still open.  The workaround
  stays until the toolchain that carries the fix is the one in use.
  - `pgauth.choose_mechanism` — a `list.get` read moved into a `Str`
    field of a variant in ANOTHER module answers the empty string, so
    the refusal that names an unsupported mechanism named nothing at
    all over the real `pgconn.connect` path.  The element is copied
    through `bytes.to_str(bytes.from_str(...))` first, which keeps it.
    Filed as
    `novo/a-list-get-into-a-variant-of-another-module-answers-an-empty-str`.
  - `pgconn.carry` — a `match`-bound `Result` payload carried into a
    tuple the handshake answers is freed with its scrutinee, which is
    a `SIGSEGV` at the SCRAM login.  Filed as
    `memory-perceus/a-match-bound-result-payload-carried-into-a-returned-tuple-is-freed-with-its-scrutinee`.
  - the `match` binder over `list.unzip` in `pgconn.connect` — a heap
    member read out of a `let`-bound runtime-built tuple is never
    released, four objects per login.  Filed as
    `memory-perceus/a-heap-member-read-out-of-a-let-bound-runtime-built-tuple-is-never-released`.
  - `Some(str.trim(body))` in `pgtype`'s array parser — a bound `Str`
    boxed inside a list literal's `Some(...)` reads back with a length
    of 1 and no bytes.  Filed as
    `memory-perceus/a-bound-str-boxed-in-a-some-inside-a-list-literal-reads-back-empty`.
  - every transport call in `pgconn` is read into a name before it is
    matched — a `match` written straight over the `Result` of a call
    through a fn-typed field leaks one cell per call.  Filed as
    `memory-perceus/a-match-that-rebuilds-the-result-of-a-call-through-a-fn-typed-struct-field-leaks`.
  - one `!` per `match` in `pgcopy.send_row`, and a bound `together`
    in `pgcopy.recv_rows` rather than an inline argument under a `!`
    — that shape strands one object per consumed chunk.  Filed as
    `memory-perceus/an-inline-argument-allocation-under-a-bang-unwrap-takes-no-scope-end-drop`;
    that filing's standalone reduction does not cover this shape (the
    small programs tried there were clean), so the measurement is what
    the site's comment records and not a cause.
- **Still holding from the interface release**: the `Database`
  contract's effect row is the widening that release documented
  (the trait declares `[io]`, this impl declares `[net, time]`); TLS
  is a hook the caller supplies and `PgSslMode` has no default because
  a server answering `N` has not failed; the pool is a value and does
  not synchronise, so a pool per cell is the shape that runs; and
  replication, multi-dimensional arrays, composite and range types and
  `async` entry points remain out of scope.

## 0.0.3 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- calendar-nv: `^0.0.2` to `^0.2.0`.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md).

`PgFault` now declares the `impl Error` its own `Result` positions
require.  `Result<T, E>` has carried the bound `E: Error` since SPEC
§ 3.4, and the compiler enforced it only when `E` was declared in the
module that named it — so `Result<_, pgerror.PgFault>` was accepted
across modules with no impl anywhere.  The impl is the signature this
package always meant; nothing else about the interface changed.

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
