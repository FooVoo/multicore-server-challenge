# Elixir: a minimal HTTP server on `:gen_tcp` + Postgrex

**Front: direct**

No Phoenix, Plug, Bandit or Cowboy. One BEAM process per keep-alive connection, `:erlang.decode_packet` for HTTP
parsing, hand-built iodata responses, Postgrex for Postgres, `bcrypt_elixir` (NIF on dirty CPU schedulers) for passwords.

## Stack

| Piece | Version |
|---|---|
| Erlang/OTP | 28.5.0.8 (hex.pm's official Ubuntu 24.04 build, sha256-pinned in `install.sh`; amd64 and arm64) |
| Elixir | 1.19.6 (official release zip, sha256-pinned) |
| postgrex | 0.22.4 (with db_connection 2.10.2, decimal 3.2.0, telemetry 1.4.2) |
| bcrypt_elixir | 3.3.2 (comeonin 5.5.1, elixir_make 0.10.0), cost 10 |

All dependency versions and hashes are in `mix.lock`; `build.sh` runs `mix release` with `MIX_ENV=prod`.
JSON is OTP's built-in `:json` for request bodies only; responses are never encoded from maps.

## Why a custom server

- Bandit/Cowboy add a request struct, header maps, a Plug pipeline and per-connection state that a 19-endpoint JSON API
  does not need. With one connection per virtual user and ~0.26 req/s each, tens of thousands of sockets are idle almost
  all the time, so what matters is the cost of an idle connection and the cost of one request, not feature breadth.
- An idle connection here is one process (hibernated if the last request grew its heap beyond `HIB_WORDS`, default 2048
  words) plus one port. No buffer is kept between requests.
- Accepting: `ACCEPTORS` (16) processes block in `accept`; each one that gets a connection spawns its own replacement and
  then *becomes* the connection process, so there is no ownership hand-off and the accept queue is always served.
- `active: true` on the socket (a client sends its next request only after our response; head and body size are capped),
  so a request costs no `setopts` calls. `decode_packet(:http_bin)` is a C BIF.

## Optimizations and why

- **Front: direct.** Nginx would cost ~11% of the box and gzips nearly every response.
- **One statement per request** (statements adapted from the reference SQL in SPEC.md): each list page and its capped count are one
  statement; slug lookups fold into the statement that needs them; follow/favorite are single CTE statements that return
  the response row; ownership-checked UPDATE/DELETE with a 403/404 probe only when no row matched. Article creation with
  tags is one transaction of two statements (insert article + sorted tag upsert; link tags + read back), without tags it
  is one statement; a deadlock/serialization failure is retried once.
- **One Postgrex pool, 1.5 connections per core** (12 on the 8-core box): reads and writes share it, because a split
  pool leaves one side's connections idle while the other side queues. Postgres runs on the same cores, and backends
  beyond what the cores can serve only make every statement slower. `cache_statement` names every statement, so each
  is parsed once per connection and costs one Bind/Execute/Sync round trip. Session settings in the startup packet:
  `jit=off`, `max_parallel_workers_per_gather=0`. `queue_target` is raised to 2 s so DBConnection's CoDel does not shed
  load before our own latency budget would. The server refuses to start above 95 connections.
- **Feed rows** return `following` as a constant `true`: the statement reaches each row's author through the viewer's
  follows row in the same snapshot, so the per-row EXISTS is redundant.
- **Raw timestamps**: a custom Postgrex extension returns `timestamptz` as integer microseconds; they are formatted by
  hand (truncated to ms) with no DateTime structs.
- **JSON output** is iodata built directly; strings take a `:binary.match` fast path when nothing needs escaping.
- **JWT**: `:crypto.mac(:hmac, :sha256, ...)`, `:crypto.hash_equals` for the compare, the standard header is recognised
  by string equality (any other header is decoded and must say HS256), a flat `{"sub","iat","exp"}` payload is scanned
  without a JSON parser (anything else takes the general `:json` path, so both agree). Nothing is remembered between
  requests.
- **bcrypt** runs on dirty CPU schedulers (`+SDcpu`, a quarter of the cores and at least 2), which also bounds how many
  hashes run at once; queued hashes wait in the VM's dirty-scheduler queue. Logins are ~0.1% of requests, so a short
  queue there costs nothing in the overall percentiles, while more hashing threads would take cores from Postgres.
- **Native bcrypt compilation** uses `-O3 -fno-semantic-interposition -flto --param=max-inline-insns-auto=500 -fipa-pta`
  by default (`CFLAGS` can override it). These flags allow inlining across the NIF's C files without changing bcrypt's
  cost or source. Local qualification checked known vectors, native artifact identity and the comparison/wipe code.
  Login verification uses the first 72 password bytes to match the reference bcrypt comparison semantics; registration
  and password changes reject passwords longer than 72 bytes. Existing boundary checks include a UTF-8 character
  crossing byte 72, nonuniform prefixes, and long inputs.
- **VM flags** (`start.sh`, sized from `nproc`): normal schedulers on ~5/8 of the cores (`+S 5:5` on 8), since Postgres
  needs about as much CPU per request as we do; `+sbwt none +sbwtdcpu none +sbwtdio none` so idle schedulers sleep
  instead of busy-waiting on cores Postgres needs; `+P 2000000 +Q 1000000` because the default port limit (65536) would
  cap connections.

## Rules

- Every request reads Postgres while it is served; nothing is cached across requests (statement plans aside).
- Writes are committed before the response (the response is built from the result of the committing statement or
  transaction).
- Every token's signature, `alg`, `exp` (and `nbf` if present) are checked on every request; bcrypt cost 10 on every
  register, login and password change.
- No DDL, no server-wide settings.

## Configuration

`DATABASE_URL`, `JWT_SECRET` (required), `HOST`, `PORT`; tunables `SCHEDULERS`, `DIRTY_CPU`, `POOL`, `ACCEPTORS`,
`HIBERNATE`, `HIB_WORDS`.

## Tests

`mix test` (ExUnit, no database needed): JWT, query-parameter validation, JSON escaping and timestamp formatting.

## Existing local evidence

Host: MacBook Pro M1 Pro (10 cores, 32 GB), running a Lima Ubuntu 24.04 arm64 VM (10 vCPUs, 16 GB), PostgreSQL 18
and the full seed (500k articles, 100k users). These are local experiments, not an eight-core x86_64 challenge score.

- The saved native-build qualification log records `mix test`: 11 tests, zero failures. Its build log uses the native
  compiler flags above; the deployment checked the stripped NIF against the independently tested candidate.
- An earlier build has a saved conformance summary: 197 steps, 197 passed, zero failed. Later dev/full database
  checks each saved a 197-step transcript; those transcripts do not include the pass/fail summary.
- These validation and performance artifacts predate the final FEED-only `following = true` rewrite. The final
  revision has not been rerun; the earlier results are not a claim of final-revision qualification.

Local performance run `eba_20261017T075317Z` used the optimized bcrypt candidate with server plus Postgres on
vCPUs 0-3 and four Linux load-generator processes on the remaining six vCPUs, over loopback. CPU isolation was
local harness setup; the submission performs no pinning. Each probe used 60 s ramp + 60 s measured + 15 s tail:

| Users | req/s | p50 / p95 / p99 (ms) | Errors | Result |
|---:|---:|---:|---:|---|
| 24,000 | 6,393.3 | 1.0 / 3.5 / 7.0 | 0% | PASS |
| 32,000 | 8,298.2 | 88.3 / 231.2 / 276.5 | 0% | PASS |

Both probes reported `lg_limited: no` and `behind=false`. No five-minute hold or failing upper bound was established
by these two probes. They describe the archived pre-FEED-rewrite build, not a measured ceiling for the final source.
