# Testing Strategy

Rule: **every business rule (BR-xx in PRODUCT.md) has at least one automated test, and negative cases for authorization rules.** A feature PR must add rows to the matrix below with real test names.

## 1. Pyramid

| Layer | Tooling | Scope |
|---|---|---|
| Unit (Rust) | `cargo test`, `proptest` for state machines | Pure domain: room lifecycle, playback seq logic, license parsing, code generation, protocol codec |
| Integration (Rust) | `cargo test` + `testcontainers` (real Redis and Postgres) | Lua scripts, repositories, REST handlers via `tower::ServiceExt`, WS flows with `tokio-tungstenite` clients |
| Multi-instance | Compose or in-process two app instances on same Redis | Cross-instance fan-out, sweeper lock |
| Frontend unit | Vitest + Testing Library | Reducers, sync math, components |
| E2E | Playwright | Host + 2 participants in separate contexts: create/join/play/end, reconnect |
| Contract | Shared protocol fixtures (JSON) used by Rust and TS tests | Protocol drift |
| Non-functional | `k6` or custom WS load script | Fan-out latency, connections per instance |

Mock only truly external services (Jamendo, Neon Auth JWKS) with local fake servers (`wiremock`, or the `fake-auth` helper that generates an Ed25519 key pair, serves `/.well-known/jwks.json`, and mints test JWTs with chosen `sub`, `exp`, `iss`, `aud`, ban flag). Neon Auth is managed and has no local emulator, so all auth tests use this fake; one smoke test against a real Neon branch runs on `main` only. **Do not mock Redis or Postgres** for rules that depend on atomicity.

## 2. Business rule → test matrix

| Rule | Test level | Required tests (name prefix) |
|---|---|---|
| BR-01 | integration | `auth_rejects_missing_token`, `auth_rejects_expired_token`, `auth_rejects_wrong_issuer_or_audience`, `auth_rejects_non_eddsa_algorithm`, `auth_refetches_jwks_on_unknown_kid`, `ws_rejects_unauthenticated` |
| BR-02 | unit+int | `create_room_sets_creator_as_host`, `room_has_single_host` |
| BR-03 | int | `host_can_play_pause_seek_set_track`, `host_can_end_room` |
| BR-04 | int | `participant_play_is_forbidden`, `participant_command_not_broadcast`, `participant_cannot_end_room` |
| BR-05 | int | `join_ended_room_fails`, `ended_room_cannot_be_revived` |
| BR-06 | int | `join_beyond_capacity_fails_room_full`, `capacity_race_is_atomic` (concurrent joins) |
| BR-07 | int | `end_room_deletes_all_redis_keys`, `end_room_updates_postgres`, `end_room_notifies_and_closes_sockets`, `end_room_is_idempotent` |
| BR-08 | int | `host_disconnect_starts_grace`, `host_reconnect_within_grace_restores_control`, `host_grace_expiry_ends_room` |
| BR-09 | int | `participant_reconnect_no_duplicate`, `second_connection_supersedes_first` |
| BR-10 | int | `no_queue_or_history_persisted`, `all_room_keys_have_ttl` |
| BR-11 | unit+int | `seq_increments_per_command`, `client_ignores_stale_seq`, `concurrent_commands_serialize` |
| BR-12 | unit | `room_code_format_and_alphabet`, `room_code_collision_retries` |
| BR-13 | int | `ws_closes_on_token_expiry`, `ws_auth_timeout`, `ws_auth_refresh_extends_session`, `ws_auth_refresh_rejects_different_sub`, `banned_user_rejected` |
| BR-14 | int | `second_hosted_room_rejected` |
| BR-15 | unit | `license_nc_blocked_in_commercial_mode`, `attribution_always_present`, `client_cannot_supply_audio_url` |
| BR-16 | int | `ws_rate_limit_enforced`, `room_create_rate_limited` |
| BR-17 | int | `postgres_has_no_live_state_columns` (schema assertion), `stats_updated_at_end` |

## 3. Conventions

- Tests are deterministic: inject a `Clock` for time-dependent logic (grace periods); no real sleeps over 100 ms.
- Each integration test uses isolated keyspace/prefix or a fresh container.
- Test names follow the matrix prefixes so the matrix can be verified by script (`scripts/check-rule-coverage.sh`, to be added in roadmap phase 1).
- Property tests for playback: arbitrary command sequences never decrease `seq`, never let a non-host mutate state.

## 4. CI gates

`cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`, `cargo deny check`, frontend `lint`, `tsc --noEmit`, `vitest run`, `rspack build`, Playwright smoke on main. Coverage is tracked but rule coverage via the matrix is the real gate.
