# ADR-0007: Neon Postgres and Neon Auth replace Supabase

Status: **Accepted** (owner-directed change, 2026-11-21). Supersedes the Supabase parts of ADR-0005 and the Supabase references in the original brief.

## Context
The owner chose Neon for both the database and authentication instead of Supabase. Findings from Neon's documentation (verified 2026-11-21):
- Neon Auth is a managed service built on Better Auth; users, sessions and credentials live in the `neon_auth` schema of the project database.
- Browser apps use an HTTP-only session cookie. For separate backends, the JWT plugin issues EdDSA (Ed25519) JWTs valid for a fixed 15 minutes, verifiable via `<auth URL>/.well-known/jwks.json`; `iss` and `aud` are the auth URL origin; `sub` is the user id; custom claims are unsupported.
- Neon pooled connections (`-pooler` host) run PgBouncer in transaction mode: protocol-level prepared statements work; session `SET`, `LISTEN/NOTIFY`, temp tables and session advisory locks do not. Neon recommends direct connections for migrations.

## Decision
1. Identity: **Neon Auth**. The Rust backend verifies the JWT (EdDSA only, cached JWKS, `iss`/`aud` origin checks, banned users rejected). The backend never writes to `neon_auth`.
2. Database: **Neon Postgres** accessed with SQLx. Runtime queries use the pooled connection; migrations use a separate direct connection (`DATABASE_URL_DIRECT`).
3. User references in our tables are `text` ids (JWT `sub`) with **no foreign key** into `neon_auth`.
4. The 15-minute token lifetime makes `auth.refresh` over WebSocket mandatory (BR-13, API_AND_WS_PROTOCOL.md).
5. Neon Data API / RLS-over-HTTP is not used; authorization stays in application code.
6. Tests use a local `fake-auth` JWKS/JWT helper; one smoke test against a real Neon branch runs on `main`.

## Alternatives
Stay on Supabase (rejected by owner); run Better Auth inside our Rust backend (more code, more security surface); self-managed JWT auth (same).

## Consequences
- Simpler stack (one vendor for DB and auth) and per-branch data isolation.
- No local Neon Auth emulator, so dev/test rely on a fake; provider drift risk is mitigated by the smoke test.
- Fixed 15-minute lifetime and no custom claims constrain the design (room role is never in the token; it is read from Redis).
- Cold starts if scale-to-zero is enabled; avoid it in production.
- Open confirmations: `neon_auth.user.id` type, Rust Ed25519 JWKS library support (P2-1, P2-2).
