# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of the official (now largely stubbed-out) `rustdesk-server`, re-implementing pieces of the paid
"RustDesk Server Pro" for free: TCP rendezvous mode, a REST API server, and a small web console
(personal/group/shared address books, users). Three binaries come out of this crate:

- `hbbs` (`src/main.rs`, package name `hbbs`) — ID/rendezvous server. Spawns the REST/web-console API
  (from the separate `sctgdesk-api-server` crate) on one thread and `RendezvousServer::start` (this
  repo's `src/rendezvous_server.rs`) on the main thread.
- `hbbr` (`src/hbbr.rs`) — relay server.
- `rustdesk-utils` (`src/utils.rs`) — CLI helper, notably `genkeypair`.

`libs/hbb_common` is a **git submodule**, shared with the RustDesk client itself. It defines the wire
protocol (rendezvous messages, socket framing, `Config`). Because it's a submodule, changes there need
a separate round-trip; prefer keeping server-only logic in `src/`.

`sctgdesk-api-server` (the REST API + web console) is a **git dependency** pulled from
`sctg-development/sctgdesk-api-server@main`, not vendored here. The Docker build runs
`cargo update -p sctgdesk-api-server` before compiling, so the API/web-console layer can change between
builds even with no commits to this repo — keep that in mind when something in that layer behaves
differently than expected.

## Protocol compatibility is the recurring failure mode

Official `rustdesk-server` stubs out `RegisterPk` on the TCP path (`register_pk_response::Result::NOT_SUPPORT`);
this fork actually implements it (`handle_register_pk` in `rendezvous_server.rs`) to support features the
official server doesn't. That means **this fork can drift out of sync with what current RustDesk clients
actually send**, silently, because nothing upstream tests that path. Confirmed example: current clients
send `RegisterPk` with an empty `pk` field when renaming an already-registered device (Change ID) — the
crypto identity doesn't change. `handle_register_pk` used to reject that unconditionally as
`INVALID_ID_FORMAT`; it now falls back to the `pk` already on file for `old_id` when the `uuid` matches.

When something a modern RustDesk client does against this server fails with a vague client-side error
(e.g. "Invalid format"), suspect this class of bug first: compare what the current official RustDesk
client actually sends on the wire against what `handle_tcp`/`handle_register_pk`/`handle_udp` in
`src/rendezvous_server.rs` require. There's no test suite covering this — verification is functional
(see below).

## Build

Needs `DATABASE_URL` pointing at a sqlite file *before* `cargo build` (sqlx compile-time query checking
against `db_v2.sqlite3`'s schema, migrated via `db_v2/migrations/`).

```bash
# Unix
DATABASE_URL=sqlite://$(pwd)/db_v2.sqlite3 cargo build --release
# Windows cmd.exe
set "DATABASE_URL=sqlite://%CD%/db_v2.sqlite3" && cargo build --release
```

Produces `hbbs`, `hbbr`, `rustdesk-utils` in `target/release/`.

### Docker

Two build paths:
- `Dockerfile` — thin runtime image, copies binaries from `sctg/sctgdesk-server-integration:latest`.
- `Dockerfile-integration` — full multi-stage build from source (installs Rust + vcpkg-free native deps,
  runs the `cargo build` above, cross-compiles per `dpkg --print-architecture`). This is the one to use
  when testing a local source change end-to-end.

The final stage of `Dockerfile-integration` does **not** set `WORKDIR`. If your compose/run command
doesn't set `working_dir: /usr/local/share/sctgdesk` (or `-w`), `hbbs`/`hbbr` will read/write the
`id_ed25519(.pub)` keypair and `db_v2.sqlite3` in the wrong place — the server will look healthy but
regenerate its key (and thus disconnect every registered client) on every container recreation. Always
verify `cat data/id_ed25519.pub` is unchanged across a restart before trusting a deployment.

## Verifying a change (no automated tests)

There is effectively no test suite here; changes to the rendezvous protocol are verified functionally:

1. Build the image (`docker build -f Dockerfile-integration -t <tag> .`) and run it via compose with
   `working_dir` set as above.
2. `docker logs -f <hbbs container>` with `RUST_LOG=debug` — watch for `Tcp connection from ...`,
   `KeyExchange phase 1`, and (if it gets that far) `update_pk <id> ...` from `src/peer.rs`. A connection
   that opens, gets a KeyExchange phase 1, and closes immediately without an `update_pk` line means the
   client's request was rejected before `handle_register_pk`/`handle_tcp` accepted it.
3. Connect an actual current-release RustDesk client (not just an old cached build) against the server
   and reproduce the real user action (e.g. Settings → Change ID) — client-side error text is often
   misleading/generic, so the server log's raw bytes are more trustworthy than the client's error string.

## ENV variables (hbbs/hbbr)

| variable | binary | description |
| --- | --- | --- |
| `KEY` | both | force a specific signing key; `_` forces "any key" |
| `RELAY` | hbbs | hbbr host(s), comma-separated |
| `ALWAYS_USE_RELAY` | hbbs | `Y` disallows direct P2P |
| `LOGGED_IN_ONLY` | hbbs | `Y` restricts control to logged-in users |
| `OAUTH2_CONFIG_FILE` / `OAUTH2_CREATE_USER` | hbbs | oauth2 config path / auto-create user |
| `S3CONFIG_FILE` | hbbs | s3 config path (`s3config.toml`) |
| `PORT` | both | listening port (21116 hbbs / 21117 hbbr) |
| `RUST_LOG` | both | `error\|warn\|info\|debug\|trace` |
| `SINGLE_BANDWIDTH` / `TOTAL_BANDWIDTH` / `LIMIT_SPEED` | hbbr | bandwidth caps (Mb/s) |
| `DOWNGRADE_START_CHECK` / `DOWNGRADE_THRESHOLD` | hbbr | relay downgrade tuning |

Default web-console login is `admin` / `Hello,world!` at `http://<server-ip>:21114/`; API docs at
`http://<server-ip>:21114/api/doc/`. The embedded API server has no DDOS protection — put it behind a
reverse proxy (see README's HAProxy example) for anything internet-facing.
