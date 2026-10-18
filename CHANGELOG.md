# Changelog

All notable changes to Kamex (formerly Kannel) will be documented in this file.

## [1.9.8] - 2026-10-18

### Fixed
- **Release builds published Docker images and RPMs under the wrong version.**
  The release workflow read the version from `rpm/kamex.spec`, which was not
  bumped for 1.9.6 and 1.9.7, so those releases were published as 1.9.5 and
  overwrote the `1.9.5` image tag (#7). The workflow now reads `VERSION`, fails
  if the git tag differs from it, and fails the RPM build if any
  `rpm/*.spec` disagrees. The spec files, addon `configure.ac` files and README
  badge are brought up to date. No code changes.

## [1.9.7] - 2026-08-30

### Security
- **SQL injection in SQLBox on MySQL under a multibyte connection character
  set.** The 1.9.6 fix stopped backslash-escaping the quote but kept doubling
  backslashes by hand, and doubling is not character-set aware. In GBK, Big5,
  SJIS/CP932 and UHC a byte of `0x5C` can be the *trailing* byte of a
  character, so the server pairs the first backslash with the byte before it
  and the second one is left over to escape the closing quote — the rest of an
  SMS body then leaves the literal and is parsed as SQL. Confirmed against
  MariaDB with `character_set_server=gbk`: a sender field of `0xBF 0x5C` broke
  out of its literal and let a following field set another column's value, in a
  single statement, using no quote characters at all. UTF-8 and other
  single-byte connections were never affected.

  Values are now escaped with `mysql_real_escape_string()`, which reads the
  character set off the connection and steps over multibyte sequences properly.
  That means escaping has to happen against the same connection the statement
  runs on, so `mysql_save_msg()` and `mysql_save_list()` now take one pooled
  connection and use it for both. Statements are sent with
  `mysql_real_query()`, since under `NO_BACKSLASH_ESCAPES` a NUL byte survives
  escaping and would truncate a `strlen()`-measured statement mid-literal.
  Where the client library provides `mysql_real_escape_string_quote()`
  (libmysqlclient 5.7.6+, which otherwise refuses to escape at all in that
  mode) it is used instead; the choice is made by `configure`, not by version
  macros, because MariaDB Connector/C handles the mode inside the plain call
  and ships no `_quote`.

- **SQL injection in SQLBox on PostgreSQL with `standard_conforming_strings`
  off.** Noted as a residual in 1.9.6 and now fixed rather than documented.
  That setting is an ordinary GUC — still set per database, per role or in
  `postgresql.conf` by deployments carrying old application code — and with it
  off a backslash escapes again, so a value ending in one swallowed the closing
  quote. `PQexec` accepts several statements separated by semicolons, so this
  was arbitrary SQL. Values now go through `PQescapeLiteral()`, which consults
  both `standard_conforming_strings` and `client_encoding` and returns the
  literal already quoted. `pgsql_save_msg()` holds one connection for escaping
  and the statement, as on MySQL. SQLite and SQL Server were re-checked and
  need no change: neither has a backslash escape or a mode that introduces one,
  and `0x27` is not a valid trailing byte in any character set either supports.

- **Command injection in an `exec` sms-service whose pattern quotes the escape
  code.** 1.9.5 escaped substituted values by wrapping them in single quotes,
  which protects them only in bare text. Written as `exec = "/path/handler
  \"%S\""` — quoting the code is a natural thing to write — the added quotes
  land inside the operator's double quotes as ordinary characters, and `$(...)`,
  backticks and `${...}` in the message body still expand. Verified by running
  the generated command: an inbound `$(id)` executed.

  Substitution now tracks the quoting of the pattern itself and escapes each
  value for the context it lands in: wrapped in single quotes in bare text,
  `'\''` inside `'...'`, and backslash-escaped `$`, backtick, `"` and `\`
  inside `"..."`. `$(...)` and backticks start a fresh command context and are
  followed as such. This also fixes patterns that already single-quoted the
  code, where a message containing an apostrophe previously produced an
  unbalanced command line that the shell rejected outright.

  One case escaping cannot cover: a pattern that hands its command line to
  another shell (`sh -c ...`, `eval ...`) gets a second round of parsing, which
  strips the escaping again. Configuration is now scanned for that shape at
  start-up and warns, naming the pattern.

### Fixed
- **Delivery-report URLs were shell-quoted for `exec` sms-services.** A service
  configured with both `exec` and `dlr-url` picked its escaping from the
  service type while smsbox services every delivery report as
  `TRANSTYPE_GET_URL`, so values went into the callback URL wrapped in single
  quotes and with `&`, `=` and spaces left unencoded — letting a message body
  add parameters to the DLR callback. Delivery reports now always use URL
  encoding, whatever the service type. Found while verifying the 1.9.5 fix.
- SQLBox on PostgreSQL leaked every `PGresult` from an update statement.
- `mysql_update()` consulted `mysql_errno()` after every statement, including
  successful ones, because of a missing pair of braces.

### Added
- `checks/check_urltrans` covers escape-code substitution for both command
  lines and URLs. Neither the 1.9.5 nor the 1.9.6 fix shipped a regression
  test, which is how the two above went unnoticed.

## [1.9.6] - 2026-08-30

### Security
- **SQL injection in SQLBox on PostgreSQL, SQLite and SQL Server.** The value
  quoting used MySQL's convention on every backend — escaping a single quote as
  `\'`. A backslash is not an escape character inside a string literal on
  PostgreSQL with `standard_conforming_strings` on (the default since 9.1), nor
  on SQLite or SQL Server, so the quote stayed live and closed the literal:
  the remainder of an SMS body was parsed as SQL. The PostgreSQL driver issues
  statements through `PQexec`, which accepts several statements separated by
  semicolons, so this was arbitrary SQL rather than a malformed insert.

  Values are now quoted the SQL-standard way, by doubling the quote. Backslashes
  are left untouched on those three backends, where doubling them would have
  corrupted stored text; MySQL keeps backslash doubling, since it does treat a
  backslash as an escape unless `NO_BACKSLASH_ESCAPES` is set. Oracle was never
  affected — it passes values as bind variables.

  Residual note: on PostgreSQL with `standard_conforming_strings` explicitly
  turned *off* — a deprecated, non-default setting — a value ending in a
  backslash can still escape the closing quote. Use the default.

  Found by **Nika Archvadze** in the same audit as the `exec` command injection
  fixed in 1.9.5.

## [1.9.5] - 2026-08-29

### Security
- **Remote code execution via `%S` (and `%C`) in an `exec` sms-service.**
  `urltrans_fill_escape_codes_ex()` escaped substituted values for
  `TRANSTYPE_EXECUTE`, but `%S` and `%C` appended their value to the command
  line without going through the escape. `%S` is a word taken straight from the
  inbound message body, and the finished string is handed to `popen()`, so for
  any deployment whose `exec` pattern uses `%S` an inbound SMS containing
  `$(...)`, backticks or a semicolon executed arbitrary commands as the smsbox
  user.

  Both now shell-escape when the result is a command line. URL substitution is
  unchanged: `%S` stays deliberately un-encoded there — that is its whole
  difference from `%s` — and still maps `*` to `~`.

  Verified by executing the generated command: `$(id)`, backticks, `;id;` and
  an embedded-single-quote payload are all passed through literally and none
  execute. Deployments not using `exec` sms-services, or not using `%S`/`%C` in
  one, were never exposed.

  Found and reported with a working proof of concept by **Nika Archvadze** —
  thank you.

## [1.9.4] - 2026-08-29

### Fixed
- **Bearerbox started with no admin interface when the admin port could not be
  bound.** `httpadmin_start()` discarded the return value of
  `http_open_port_if()`, so a port already in use, an `admin-interface` that
  does not resolve, or a privileged port without permission left the gateway
  running with no admin port, no `/health` and no `/metrics` — while logging
  "Admin panel available at / and /admin". Binding the admin port is now fatal,
  with a message naming the likely causes.

### Changed
- **systemd: `ProtectSystem` relaxed from `strict` to `yes`.** `strict` remounts
  almost the entire filesystem read-only and has been reported to stop the
  service starting; `yes` keeps `/usr` and `/boot` read-only, and the existing
  `ReadWritePaths` still scopes what the daemon may write.
- **systemd: the config is validated before the daemon starts.** The bearerbox
  and smsbox units gained `ExecStartPre=... -t /etc/kamex/kamex.conf`, so a
  configuration that cannot start fails the unit with a clear reason instead of
  a restart loop. Since 1.8.5 that check applies the same mandatory-group rules
  as start-up, so it catches more than syntax. It cannot catch a port that is
  already bound — the fix above covers that.
- **systemd: units log to the journal** (`StandardOutput`/`StandardError`), so
  `journalctl -u kamex-bearerbox` shows start-up failures that previously went
  nowhere.

  These three came from reviewing the omaxdevelop/kamex fork, which is a copy of
  1.8.3 that had hit them in deployment.

## [1.9.3] - 2026-08-29

### Fixed
- **Concatenated inbound messages were not reassembled when their parts carried
  differing UDH elements.** The reassembly key included the residual UDH — the
  information elements left after the concatenation header is removed — so two
  parts of one message that differed there hashed to different buckets. Neither
  set ever completed, and each part was eventually delivered on its own once the
  reassembly timeout expired.

  The application therefore received fragments instead of one message, delayed
  by the whole of `sms-combine-concatenated-mo-timeout` — half an hour at the
  default. Parts are now keyed on sender, receiver, SMSC, reference number and
  part count, which is everything the concatenation header actually promises the
  parts share. The reassembled message takes the first part's residual UDH; it
  already took one part's and discarded the rest, but which one depended on
  arrival order.

  Reported to the Kannel devel list in April 2020 and never answered.

## [1.9.2] - 2026-08-29

### Added
- **`access-log-utf8` in the `core` group** logs message bodies as readable
  UTF-8 instead of hex or a row of dots. The access log renders bodies with
  `isprint()`, so a Georgian or Cyrillic SMS became dots and a UCS-2 body became
  hex; with this set, UCS-2 is converted to UTF-8 and text passes through a
  UTF-8 aware printable filter. Binary bodies are still hex, and a failed
  conversion falls back to hex rather than logging raw UTF-16BE, which begins
  with a NUL for ASCII text and would truncate the field to nothing. Off by
  default; the log is unchanged until it is set.

  Control characters are filtered on this path, which the old `isprint()` route
  also did — so a message body still cannot inject a line break into the log and
  forge a record, nor pass terminal escape sequences to whoever tails it.

  Inspired by a patch Stipe Tolj posted to the Kannel devel list in May 2023,
  but not derived from it. That patch decoded to `wchar_t` and added an
  unconditional process-wide `setlocale(LC_ALL, "")` in `octstr_init()`, which
  would have applied to every Kamex process whether the feature was enabled or
  not. On a host with a comma decimal separator that silently breaks the
  Prometheus `/metrics` output, the JSON admin endpoints, and `throughput`
  parsing — where `0.5` becomes `0.0`, and zero means *no limit*, so a
  rate-limited SMSC would quietly become unthrottled. Its wide-character
  accessor also returned string literals on two paths and heap memory on a
  third, and the caller freed the result unconditionally: in the C locale that a
  daemon actually runs in, a single inbound message containing one byte above
  0x7F aborted bearerbox. This implementation stays in UTF-8 octets, touches no
  locale, and filters control characters that the original passed through
  untouched on the common text path.

## [1.9.1] - 2026-08-29

### Added
- **`sms-combine-concatenated-mo` and its timeout can now be set per SMSC**,
  falling back to the `core` value when omitted. Inbound fragmentation is a
  property of the operator rather than of the gateway: a timeout long enough for
  a slow route pins incomplete fragments from *every* route in memory for that
  long, while a short one drops parts the slow route would have completed. Either
  directive can be overridden on its own.

  Adapted from a patch Stipe Tolj posted to the Kannel devel list in February
  2016, which was never committed. It carries the resolved timeout in the
  reassembly record so the periodic sweep can honour a per-connection deadline
  without looking the connection up. One fix was needed: the patch read
  `conn->handle_concatenated_mo` at a call site that is reached with no
  connection at all — the NACK path passes NULL, as the guard on the line below
  it shows — which would have dereferenced NULL on the first message. It now
  falls back to the core values there.

## [1.9.0] - 2026-08-29

### Added
- **`dlr-smsc-id` in the `smsc` group**, letting several connections share one
  delivery-report namespace. When an operator is reached over more than one bind
  — a primary and a secondary on different hosts or ports — a message submitted
  on one bind can have its receipt returned on another. Receipts are stored and
  found under the connection's `smsc-id`, so one arriving on a sibling bind
  matched nothing and was dropped, leaving only a `DLR ... not found` warning.

  Give the siblings a shared `dlr-smsc-id` and their receipts go in one
  namespace, while `smsc-id` stays distinct for routing, logging and status.
  Omitted, a connection uses its own `smsc-id` exactly as before. Applies to the
  SMPP, EMI, AT, fake and loopback drivers.

  The problem was reported to the Kannel devel list in June 2019 by an engineer
  at Vodafone Automotive, running it in production at the time. The patch he
  posted took a different route — storing receipts under `conn->name` and
  redefining that name to drop host and port — which was not adopted here:
  `conn->name` is also the fallback `smsc-id` for connections that do not set
  one, and collapsing it would have silently changed their identity and made any
  two connections sharing an `smsc-username` match each other's receipts. An
  explicit directive was the design he himself argued for when the patch was
  reviewed; that discussion never concluded and upstream has not moved since.

## [1.8.11] - 2026-08-29

### Fixed
- **OpenSMPPBox tore down the wrong session on a duplicate bind.** The
  duplicate-bind check compared the incoming `system_type` against a live
  session's `boxc_id`, which is wrong in both configurations:
  - By default `boxc_id` *is* the `system_type` — free text that is routinely
    `"SMPP"` or `"VMS"` for every account on the box. One customer rebinding
    therefore disconnected an unrelated customer's live session.
  - With `use-systemid-as-smsboxid = true`, `boxc_id` is the `system_id`, so the
    check put two different fields against each other and matched only by
    coincidence.

  A bind is now matched against the identity the session is actually labelled
  with, so a second bind displaces only its own earlier session.

  **Note for deployments running `use-systemid-as-smsboxid = true`:** duplicate
  binds were effectively never detected there, so a client holding two
  simultaneous sessions of the same type under one account was tolerated. That
  now behaves as intended and the earlier session is disconnected. If you rely
  on multiple concurrent binds per account, this will disconnect them — a
  configurable ceiling is being worked on separately (#6).

## [1.8.10] - 2026-08-29

### Added
- **`throughput-mt` and `throughput-mo` in the `smsc` group**, splitting the
  existing rate limit into separate MT and MO ceilings. Previously `throughput`
  applied only to messages sent *to* the SMSC, so there was no way to limit how
  fast an SMSC could deliver MO traffic and DLRs *to* the gateway. A bare
  `throughput` still sets both, so existing configurations are unchanged.

  Implemented for the SMPP, HTTP, EMI, AT and fake drivers. When the MO ceiling
  is reached, SMPP replies to `deliver_sm`/`data_sm` with `ESME_RTHROTTLED` so a
  well-behaved SMSC backs off and retries.

  Adapted from a patch posted to the Kannel devel list in July 2022 by Stipe
  Tolj, the upstream maintainer, which was reviewed once and never committed;
  upstream trunk has not moved since May 2024. Two changes were needed: the
  throttle path returned `ESME_RX_T_APPN`, an application-error code rather than
  a throttling one — the point raised in the original review and never addressed
  — and the hunk for the SMASI driver was dropped, that driver having been
  removed in 1.7.5.

## [1.8.9] - 2026-08-06

### Fixed
- **IP access rules with fewer than four parts silently matched nothing.** A
  trailing wildcard advanced only as far as the next dot and then required the
  address to be exhausted, so any pattern that did not have exactly four
  dot-separated components matched no IPv4 address at all — and said nothing
  about it. `box-deny-ip = "192.168.*"` denied no one, `admin-deny-ip = "10.*"`
  denied no one, and a deny list of `"*"` denied no one. In the deny direction
  this failed **open**: a rule written to lock an interface down had no effect,
  with nothing logged and a configuration that parsed cleanly. A trailing
  wildcard now covers the rest of the address, so these patterns match what
  they look like they match. (#5)

  **This changes what some existing configurations do.** A rule that already
  matched is unaffected — the change only widens a wildcard that previously had
  to land exactly on the end of the address, so nothing that was being denied
  stops being denied. But a rule that was inert now takes effect:

  - in a **deny** list it starts denying the addresses it names, which is what
    it was written to do;
  - in an **allow** list it starts admitting them. An allow entry that matched
    nothing was previously refusing the very peers it was meant to permit, so a
    working deployment is unlikely to depend on one — but check any
    `*-allow-ip` with fewer than four parts before upgrading.

  Wildcards that are not trailing are unchanged and still cover a single
  component, so `10.*.0.5` continues to match `10.9.0.5` and not `10.9.9.0.5`.
  Exact patterns are unchanged: `127.0.0.1` still does not match `127.0.0.10`.

## [1.8.8] - 2026-08-06

### Added
- **IPv6 support, opt-in via `ipv6 = true` in the `core` group.** Off by default,
  so an existing deployment behaves exactly as before until it is turned on. The
  setting gates both ends: with it off the wildcard listener binds `AF_INET` and
  outbound resolution asks for `AF_INET` only, which is indistinguishable from
  the previous behaviour. Read once at startup by bearerbox, smsbox, OpenSMPPBox
  and RabbitMQBox via the new `socket_enable_ipv6()`.
  - SQLBox takes the same setting from its **`sqlbox` group** rather than from
    `core`, because it reads its own configuration file and that schema
    (`sqlbox-cfg.def`) has no core group at all. Without it SQLBox would stay
    IPv4-only and become the one single-stack hop in a
    smsbox -> SQLBox -> bearerbox chain.
  - Name resolution uses `getaddrinfo()` instead of `gethostbyname()`, so AAAA
    records and literal IPv6 addresses resolve; previously any IPv6-only host
    failed with `gethostbyname failed`.
  - Outgoing connections try every address the resolver returns, across both
    families. The socket is created per candidate, since the family is only known
    after resolution.
  - With `ipv6 = true` the wildcard listener binds the IPv6 wildcard with
    `IPV6_V6ONLY` cleared so one socket serves both families, falling back to
    IPv4 if that cannot be set up. A *named* interface still resolves IPv4 first,
    matching what `gethostbyname()` returned, because one port is one socket:
    `admin-interface = "localhost"` keeps serving IPv4 clients. An IPv6-only name
    yields an IPv6 listener.
  - Bracketed IPv6 literals in URLs (`http://[::1]:8080/x`) now parse. The port
    separator is located after the closing bracket, which must precede the path,
    and the brackets are stripped before the host reaches `getaddrinfo()`.
  - New `http_host_for_url()` re-brackets a host on the way out, so a `Host:`
    header and any rebuilt URL are RFC 3986 sec. 3.2.2 / RFC 7230 sec. 5.4
    conformant.
  - New `checks/check_sockaddr` and `checks/check_url` regression tests.

### Fixed
- **Truncated peer addresses on `accept()`.** `gw_accept()` and the listeners in
  bearerbox, smsbox, the fake and EMI SMSC drivers, SQLBox, OpenSMPPBox and
  `test/drive_smpp` received peer addresses into a `struct sockaddr_in` (or a bare
  `struct sockaddr`), which is too small for an IPv6 peer. They now use
  `struct sockaddr_storage`. Latent while every listener was IPv4-only.
- `errno` is captured immediately after `connect()` in the non-blocking path.
  `freeaddrinfo()`, `error()` and `octstr_destroy()` ran before it was tested and
  none of them promise to preserve it, so a clobbered value could misread a
  pending connect as a failure.
- A wildcard listener that cannot clear `IPV6_V6ONLY` now gives up that candidate
  and falls back to IPv4 instead of warning and binding IPv6-only. On platforms
  where dual-stack sockets are forbidden this would otherwise have silently
  stopped serving IPv4.
- Per-candidate connect failures log at warning level while further candidates
  remain, so a dual-stack destination that works on the second address no longer
  logs an error for the first.
- **Every address after the first was dead code for a multi-homed host, IPv4
  included.** The old loop reused one socket across candidates, and on Linux a
  `connect()` retried on a socket whose previous attempt failed returns an error
  unconditionally - so a host with several A records only ever really tried the
  first. Creating the socket per candidate was needed for IPv6, where consecutive
  candidates differ in family, but it repairs the IPv4 case as a side effect: a
  multi-A host whose first address is down can now succeed on a later one.
- `parse_url()` refuses trailing garbage after an IPv6 literal. Anything between
  the closing bracket and the port, path, query or fragment is rejected, so
  `http://[::1]extra/foo` no longer parses with a host of `[::1]extra`, and
  `http://[::1?foo]bar` no longer parses with a host of `[::1` and emits an
  unbalanced `Host:` header. Both are refused without brackets, and accepting a
  bracketed literal must not be the looser path. `http://[::1]@evil.com/`, which
  previously tripped an assertion in `octstr_copy()`, is refused by the same
  check.

### Changed
- **Two log levels moved, visible on an IPv4-only system too.** A connect failure
  is logged at `warning` rather than `error` while another candidate remains -
  only the last one is an error - and a listener whose name fails to resolve in
  one family logs at `debug` rather than `error`, since the caller may still have
  another family to try. A genuinely unusable `admin-interface` still produces an
  error. Anyone scraping logs for `ERROR:` should know these moved.
- `gw_sockaddr_to_octstr()` renders IPv4-mapped addresses as plain IPv4, so a
  dual-stack listener describes an IPv4 peer the familiar way and IPv4-style
  ACLs keep matching. It never returns NULL: the result is stored on connection
  objects and logged widely, and an unrecognised address family yields a
  placeholder rather than a NULL surfacing far from the call site.

### Access control, and what the dual-stack change requires
These are consequences of making listeners dual-stack, not fixes to a
pre-existing hole: with `ipv6` off - the default - every listener is `AF_INET`
only, no IPv6 peer can connect, and none of the below applies. The matcher
consults the same setting, so with IPv6 off an address is matched exactly as it
was before this change; the rules below take effect only once it is turned on.
- `*` and `*.*.*.*` are treated as "any address" in both families, so a
  deny-everything rule denies IPv6 peers too. **Only those two exact forms.** A
  partial wildcard such as `*.*` still matches no real address, exactly as
  before - treating it as universal would have turned an inert allow-list entry
  into allow-all, for IPv4 as well.
  Note the flip side: an *allow* rule written `*.*.*.*` now admits IPv6 peers.
  A deny list that enumerates IPv4 subnets, e.g. `box-deny-ip = "10.0.0.*"` with
  no allow list, matches no IPv6 peer and therefore admits it. Review such rules
  before enabling `ipv6`.
- A loopback rule in one family covers the other family's loopback, because
  `getaddrinfo()` returns `::1` before `127.0.0.1` for `localhost`: smsbox
  reaches bearerbox over IPv6 and a dual-stack listener sees `::1`, so
  `box-allow-ip = "127.0.0.1"` would otherwise start rejecting smsbox. The IPv4
  side is a prefix test, so a rule naming `::1` accepts any `127.x.y.z` peer and
  vice versa; both are loopback. Within one family the normal globbing applies -
  `127.0.0.1` still does not match `127.0.0.2`.
- `connect_denied()`'s implicit localhost-only default covers `::1` as well as
  `127.0.0.1`.
- Also expect local peers to appear as `::1` in logs, status pages and access
  logs once `ipv6` is on; log parsers expecting dotted quads need adjusting.
- A multi-host setup whose bearerbox hostname has an AAAA record will see smsbox
  arrive from the IPv6 address, which an IPv4-subnet `box-allow-ip` will not
  match. There is no shim for that case: list the IPv6 address, or leave `ipv6`
  off.

### Known limitations
- A non-blocking connect cannot fall back after `EINPROGRESS`: the caller polls
  for completion, so if that candidate - typically the IPv6 one - fails
  asynchronously, the remaining addresses are never tried. On a host whose IPv6
  route is black-holed this makes an outbound connection fail where an IPv4-only
  build succeeded. `ipv6 = false` is the remedy; retrying across the deferred
  failure needs the candidate list to outlive the call and is left for later.
- URLs combining userinfo with an IPv6 literal are rejected -
  `http://user@[::1]/x` as well as `http://user:pass@[::1]/x`. Unchanged from
  before this work, and fail-closed. Note `http://[::1]@evil.com/` is a
  different shape: it used to trip an assertion inside `octstr_copy()` rather
  than being refused, and is now rejected by the trailing-garbage check below.
- Matching in the access lists is textual: `0:0:0:0:0:0:0:1` is not recognised as
  `::1`, and hex case is significant.
- A bare `*` in an access list has never matched a dotted quad - the matcher
  advances a wildcard only as far as the next dot - so a deny list of `*` still
  denies nobody, in either mode. That is unchanged here, and deliberately so:
  making it universal in the IPv4 case too would alter what existing
  configurations mean. Write `*.*.*.*`, which does match.
- Not covered, and unchanged by this work: the UDP helpers (`udp_bind()`,
  `udp_create_address()` and friends) are still IPv4-only and still resolve with
  `gethostbyname()`. Nothing calls them since the WAP and RADIUS removal - only
  `test/test_udp` does, and it passes an explicit `0.0.0.0`.
- Also not covered: RabbitMQBox reaches its broker through librabbitmq's own
  `amqp_socket_t`, not through gwlib, so the `ipv6` setting does not govern that
  connection. It governs RabbitMQBox's link to bearerbox, which does use gwlib.
- The official hostname and IP shown in the startup banner are resolved from
  `gwlib_init()`, before any configuration has been read, so they are always
  looked up over IPv4. On an IPv6-only host the banner still reports
  `127.0.0.1`. Cosmetic: these two values are not used for anything else.

## [1.8.7] - 2026-08-05

### Fixed
- **SQLBox spun forever on a message it could not process, filling the disk.**
  In list mode, a message failing `charset_processing()` was destroyed before
  it reached the save list — and the save list is what deletes the row. The row
  stayed in the queue table, was selected again on the next pass, and since that
  branch has no sleep, this was a tight loop: measured at ~2,500 log lines per
  second, with two MySQL syntax errors per cycle (an empty batch produced
  `VALUES ` and `WHERE sql_id in ()`). Failed messages are now handed to the
  save list so their row is removed, and an empty batch issues no statements.

  This is the default path: list mode is selected whenever `limit-per-cycle`
  is above 1, which it is by default (10), and MySQL is the only backend that
  implements it. A single message containing a literal `%` was enough to
  trigger it, because `msgdata` is stored URL-encoded.
- **SQLBox reported the failure without saying what was wrong.** "Could not
  charset process message, dropping it!" named neither the row nor the reason.
  It now reports the `sql_id` and receiver, and distinguishes invalid URL
  encoding (with a hint that a literal `%` must be written `%25`) from a failed
  charset conversion, naming the charset.
- **`-t` accepted configurations that then refused to boot.** Config testing
  stopped after the syntax and field-name pass, so a config missing a group the
  daemon requires — `smsbox-port` set with no `smsbox` group, for instance —
  reported "test is successful" and exited 0, then panicked on a real start.
  Both `bearerbox -t` and `smsbox -t` now apply the same mandatory-group checks
  the startup path applies, print `test failed`, and exit non-zero, which is
  what makes them usable in CI. Configurations that were already valid are
  unaffected.
- The "DLR not found" warning now includes the SMSC timestamp. Without it the
  message could not be correlated with a submission at default log level, since
  only the debug line above it carried the field.

## [1.8.6] - 2026-08-05

### Added
- **SNI on outbound TLS connections.** The hostname was passed into
  `conn_open_ssl()` and then discarded, so no `server_name` extension was sent.
  Any server that selects a certificate by name — anything behind Cloudflare or
  sharing an address between virtual hosts — either failed the handshake or
  served the wrong certificate. SNI is now sent for host names and, per RFC 6066,
  omitted for literal addresses.
- **`ssl-verify-host` (core and sqlbox groups, default `false`).** When enabled,
  a server Kamex connects to must present a certificate issued by a trusted CA
  *and* issued for the host that was configured. Literal addresses are matched
  against the certificate's IP SANs. Enabling it implies peer verification,
  falling back to the system CA store when `ssl-trusted-ca-file` is unset.

  Without it, outbound TLS remains unauthenticated: peer verification is off
  unless a CA bundle is configured, and even then **any** valid certificate from
  **any** host was accepted, because nothing compared the certificate to the host
  requested. That affects SMPP over TLS, `dlr-url` callbacks and `sms-service`
  fetches. It defaults to `false` only because turning it on can stop a gateway
  from reaching an SMSC using a self-signed or name-mismatched certificate;
  operators are encouraged to enable it. See `doc/configuration.md`.
- `test/test_http` gained `-N` to require certificate/host matching, so the
  behaviour above can be exercised directly.

## [1.8.5] - 2026-08-05

Fixes mined from the upstream Kannel mailing list archives and Debian's patch
queue. Upstream trunk has been frozen at r5336 (2024-05-14) since before Kamex
forked, so none of these would ever have been fixed there.

### Fixed
- **SMPP delivery receipts with padded fields were parsed wrong, and reported a
  fabricated error code.** The receipt parser required `stat:` and `err:` values
  to follow the label with no space, so a real-world receipt of the form
  `... stat:UNDELIV err: 042 ...` failed the strict parse and fell back to the
  lenient path. That path never assigned the network error code, so **every**
  fallback-parsed receipt reported error `0` regardless of what the SMSC sent.
  Both are fixed: the parser now tolerates whitespace after the label, and the
  fallback path parses the error code it extracted.
- **SMPP receipt status matching is now case-insensitive.** A receipt carrying
  `stat:delivrd` parsed correctly and was then silently reclassified as a
  failure.
- **`reply-couldnotrepresent` was read from the wrong configuration key.** It was
  loaded from `reply-couldnotfetch`, so setting it had no effect while
  `reply-couldnotfetch` silently overrode both messages. Because the correct name
  is declared in the config schema, `-t` validation accepted it and gave no hint.
  Inherited from Kannel; never reported upstream.
- **OpenSMPPBox reported expired messages as accepted.** A missing `break` made
  `DLR_EXPIRED` fall through to `DLR_BUFFERED`, so every expired receipt went out
  as `stat:ACCEPTD` (state 6) instead of `stat:EXPIRED` (state 3). Upstream Kannel
  has the same defect.
- **Build failure on 32-bit time64 targets.** `send_enquire_link()` took a
  `long *` while its caller passed a `time_t *`. Harmless on 64-bit, but an error
  under GCC 14+ on platforms such as Debian i386/armhf, where it would also have
  truncated the enquire-link timer.

### Changed
- `make check` now exits non-zero when a check fails and prints `check.log` to
  stdout. Previously it announced the failure and exited 0, so failing tests
  could pass unnoticed in CI.
- Removed the unused `<openssl/engine.h>` include and `CRYPTO_CALLBACK_PTR`
  typedef; the ENGINE header is gone in OpenSSL 4.0.
- `dbpool_sqlite3` reports the runtime library version via `sqlite3_libversion()`
  instead of the `sqlite3_version` symbol.
- `checks/check.log.new` is no longer tracked; the check runner deleted it on
  every run, leaving the working tree permanently dirty.

## [1.8.4] - 2026-08-05

### Fixed
- **SMPP binds failed with passwords longer than 8 characters** - the SMPP 3.4
  spec caps the bind `password` field at 9 octets, but many commercial operators
  use credentials up to 16 characters. Outbound binds silently truncated the
  configured password to 8 characters (the operator then rejected the bind), and
  inbound binds to OpenSMPPBox with a long password failed with misparsed PDU
  fields. The field limit in the bind PDUs (`bind_transmitter`, `bind_receiver`,
  `bind_transceiver`, `outbind`) is now 17 octets, allowing 16-character
  passwords in both directions. Operators using spec-compliant passwords are
  unaffected. (#1)
- **PostgreSQL DLR storage silently stopped working on PostgreSQL 12+** - the
  driver located single rows with `WHERE oid = (SELECT oid ...)`, but user tables
  have not carried an `oid` system column since PostgreSQL 12 removed
  `default_with_oids`. Both statements failed with
  `column "oid" does not exist`, so:
  - consumed DLRs were never deleted, and the table grew by one row per message
    for the lifetime of the deployment
  - intermediate DLR status updates were never applied
  Delivery receipts still reached the application, which is why this went
  unnoticed. Both statements now use `ctid`, which every server version provides
  and which gives the same "exactly one matching row" guarantee within a single
  statement.

## [1.8.1] - 2026-01-18

### Added
- **Prometheus /metrics endpoint** - Native Prometheus monitoring support
  - Counters: `kamex_sms_sent_total`, `kamex_sms_received_total`, `kamex_dlr_*_total`
  - Gauges: `kamex_uptime_seconds`, `kamex_smsc_online`, `kamex_sms_queue_*`
  - Rates: `kamex_sms_sent_rate`, `kamex_sms_received_rate` (per second)
  - Log queue metrics: `kamex_log_queue_depth`, `kamex_log_dropped_total`
  - No authentication required (standard for metrics endpoints)
- **OpenAPI specification** - Complete API documentation in `doc/openapi.yaml`
  - Admin API endpoints (monitoring, control, SMSC management)
  - SMS API endpoints (sendsms with all parameters)
  - Compatible with Swagger UI and code generators
- **Reproducible builds** - Enterprise-grade build verification and compliance
  - Supports `SOURCE_DATE_EPOCH` for deterministic timestamps
  - `--enable-reproducible` configure flag (auto-enabled with SOURCE_DATE_EPOCH)
  - Strips absolute paths from binaries with `-ffile-prefix-map`
  - Identical SHA256 hashes for same source + environment
  - Docker images: pinned base image digest and EPEL version
  - GitHub Actions CI sets `SOURCE_DATE_EPOCH` automatically
  - Addons (SQLBox, OpenSMPPBox) support reproducible builds
- **Config validation** - Validate configuration files without starting services (nginx-style)
  - `bearerbox -t /etc/kamex/kamex.conf` - test bearerbox config
  - `smsbox -t /etc/kamex/kamex.conf` - test smsbox config
  - Clean output: `bearerbox: configuration file ... test is successful`
  - Returns exit code 0 on success, 1 on failure
  - Useful for CI/CD pipelines and deployment automation
- **Command-line help** - `bearerbox --help` and `smsbox --help` now work
  - Full usage information with all options
  - Examples for common operations
  - See `doc/cli.md` for complete documentation
- **Structured JSON logging** - Machine-readable log format for log aggregation
  - Enable with `log-format = json` in core or smsbox groups
  - Output: `{"ts":"...","level":"info","pid":123,"tid":0,"msg":"..."}`
  - Compatible with ELK, Loki, Splunk, Fluentd, and other log systems
  - See `doc/logging.md` for examples with jq
- **Environment variable expansion** - Reference env vars in config with `${VAR}` syntax
  - `admin-password = ${ADMIN_PASSWORD}` expands from environment
  - Mixed content supported: `host = smsc.${ENV}.example.com`
  - Enables Docker/K8s secrets injection without config changes
  - See `doc/configuration.md` for usage examples
- **SBOM** - CycloneDX 1.7 Software Bill of Materials (`sbom.json`)
  - Runtime dependencies with exact versions from UBI 10
  - Package URLs (PURLs) for vulnerability scanning
  - For security compliance and supply chain verification

## [1.8.0] - 2026-01-12

### Added
- **Async logging** - Log messages are now queued and written by a dedicated writer thread
  - Bounded queue (128K entries, ~512MB max) prevents unbounded growth
  - Calling threads no longer block on I/O - ~10x throughput improvement
  - PANIC level remains synchronous (crash context must hit disk immediately)
  - Per-SMSC exclusive logging preserved via `exclusive_idx` routing
  - 4KB buffer per entry handles 9-segment SMS in hex logs
- **Logging observability** - New monitoring endpoints for log queue health
  - `/health` returns `warn` status when queue >= 80% or messages dropped
  - `/status.json` includes `logging` section with queue depth, dropped count, writer status
- **Architecture documentation** - `doc/logging.md` explains async logging design
- **RPM logrotate** - Logrotate config now included in RPM package

### Fixed
- **Async logging security** - Fixed multiple issues found during security audit:
  - Race condition: capture `log_queue` to local variable before use
  - Memory leak: use `gw_native_free` destructor in `gwlist_destroy`
  - Out-of-bounds: validate `exclusive_idx < num_logfiles` before array access
  - Shutdown race: set `log_queue = NULL` before destroying queue
- **fakesmsc installation** - Now installs real binary instead of libtool wrapper
- **test_headers.c** - Removed WAP/WSP dependencies, now tests HTTP headers only
- **check_sendsms.sh** - Fixed incorrect path and cumulative auth failure count
- **check_headers.sh** - Updated for simplified test_headers
- **run-checks** - Now checks exit codes instead of treating any stderr as failure

### Changed
- Log writer thread uses `gwthread_create()` for proper gwlib integration
- `LogQueueStatus` struct added to `gwlib/log.h` for queue monitoring
- Queue size reduced from 512K to 128K entries (still handles sustained bursts)

## [1.7.8] - 2026-01-12

### Added
- **OpenSMPPBox packaging** - RPM package for kamex-opensmppbox addon
- **OpenSMPPBox systemd service** - `kamex-opensmppbox.service` with security hardening

### Changed
- Modernized OpenSMPPBox configure.ac, removed DocBook build system
- GitHub workflow now builds all 3 packages: kamex, kamex-sqlbox, kamex-opensmppbox

## [1.7.7] - 2026-01-12

### Removed
- **SQLite2 support** - Removed obsolete SQLite 2.x database backend (use SQLite3)
- **libsdb support** - Removed dead libsdb database abstraction library
- Removed ~500 lines of dead code from gwlib, gw, and sqlbox

### Changed
- Cleaned up database pool enum and initialization code
- Updated test_dbpool.c to remove SQLite2 tests

## [1.7.6] - 2026-01-12

### Added
- **SQLBox packaging** - RPM package for kamex-sqlbox addon
- **SQLBox systemd service** - `kamex-sqlbox.service` with security hardening

### Changed
- **Systemd services** - Use `RuntimeDirectory`, `StateDirectory`, `LogsDirectory` for better compatibility
- **Systemd paths** - Service files now use `@SBINDIR@` template for correct paths in both `make install` and RPM

### Fixed
- **Namespace errors** - Fixed `status=226/NAMESPACE` errors in containers/VMs
- **SQLBox build** - Modernized configure.ac, removed DocBook build system

## [1.7.5] - 2026-01-10

### Rebrand
- **Renamed from Kannel to Kamex** due to licensing restrictions
- New MIT license for Kamex code, original Kannel code remains under Kannel Software License 1.0
- Configuration files remain compatible with Kannel
- Systemd service files renamed to `kamex-bearerbox`, `kamex-smsbox`
- Paths changed to `/etc/kamex`, `/var/log/kamex`, etc.

### Added
- **Web Admin Panel** - Built-in dashboard at `/` and `/admin` with real-time monitoring
  - Dashboard with SMS/DLR traffic stats and SMSC status
  - Queue viewer showing pending messages from store-status
  - Send SMS form for testing
  - Gateway controls (suspend/resume/shutdown/restart SMSCs)
  - Auto-refresh toggle (5s/15s/30s/Off)
  - Admin mode vs view-only mode detection
- **JSON API** - Modern REST-like endpoints
  - `/api/sendsms` - POST-only JSON endpoint for sending SMS
  - `/status.json` - JSON status output with rates and SMSC details
  - Token authentication via `X-API-Key` header and `api-token` config
- **Health Check** - `/health` endpoint for load balancers and Kubernetes
- CORS headers for smsbox sendsms endpoint

### Removed
- **libxml2 dependency** - No longer required
- **WAP/WML support** - Removed all WAP-related code and files
- **RADIUS support** - Removed RADIUS authentication
- Legacy platform support (Solaris, Interix3, FreeBSD c_r)
- SVN/CVS artifacts and dead code

### Changed
- Admin panel HTML embedded in binary (no external file needed)
- OpenSSL 1.1+ thread safety test skipped (always thread-safe)
- Modernized autoconf configuration

### Fixed
- JSON SMSC status comma handling for multiple SMSCs
- OpenSSL auto-detection for modern distros
- iconv library detection for Linux systems

## [1.6.5] - 2025-12-01

### Added
- Unix socket support for Redis connections
- Systemd service files with security hardening
- Logrotate configuration
- Performance benchmarks
- GitHub-friendly README.md and markdown documentation

### Changed
- Updated build dependencies for Fedora/EL10
- Replaced bootstrap.sh with standard autoreconf

### Fixed
- OpenSSL detection for modern distros
- gettext m4 macros
- Benchmark scripts

## [1.6.4] and earlier

See the original Kannel changelog for historical changes.
