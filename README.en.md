# VHDB — a history database for vibe coding

[한국어](README.md) | English

VHDB stores the **conversations, decisions, errors, fixes, edits and commits** of a vibe-coding session, and later answers "have I done this before?", "why did we decide that?" and "which conversations shaped this file?". It is written entirely in Rust and depends on no external database.

- **The session log is the WAL.** It reads the `sessions/*.jsonl` files written by the decision ledger of [smart-workbench](https://github.com/nathak/smart-workbench) (`/swb ledger on`) as they are, and only reads them. Every derived file (segments, indexes, manifest) can be deleted and rebuilt identically from the log alone.
- **Immutable files, atomic replacement.** Checksummed segments are written first, and the state only changes when `MANIFEST` is atomically replaced. Whenever a process dies, reading the same log bytes again reaches the same result.
- **Search uses indexes; the reference answer is an exhaustive scan.** Random data is used to keep checking that the indexed result equals scoring every candidate one by one, down to the score bits.
- **One engine runs in three places.** Locally (WASM, inside the Claude Code plugin), as a personal daemon (`vhdbd`), and as a team server (`vhdb-server`).

This document is the guide to VHDB. The source is not published; the install files are on the [releases page](https://github.com/nathak/vhdb-public/releases/latest). The Claude Code plugin lives in [`nathak/vhdb-plugin`](https://github.com/nathak/vhdb-plugin).

---

## Contents

1. [Overview](#1-overview)
2. [Quick start](#2-quick-start)
3. [Data model and directories](#3-data-model-and-directories)
4. [Query language (VQL) and ranking](#4-query-language-vql-and-ranking)
5. [Command line `vhdb`](#5-command-line-vhdb)
6. [Local daemon `vhdbd`](#6-local-daemon-vhdbd)
7. [WASM engine and history logic](#7-wasm-engine-and-history-logic)
8. [Team server `vhdb-server`](#8-team-server-vhdb-server)
9. [Large data: partitions, compression, tiers, vectors](#9-large-data-partitions-compression-tiers-vectors)
10. [Guarantees](#10-guarantees)
11. [Performance](#11-performance)
12. [Limits and roadmap](#12-limits-and-roadmap)

---

## 1. Overview

![VHDB overview: smart-workbench logs and plugin logs enter vhdb-core (segments, indexes, MANIFEST) through ingest, and vhdb-wasm, vhdbd and vhdb-server answer queries on top of it (the labels in the picture are in Korean)](assets/architecture.png)

| Component | Role | Distribution |
| --- | --- | --- |
| Engine | Segments, index SSTs (bloom filters), MANIFEST, ingest, compaction, search (VQL, BM25), recovery, partitions, tiers, vectors, history logic. It does no OS I/O; storage is reached only through the `Storage` trait | Part of every component |
| `vhdb` command | stat / ingest / search / history / verify / compact / rebuild / dump / archive | Installed as `vhdbctl` in the server package |
| WASM engine | The engine as a single WASM file; a node host provides file access | Included in `vhdb-plugin` |
| `vhdbd` | One local daemon per project directory. Holds the write lock, follows the log tail and answers queries over a Unix socket | Not published yet |
| `vhdb-server` | Team server. Ingest and compaction, the `/v1/*` API, an admin page for accounts, projects and tokens, an audit log | .deb, .rpm, tar.gz + `install.sh` (x86_64, aarch64) |

---

## 2. Quick start

### 2-1. Search locally

Installing the server package (2-3) also installs the `vhdb` command under the name `vhdbctl`.

```sh
L="$(git rev-parse --git-common-dir)/swb-ledger"        # smart-workbench ledger directory
vhdbctl ingest  "$L"                                     # take in new lines, write indexes, merge small segments
vhdbctl search  "$L" token refresh kind:decision,reply since:30d
vhdbctl search  "$L" '"refresh race"' file:src/auth.js --limit 5
vhdbctl history "$L" src/auth.js                         # every turn that touched this file, newest first
vhdbctl stat    "$L"
vhdbctl verify  "$L"                                     # checksum check of every segment and index
```

`ingest` gives the same result from the same log no matter how often it runs or where it was killed.

### 2-2. Use it from Claude Code

Install [`vhdb-plugin`](https://github.com/nathak/vhdb-plugin) to get the `/vhdb:search` and `/vhdb:history` slash commands and the `query_history` and `file_history` tools that Claude calls on its own. Recording (record), context injection (inject) and the check before edits (guard) are all off by default; turn them on with `/vhdb:on`. The plugin currently uses **only the local WASM engine** and does not connect to a team server (the server connection is not implemented yet).

### 2-3. Start a team server

Get the install files (.deb, .rpm, tar.gz; x86_64 and aarch64) from the [releases page](https://github.com/nathak/vhdb-public/releases/latest). Change the version in the file names below to match what you downloaded.

```sh
sudo apt install ./vhdb-server_0.1.0-1_amd64.deb      # Ubuntu 22.04+ / Debian 12+
systemctl status vhdb-server
curl -s http://127.0.0.1:7600/healthz                  # {"ok":true}
```

The default configuration (`auth = "off"`) needs no accounts and only runs on loopback. To share it between several people, set up Google login in [section 8](#8-team-server-vhdb-server).

---

## 3. Data model and directories

### 3-1. Records

A record is one line (event) of the ledger. It has a session id, a turn number, a time, a kind (`kind`), a body and optional fields (`file`, `commit`, a file list and so on).

| kind | Meaning |
| --- | --- |
| `prompt`, `reply` | User input, model reply |
| `tool.start`, `tool.end` | A tool call and its result |
| `edit` | A file edit |
| `commit` | A commit (hash, message, file list) |
| `contract`, `session.start`, `turn.end` | Contract, session start, end of turn |
| `guard.*` | Guard verdicts (every kind starting with `guard.`) |
| `check` | Result of a check (test/build/lint and so on) |
| `error` | A failure (with an error fingerprint) |
| `decision` | A decision and its reason (a chain that supersedes earlier decisions) |
| `anchor`, `rollup` | Summary records made when old ranges are condensed (see [tiers](#9-large-data-partitions-compression-tiers-vectors)) |

Unknown kinds are stored as `other`.

### 3-2. Directories

```
<ledger dir>/                e.g. <git common dir>/swb-ledger
  sessions/*.jsonl           session logs = WAL (written by smart-workbench, VHDB only reads)
  data/seg-*.vdb             record segments (immutable once sealed; compaction output is deflated per slot)
  data/arc-*.vdb             originals of ranges moved to a tier (compressed, not indexed)
  index/idx-*.sst            one per segment: postings for words, files, commits, sessions, turns + document meta + bloom filter
  index/vec-*.vix            (opt-in) int8 vector index of a segment
  meta/ck-*, meta/sq-*       checkpoints and per-session highest seq, 64 shards (manifest format 2)
  MANIFEST                   list of valid segments and indexes, CRC line at the end
  LOCK                       the single write lock (pid@host; a lock of a dead process is taken over)
  vhdbd.sock                 (while the daemon runs) Unix socket
```

- **There is exactly one writer.** One process holds `LOCK` and does ingest and compaction. Any number of readers run at the same time, and reading never changes a file.
- **MANIFEST is the truth.** Segment and index files are written first, and new data becomes visible only when MANIFEST is atomically replaced. Files MANIFEST does not point to are deleted at the next write open (orphan cleanup). Files replaced by compaction are not deleted at once but two MANIFEST generations later, so a reader still using the previous MANIFEST can finish.
- **Ledger rule compatibility.** Broken lines and lines in other formats are skipped (the same rule as smart-workbench `parseLedger`), and a line that is still being written (no newline yet) is read next time. Logs copied under another name are filtered by the highest seq per session, so no duplicates appear.
- The engine accepts several `sessions/` directories. The plugin reads its own log (`vhdb/log/`) together with the smart-workbench ledger.

---

## 4. Query language (VQL) and ranking

| Syntax | Meaning |
| --- | --- |
| `token refresh` | Contains all the words (any of them with `--any` / `any=1`). Korean is matched by syllable bigrams; code also by camelCase, snake_case and path pieces |
| `"refresh race"` | The phrase appears as written (case-insensitive) |
| `file:src/auth.js` | Events that touched the file (edits, tool calls that mention it, commit file lists) |
| `commit:abc1234` | That commit (7 characters or more) |
| `session:ab12` | Start of a session id |
| `kind:decision,guard.*` | Event kinds. `*` groups them |
| `since:30d` `until:2026-10-01` | Time range: `90m` `12h` `30d` `8w`, a date (UTC), or epoch ms |

An `xxx:` that is not a field name is treated as an ordinary word (URLs and so on). The plugin adds `status:active|superseded|all` (decision state) and `error:<fingerprint>` on top of this (`history::query_ext`).

**Ranking**: BM25 (k1 = 1.2, b = 0.75) × kind weight × recency.

- Kind weights: decision 1.5, error 1.4, commit and contract 1.2, conversation (prompt, reply) 1.0, edit, guard and check 0.8, tool 0.6, anything else 0.7
- Recency: `0.3 + 0.7 × 0.5^(age / 30 days)`. It never falls below 0.3, so an old exact match beats a recent vague one.
- A query without words (filters only) is ordered newest first; ties go to the newer, then to the larger document number.

**Semantic search (opt-in)**: with `--semantic` / `semantic=1` the BM25 result and the vector cosine result are merged by RRF (k = 60). It works only on directories where vectors are enabled ([section 9](#9-large-data-partitions-compression-tiers-vectors)). Without `--semantic` the output equals the plain text path and has no `cos` field.

---

## 5. Command line `vhdb`

```
vhdb <command> <dir> [args]

  stat    <dir> [--json]            segments, indexes, records, sessions, time range
  ingest  <dir> [--tier] [--embed hash[:DIM]]
                                    take in new log lines, index, merge small segments
                                    --tier: move old quarters to warm/cold (originals go to the archive)
                                    --embed: turn on semantic search (built-in hash embedder, default DIM 64)
  archive <dir> <session> <turn> [--part P]
                                    original events of a turn moved to warm/cold
  search  <dir> <query> [--any] [--semantic] [--limit N] [--json]
  history <dir> <file> [--limit N] [--json]
                                    turns that touched the file, newest first, whole turns
  verify  <dir>                     compare every segment and index with the manifest and checksums
  compact <dir>                     merge a group of small segments now
  rebuild <dir>                     drop all derived files and rebuild them from the log
  dump    <dir> [--kind K] [--session S] [--since MS] [--until MS] [--limit N]
                                    records as JSON lines (newest last)
```

Exit codes: 2 for a usage error, 1 for any other error. When something is corrupt (`Corrupt`), it points you to `rebuild`. The hidden developer command `bench` repeats one query in a single process and measures the first and the warm latency: `vhdb bench <dir> "<query>" --limit 60` (here `--limit` is the repeat count, default 20).

`<dir>` is the ledger directory. In the server package the same program is installed under the name `vhdbctl`.

---

## 6. Local daemon `vhdbd`

```sh
vhdbd <dir> [--tier] [--idle-secs N] [--tick-ms N] [--sock PATH]
```

One daemon per project directory holds the write lock (so it is "the only indexer of the project") and follows the session logs every `--tick-ms` (default 250 ms), taking in new lines. A client sends line-oriented JSON over a **Unix socket**. One line is a request, one line is the answer, and a query does not change the engine. The socket is `<dir>/vhdbd.sock`; if that path is longer than 100 bytes it uses `$XDG_RUNTIME_DIR/vhdbd-<hash>.sock` (or `/tmp` when that variable is unset).

| Request | Description |
| --- | --- |
| `{"cmd":"hello","session":ID}` / `{"cmd":"bye","session":ID}` | Register / unregister a client |
| `{"cmd":"search","q":"VQL","limit":20,"any":false,"partitions":3,"semantic":false}` | → `{"ok":true,"hits":[…],"partitions_used":…,"partitions_skipped":…}` |
| `{"cmd":"history","file":"src/a.ts","limit":5}` | → `{"ok":true,"turns":[…]}` |
| `{"cmd":"archive","session":"s","turn":3,"part":"2025Q1/src"}` | Original events of an old turn |
| `{"cmd":"stat"}`, `{"cmd":"ping"}` | Status, liveness check |

`limit` is clamped to 1–500. **Exit condition**: when no session is registered and no connection is open for `--idle-secs` (default 60 s), the daemon removes the socket and exits (sessions of connections dropped without `bye` are released too). Moving data to a tier cannot be undone, so it happens only with `--tier`.

---

## 7. WASM engine and history logic

### 7-1. WASM

`vhdb-wasm` is the same engine built for `wasm32-unknown-unknown` (size-optimised profile, about 1 MB). It has no file access, so a node host (`vhdb.mjs`) provides reading, writing and locking. The commands are `stat / ingest / compact / verify / rebuild / search / history`. Write commands take the `LOCK` file. If the host and the wasm do not speak the same protocol number (`h.version`), nothing is written. Every build is checked with a node smoke test.

### 7-2. History logic

Pure Rust that runs without storage; the local engine (WASM), the CLI and the server use the same code. The plugin's JS only handles hooks, MCP, files and locks, and everything below is computed here. To give the same results as the old JS implementation, `history::js` reproduces JS string semantics (UTF-16 length, the `\s` whitespace set, `String(x)`, `x | 0`, `Math.round`).

| Module | What it does |
| --- | --- |
| `redact` | Secret masking (tokens, keys, URL credentials, private key blocks and more, 15 kinds) |
| `checks` | Whether a Bash command is a check (test/build/typecheck/lint), whether its exit code is hidden (pipes, `;`, `&`), whether it is sensitive |
| `fingerprint` | Error fingerprint with run-dependent values removed (first 16 characters of sha1) |
| `events` | Analysis of failed/passed calls: the event to record and the value for the error cache |
| `errors` | Error cache transitions: apply a failure, resolve when the same check passes (observed/inferred), clean up open errors |
| `decisions` | Decision state (supersede chain), validation of whether something can be superseded |
| `inject` | Choosing, scoring, budgeting and rendering injection candidates, T2, prompt word extraction |
| `format` | Search and history results as one-line text (tool output bodies are never included) |
| `query_ext` | `status:`/`error:` query words and the decision state filter |

A WASM command is `h.<name> <one JSON>` and answers `{"ok":true,"value":…}` (it does not open a store): `h.version h.redact h.sha1 h.tokens h.estimate_tokens h.safe h.classify h.is_sensitive h.fingerprint h.analyze_fail h.analyze_pass h.errors_fail h.errors_pass h.errors_prune h.decisions_resolve h.decisions_chain_end h.supersede_check h.inject h.t2 h.prompt_files h.chat_query h.format_search h.format_history h.parse_query h.filter_hits`.

Tests: about 100 input/output pairs per verb (checked against the pre-migration JS implementation with random inputs) run on every build.

Intended differences: for dates outside ±8.64e15 ms JS throws or prints a 6-digit year, while Rust prints `?`. When a UTF-16 pair is cut in the middle, JS keeps half a surrogate (it becomes U+FFFD when encoded) while Rust writes U+FFFD from the start. The fingerprint hash input is the same in both cases. Performance (wasm, input size): fingerprint of a 64 KB failure output about 40 ms and masking about 12 ms; for 1 MB about 0.56 s and 0.18 s (JS took 66 ms and 11 ms), and it only runs in the failure hook. `opt-level = "z"` shrinks the file by only 11% and is 1.7× slower, so it is not used.

---

## 8. Team server `vhdb-server`

### 8-1. Install (Linux)

Get the files from the [releases page](https://github.com/nathak/vhdb-public/releases/latest).

```sh
sudo apt install ./vhdb-server_0.1.0-1_amd64.deb      # Ubuntu 22.04+ / Debian 12+
sudo dnf install ./vhdb-server-0.1.0-1.x86_64.rpm     # RHEL / Rocky 9+
sudo ./install.sh --start                             # anything else: unpack the tar.gz and run
```

| Item | Location |
| --- | --- |
| Configuration | `/etc/vhdb/vhdb.toml` (after editing, `systemctl restart vhdb-server`) |
| Data | `/var/lib/vhdb` (kept when the package is removed) |
| Service | `systemctl status vhdb-server` (sandbox: `ProtectSystem=strict`, `NoNewPrivileges`, `MemoryDenyWriteExecute` and more) |
| Audit log | `/var/log/vhdb/audit.jsonl` |
| Admin tool | `vhdbctl` (the same commands as the CLI) |
| Configuration check | `vhdb-server --check-config --config /etc/vhdb/vhdb.toml` (runs automatically before the service starts) |

By default it listens only on `127.0.0.1:7600`. For remote access put a TLS reverse proxy in front and make it pass the `Host` header through unchanged (for nginx: `proxy_set_header Host $host;`). Otherwise every browser path (`/auth/*`, `/admin*`) answers 421.

### 8-2. Modes

| `auth` | Description |
| --- | --- |
| `"off"` (default) | Serves a single `data_dir` without accounts. **Starts only on a loopback address.** The `project` of a request is ignored |
| `"oidc"` | Google login (admin page) + personal API tokens. One directory per project, completely isolated from each other |

### 8-3. Configuration keys (`vhdb.toml`)

A flat subset of TOML (`key = value`, ` #` inline comments, `#` comments at the start of a line). A list is one comma-separated string and is lower-cased (`admin_emails = "a@x.com,b@x.com"`). If the file is missing the defaults are used, and **an unknown key is an error**.

| Key | Default | Description |
| --- | --- | --- |
| `listen` | `127.0.0.1:7600` | Address to listen on. With `auth = "off"` only loopback is allowed |
| `data_dir` | `/var/lib/vhdb` | Data. The single directory used when `auth = "off"` |
| `ingest_interval_secs` | `5` | How often new log lines are taken in |
| `tiering` | `false` | Move old quarters to warm/cold (off by default because it cannot be undone) |
| `auth` | `off` | `off` or `oidc` |
| `public_url` | (empty) | The address at which browsers reach the server. The base of the redirect URI. http only on loopback, https otherwise |
| `projects_dir` | `/var/lib/vhdb/projects` | One `<name>/` directory per project |
| `state_dir` | `/var/lib/vhdb/control` | Users, projects, permissions, token hashes (`state.json`, up to 14 daily backups) |
| `audit_log` | `/var/log/vhdb/audit.jsonl` | Audit log (rotated, 100 MB × 5) |
| `max_projects` | `32` | Upper limit on the number of projects |
| `max_connections` | `256` | Upper limit on concurrent connections. Beyond it, an immediate 503 |
| `readers_per_project` | `0` | Read-only engines per project. 0 means the CPU count (2–16) |
| `oidc_client_id` | (empty) | Google OAuth client id |
| `oidc_client_secret_file` | `/etc/vhdb/oidc-client-secret` | Secret file. `root:vhdb 0640`. The server does not start if anyone else can read it |
| `oidc_issuer` | `https://accounts.google.com` | For another OIDC provider or a fake Google in tests |
| `oidc_authorize_url` | `https://accounts.google.com/o/oauth2/v2/auth` | Same as above (http only on loopback) |
| `oidc_token_url` | `https://oauth2.googleapis.com/token` | Same as above. A warning is logged at startup if it is not Google |
| `allowed_domains` | (none) | If set, only accounts of these domains (`hd` claim) can log in |
| `admin_emails` | (none) | Emails that become administrators at their first login |
| `session_idle_minutes` | `120` | Idle expiry of admin page sessions |
| `session_max_hours` | `12` | Maximum lifetime of admin page sessions |
| `token_max_days` | `90` | Maximum lifetime of a personal API token (clamped to 1–365) |

`auth = "oidc"` needs `public_url`, `oidc_client_id`, at least one well-formed `admin_emails` entry and `max_*` values of 1 or more; if the check fails the server does not start.

### 8-4. Setting up the team server (`auth = "oidc"`)

1. In the Google Cloud console create an OAuth client (web application) and register `<public_url>/auth/callback` as an authorized redirect URI.
2. Put the client secret in `/etc/vhdb/oidc-client-secret` (`chown root:vhdb`, `chmod 0640`).
3. In `vhdb.toml` write `auth`, `public_url`, `oidc_client_id`, `admin_emails` (the first administrator) and optionally `allowed_domains`, then `systemctl restart vhdb-server`.
4. Open `<public_url>/admin` in a browser and log in with the administrator account. Create a **project (repository)**, register **users**, and assign each project a **permission** (read / read+write).
5. Each user issues a **personal API token** on the same page and uses it in the command line and the plugin.

### 8-5. One connection, one project

A token is issued bound to **exactly one project**. The server enforces "you cannot work on A and ask about B".

- A request sends `Authorization: Bearer vhdb_…` together with `project=<name>`. `project=` is required and must equal the token's project exactly once.
- Missing, repeated, mismatched, `a,b`, `*` and empty values are all the same 400 (`project must be the one this token was issued for`). The decision compares strings before looking at whether the project exists, so project names do not leak.
- A missing or wrong token is 401, a revoked permission is 404, and a deactivated user is 401. An old token that is not bound to a single project gets 401 (`this token must be reissued for one project`) and shows as "needs reissue" in the admin page.
- Archiving a project or recreating it under the same name revokes the tokens bound to that name.
- Someone who works on two projects has two tokens. Cross-project search was not built because it conflicts with this rule.
- The binding sits on the token, not on the TCP connection, because the server sends `Connection: close` on every request and proxies share upstream connections.

A project is one directory, so indexes, scores (document frequency) and statistics are never mixed between projects. Members of the same project see each other's history in full.

### 8-6. HTTP API

`/v1/*` accepts only a Bearer token (a token in the query string is rejected) and **only `GET`** (anything else is 405). The upload route `/v1/ingest` also takes `POST`. The admin API (`/admin/api/*`) uses cookie authentication and also uses `POST`. `limit` is clamped to 1–500.

| Request | Description |
| --- | --- |
| `GET /healthz` | No authentication. `{"ok":true}` |
| `GET /v1/stats?project=P` | Segments, records, sessions, time range |
| `GET /v1/search?project=P&q=<VQL>&limit=20&any=1&semantic=1&partitions=3` | `{"hits":[…],"partitions_used":…,"partitions_skipped":…}`. `semantic=1` needs vectors (400 otherwise). `partitions=N` opens only the newest N partitions |
| `GET /v1/history?project=P&file=<path>&limit=5` | Turns that touched the file (whole turns), newest first |
| `POST /v1/ingest?project=P` | Upload events (NDJSON). `{"session":…,"accepted":N,"duplicate":M,"high_seq":H}`. Tokens that may write only |
| `GET /v1/ingest?project=P&session=S` | Where to resume: `{"session":…,"high_seq":H}`. Tokens that may write only |

| Status | Meaning |
| --- | --- |
| 400 | `q`/`file` missing, query error, project mismatch, malformed request, an invalid upload line (only its number is reported) |
| 401 | Token missing, wrong, expired or revoked; old unbound token (`WWW-Authenticate: Bearer`) |
| 404 | No permission or no such project (not distinguished), admin API for a non-admin user |
| 405 | A method `/v1/*` does not allow |
| 409 | The uploaded session id already belongs to someone else (or is a session that exists only in a server-local log) |
| 411 | An upload without `Content-Length` |
| 413 | Request body over 64 KiB (an upload may carry 4 MiB, a session file may grow to 64 MiB) |
| 431 | Request line or header line too long, or more than 64 headers |
| 421 | The `Host` of a browser path does not match `public_url` |
| 500 | The project cannot be opened, read error (including corruption) |
| 501 | Chunked request body |
| 503 | Connection limit exceeded, more than 4 large uploads at once, all read engines busy for more than 10 s, the snapshot keeps moving, the project cannot be written, an upload could not be written to disk (send it again) |

**Upload (`/v1/ingest`)**: only a token that may write (scope `write` or `rw`, and write permission in the project) can use it. The `POST` body is NDJSON (one event per line) and one request carries one session (up to 4 MiB per request, 1 MiB per line, 10,000 events). Every line is checked for `session`, `seq`, `kind` and `at` (not before 2020 and not more than 5 minutes ahead of the server clock), and one bad line refuses the whole request. The server overwrites `owner` with the uploading user and masks secrets (keys, tokens, passwords) once more. The answer is sent only after the data has been fsynced. A line whose `seq` is at or below `high_seq` is counted as a duplicate and skipped, so sending the same request again changes nothing. To resume, use the `high_seq` of the last answer or ask with `GET /v1/ingest`. A session id belongs to the first user that uploads it; another user using the same id gets the same 409 whatever the reason, and it does not name the owner. Uploaded data becomes searchable from the next ingest tick (`ingest_interval_secs`). With `auth = "off"` uploads are not accepted (404). A body over 64 KiB is read only after the token in its headers has been checked.

A request sees the data as of the last finished ingest tick. The reading side is a pool of read-only engines (`Engine::refresh` catches up with the writer's commits in place), and searches keep answering during ingest.

### 8-7. Admin page and security model

- `<public_url>/admin` — a Korean-language admin page (built into the binary). It has tabs for my account (issue and revoke tokens), users, projects, all tokens and the audit log.
- Login is Google OIDC (authorization code + PKCE). Sessions live in server memory (a restart logs everyone out), and the cookie is `HttpOnly`, `SameSite=Lax`, and `Secure` on https. The admin API checks CSRF (JSON type + `X-CSRF-Token` + Origin/Sec-Fetch-Site) and `Host`.
- A token has the form `vhdb_<8-character id>_<32 bytes base64url>` and the server keeps only its SHA-256 hash. The secret is shown once, right after issue. Comparison is constant-time.
- The audit log writes logins, token, project and permission changes and every `/v1` request (including denied ones) as JSONL. It stores no tokens or secrets, and a search query is stored only as the first 16 characters of an HMAC under a per-process key plus its length.
- The request parser has limits: request line 8 KiB, 64 headers, body 64 KiB (4 MiB for an upload whose token has been checked), chunked refused, 20 s deadline overall (against slowloris).
- The release profile uses `panic = "abort"`, so a panic ends the process and systemd restarts it.

### 8-8. Not there yet

- **An uploader in the plugin.** The server accepts `POST /v1/ingest`, but nothing calls it yet (the plugin).
- A storage limit per project or user (today only one session file is capped at 64 MiB and 4 large uploads at once).
- Login verified with real Google credentials (the whole flow is verified with a fake Google server).
- A procedure to delete data of leavers or after a leak (the log structure means rebuilding the directory).
- Persistence of login sessions (everyone is logged out after a restart).

---

## 9. Large data: partitions, compression, tiers, vectors

- **Partitions**: segments are split by (UTC quarter × module). The module is the first path component of the record's main file (`file`, or the first of the file list), configured prefix rules win, root files are `_root` and events without a file are `_`. The key is decided from a single record, so rebuilding gives the same result.
- **Query routing**: time conditions and `file:` narrow the segments to open, and the automatic injection path (`partitions=N`) opens only the newest N partitions. The document frequency in a score is that of the opened partitions.
- **Compression**: segments written by compaction are deflated per slot (64 records, format 2). On 1M synthetic events this took 201 MB → 76 MB with unchanged query latency. (Index SST compression is not done yet.)
- **Tiers (opt-in)**: `--tier` / `tiering = true` / `vhdbd --tier` / `VHDB_TIER=1` in the WASM host move partitions whose quarter ended more than 90 days ago to **warm** (long tool outputs keep only the first and last 1 KB) and after one year to **cold** (decisions, errors, commits, resolved checks + an `anchor` per prompt + a partition `rollup`). Originals stay in `data/arc-*.vdb` and are read with `vhdb archive`, the daemon `archive` command or `Engine::archive_turn`. Moving over time and rebuilding from the log and moving in one go give the same result (checked by tests). It cannot be undone, so it is off by default.
- **Checkpoint sharding** (manifest format 2): per-log checkpoints and the per-session highest seq are split into 64 shards under `meta/`, and one flush rewrites only the shards of changed sessions. Readers do not read the shards. A format 1 directory is still readable and is converted at the first flush on the writing side; old builds reject format 2 with `Error::Format`.
- **Vector index (opt-in)**: turned on with `vhdb ingest <dir> --embed hash[:DIM]`. int8 quantisation; below 100k per partition an exhaustive scan, above it HNSW (M = 16, ef_construction = 128, ef_search = 64). Insertion order and ties are fully deterministic, so the same input gives the same bytes (`vec-*.vix`, CRC). On 100k synthetic clustered vectors × 64 dimensions the graph recall was 0.998 (against an int8 exhaustive scan) and search p95 was 0.3 ms.
  - The only embedding providers are the `Embedder` trait and a hash-based stand-in (`HashEmbedder`); connecting a real model has not been decided.
  - Targets are prompt, reply, decision, error, commit, anchor, rollup and checks that have a resolution link; tool output and edits are not embedded. Embedding happens only in the backfill step of `maintain` (`ensure_vectors`) and does not block flush or ingest. Compaction copies the vector rows of the input segments as they are, so it works without an embedder and its result bytes equal a rebuilt one.
  - `set_embedder` rejects a different model or dimension. `rebuild` keeps only the `vec` setting, and the vector files are rebuilt by the backfill of the next `ingest`.
  - The manifest of a directory with vectors enabled is written as format 3, so an old build that does not know vectors refuses to open it (so it cannot delete the `.vix` files as orphans by mistake). Directories without vectors stay on format 2.
  - Limits: the vector side takes the top `max(8×limit, 64)` and only then applies the `file:`, `session:` and `commit:` filters, so a very selective filter can come up short; segments without vectors yet and the memtable are left out of semantic search. It is an exhaustive scan per segment, so a very large directory would need per-partition HNSW merging.

---

## 10. Guarantees

- **Crash safety**: segments are written first and MANIFEST is replaced atomically. Checkpoints move only together with MANIFEST, so wherever it dies, reading the same log bytes again reaches the same result. Files MANIFEST does not point to are deleted at the next write open.
- **Incremental = rebuilt**: however ingests and flushes are interleaved, the result equals rebuilding from the whole log from scratch (random property tests).
- **Corruption detection**: CRC-32C on every record, a segment footer CRC, an index block CRC, and a whole-file CRC per file in MANIFEST. Find corruption with `vhdb verify` and repair it with `vhdb rebuild`.
  - *Note*: search checks **only the blocks it reads**. The fast path does not read lists it does not need (skipping parts without a required word, parts skipped by the score bound), and the server/daemon remember document frequencies they have counted once, so corruption in a place a query skipped is not reported by that query. A list that is used is checked on reading against its CRC and the remembered count. To check every block use `vhdb verify`.
- **Reading deletes nothing**: reads such as `stat` and `search` do not change files. Orphan cleanup is done only by a write that holds the lock.
- **Search = exhaustive scan**: random data is used to check that the fast path (a global top-k heap, skipping documents and partitions by score bound, a document frequency cache) returns the same documents in the same order with the same score bits as the reference path that scores every candidate (`search_exhaustive`), whether the cache is cold, warm or tiny.
- **Server isolation**: a directory per project and one project per token. It is verified by a differential test that compares response bytes with a "server that has only A" and by a canary matrix (strings planted in bodies, file, session, commit and decision appear 0 times in responses of other projects).

---

## 11. Performance

### 11-1. Search (1M synthetic events, 107 segments, release build)

Warm p50 (ms) repeated in a single process with `vhdb bench`. The server and the daemon live in this state for a long time, so these numbers are closer to real use. `w0` is a word in about 30% of all documents. Because the data is synthetic, common words are far more common than in reality.

| Query | Before optimisation | 1st round | 2nd round (now) |
| --- | --- | --- | --- |
| `w0` | 15.7 | 8.1 | **3.7** |
| `w0 w2` | 24.9 | 14.4 | **7.8** |
| `w3 w1 w0` | 30.3 | 16.6 | **8.3** |
| `w3225` (rare word) | 13.7 | 7.9 | 6.9 |
| phrase `"w34 w95"` | 38.5 | 25.5 | 23.5 |
| `retry` | 5.1 | 2.6 | 2.0 |
| with an absent word `w5000 w0` | 20.9 | 0.0 | 0.0 |

- "Before optimisation" is the value from the internal measurement record; "1st round" and "2nd round" were measured again in the same session (the 1st round agrees with the earlier measurement to within 1 ms).
- The first (cold) query is not made faster by the document frequency cache of the 2nd round. For phrases and rare words the remaining cost is candidate intersection and record checking, which barely changed.
- Server (8 clients, the 1st-round and 2nd-round builds measured alternately on the same PC, machine busy): `w0 w2` 428 → 726 req/s, `retry` 2020 → 2314, `w3225` 640 → 744.
- **1st round**: slicing-by-8 CRC-32C, scanning index blocks in place without copying, filtering absent words with bloom filters before reading, positioned reads. **2nd round**: CPU-instruction CRC-32C (SSE4.2 / aarch64 CRC, detected at run time; software in WASM), a document frequency cache for words, and lazily reading only the lists of the parts that are needed. The storage format and the search results (down to the score bits) are unchanged.

### 11-2. Storage and server resources

| Item | Result |
| --- | --- |
| Full verification (`verify`, 1M events, 107 indexes) | about 0.17 s (after the 1st-round optimisation) |
| Filter queries `file:` `session:` `since:` (at M0, 34 segments) | warm 3–10 ms |
| Index backfill of an old directory without indexes (at M0, 34 indexes, 24 MB) | 13.7 s |
| Ingest with no new lines | about 7 ms (measured at M0) |
| First ingest (188 MB of log, 1M events) | 5.9 s (measured at M0, 34 segments, 191 MB) |
| Idle server RSS | 14 MB with 1.1M records open (about 53 MB when semantic search is used) |
| Peak RSS during ingest | 63 MB in the 500k–1M range |
| Disk | 1.1M records: 267 MB of original log, about 475–525 MB in total |
| Server, project of 120k records (with authentication + audit, 2 vCPU VM) | 1 / 8 / 32 clients: 242 / 300 / 298 req/s, 0 errors, RSS 9 MB |

Server resources were measured on a 2 vCPU / 2 GB VM. "Measured at M0" refers to older data with 34 segments, so do not compare it with the data of 11-1 above.

---

## 12. Limits and roadmap

- **No cross-project search**: it was not built because it conflicts with the rule that one token reaches one project.
- **No plugin connection to the server and no uploader in it**: the server accepts uploads (`POST /v1/ingest`), but the plugin currently uses only the local (WASM) engine. The engine build inside the plugin is recorded in `vhdb-plugin/engine/SOURCE`.
- A real embedding model connection: for now there is only the hash-based stand-in.
- Performance candidates: index SST compression, a verified block cache, file handle reuse, faster ingest (about 23 µs/event, dominated by indexing and compaction), candidate intersection for phrase and rare-word queries.
- Login sessions of the server are memory-only, so a restart logs everyone out.
- Because of the log structure, individual history cannot be deleted. For leavers or after a leak the directory is rebuilt.
- Times are shown in UTC. Line-level history before a commit (recording edit hunks) does not exist yet.

License: [MIT](LICENSE)
