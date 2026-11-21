# ADR-0001: Modular monolith

Status: Accepted

## Context
Small team of AI agents; need fast iteration and simple operations. Real-time rooms share tightly coupled state.

## Decision
One Rust binary with internal modules and enforced dependency directions (ARCHITECTURE.md). One React SPA.

## Alternatives
Microservices (auth, rooms, realtime, tracks): rejected as premature; adds network failure modes, deployment and testing overhead.

## Consequences
Simple deploy/debug. Must police module boundaries by convention and review. Extraction later requires a new ADR with evidence.
