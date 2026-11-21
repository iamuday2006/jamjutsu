# ADR-0006: Jamendo as the initial music source; no YouTube extraction

Status: Accepted for non-commercial operation

## Context
YouTube/YouTube Music do not grant unrestricted extraction or redistribution. Jamendo offers a REST API over Creative Commons tracks, free for non-commercial use per its API Terms; commercial use needs a quote from licensing@jamendo.com; each track has its own CC license (LICENSING.md).

## Decision
Use Jamendo for the initial implementation, with per-track license evaluation, mandatory attribution, server-side-only API credentials, and `COMMERCIAL_MODE` filtering. Forbid YouTube extraction in code and dependencies.

## Alternatives
YouTube (legal risk); user-uploaded audio (copyright and moderation burden); other CC catalogs (candidates for future ADRs).

## Consequences
Legally conservative start. Catalog quality and variety limited. Public/commercial launch is blocked until Jamendo licensing and rebroadcast questions are answered (spike S3).
