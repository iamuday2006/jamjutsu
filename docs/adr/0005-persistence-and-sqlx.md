# ADR-0005: SQLx against Supabase Postgres

Status: Accepted; **partially superseded by ADR-0007** (the database host is now Neon, not Supabase; the SQLx decision itself stands)

## Context
Need async, type-checked Postgres access from Rust with explicit SQL and migrations.

## Decision
Use SQLx with compile-time-checked queries (offline metadata committed) and forward-only migrations. Backend connects directly to Supabase Postgres (pooler or direct, chosen at P2-2 after verifying prepared-statement compatibility with the pooler mode). Authorization is enforced in application code; RLS is enabled with no permissive client policies.

## Alternatives
supabase-rs/PostgREST client (less type safety, extra hop); Diesel/SeaORM (heavier abstractions, against "avoid unnecessary abstractions").

## Consequences
Full SQL control and testability with a local Postgres container. Requires keeping `.sqlx` metadata current in CI.
