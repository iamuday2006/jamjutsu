# Security

## 1. Trust boundaries

Browser (untrusted) → Backend (trusted, enforces everything) → Redis/Postgres/Jamendo. **Authorization decisions are made only on the server from server-held state.**

## 2. Authentication

- Neon Auth (managed Better Auth) issues JWTs. The backend verifies signature, `exp`, `iss`, `aud` on every REST request and on WebSocket `auth` / `auth.refresh`.
- **Verification facts (per Neon docs, verified 2026-11-21):** keys come from `<NEON_AUTH_URL>/.well-known/jwks.json`; signing algorithm is fixed to **EdDSA (Ed25519)**; `iss` and `aud` both equal the auth URL **origin** (scheme + host, no path); `sub` is the user id; lifetime is a fixed **15 minutes**; custom claims are not supported; claims include user fields (email, name, role, ban status) and an active-organization claim `o` (unused by Jam Room).
- Implementation requirements: pin the allowed algorithm to EdDSA (never accept `none` or algorithm switching); cache the JWKS with a TTL and refetch once on unknown `kid` (rate-limit refetches to prevent abuse); allow small clock skew (<= 30 s); reject banned users; confirm the chosen Rust JWT library handles Ed25519/OKP JWKs at P2-1 and record it in the ADR. Neon's own verification examples are explicitly not production-ready, so do not copy them without caching and error handling.
- Browsers authenticate to Neon Auth with an HTTP-only session cookie; our API is a separate origin, which is the case where Neon says to use the JWT instead. The JWT is held in memory only (never localStorage) and refreshed via the SDK.
- Do not use the user's `email` as an identifier or log it; `sub` is the identity. Display names come from the `name` claim and are rendered as text only.
- Never log tokens. Tokens are sent in the `Authorization` header (REST) or first WS message, never in URLs.
- Service-role keys are server-only and never exposed to the frontend.

## 3. Authorization matrix

| Action | Host | Participant | Anonymous |
|---|---|---|---|
| Create room | yes (authenticated) | yes | no |
| Join room | yes | yes | no |
| Playback commands | **yes** | **no (BR-04)** | no |
| End room | yes | no | no |
| Leave room | yes | yes | no |

Host identity is read from Redis `room:{code}:meta.host_id`, compared with the verified JWT `sub`.

## 4. Threat model (STRIDE-lite)

| Threat | Mitigation |
|---|---|
| Spoofed host commands | Server-side role check inside the Lua script; tests for BR-03/04 |
| Room code guessing | 8-char CSPRNG codes, rate limiting on join/lookup, uniform `ROOM_NOT_FOUND` behavior |
| Token replay / expiry | Verify exp, enforce close at expiry, 15-minute tokens, `auth.refresh` must keep the same `sub` |
| Forged tokens / algorithm confusion | Pin EdDSA, verify against JWKS only, check `iss`/`aud` origin |
| Compromised auth data in Postgres | Application role cannot write `neon_auth`; Data API not enabled |
| WS flooding | Per-connection rate limit, size cap, connection caps per user/IP |
| Malicious audio URL injection | Clients send track IDs only; server resolves from allowlisted source |
| XSS via display names/track titles | Render as text, never HTML; CSP; sanitize on output |
| CSRF | Bearer tokens in headers (no cookie auth); strict CORS allowlist; WS `Origin` check |
| Secret leakage | Env-only secrets, `.env` gitignored, secret scanning in CI |
| Dependency compromise | `cargo audit`/`cargo deny`, `npm audit`, Dependabot, lockfiles committed |
| Redis exposure | Not publicly exposed; password/TLS in non-local envs; network isolation |
| DoS via room creation | One hosted room per user, per-user rate limit, capacity caps |
| Privacy | Store minimal metadata; no playback history; no PII beyond the Neon Auth profile (managed by Neon) and display name |

## 5. Hardening checklist for every PR

- [ ] New inputs validated (type, size, range).
- [ ] AuthN/AuthZ enforced server-side; test exists for the negative case.
- [ ] No secrets or tokens in logs/errors.
- [ ] Errors do not leak internals.
- [ ] New dependencies justified, license-compatible, audited.
- [ ] Rate limits considered.
- [ ] Docs/STATUS updated.

## 6. HTTP security

Security headers (CSP, `X-Content-Type-Options`, `Referrer-Policy`, HSTS in prod), TLS terminated at the edge, WebSocket `wss://` only in non-local environments, request body size limits, timeouts on every outbound call (Jamendo, Postgres, Redis).
