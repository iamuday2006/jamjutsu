# ADR-0003: Audio delivery model

Status: **Proposed** (owner sign-off requested; implementation may proceed with Model A as default)

## Context
Host-to-participant audio can be delivered by (A) synchronized client-side playback of a licensed stream URL or (B) broadcasting host audio through WebRTC/SFU (LiveKit). See AUDIO_ARCHITECTURE.md for the evaluation. Jamendo's API terms limit use to non-commercial and tracks carry per-track CC licenses; rebroadcast permission is unconfirmed (spike S3). CORS behavior of Jamendo audio for browser capture is unverified (S1).

## Decision
MVP uses **Model A**: server holds playback state and clock; clients stream audio directly from the licensor and stay in sync via time-sync and drift correction. **Model B (LiveKit SFU)** is deferred behind a feature flag and gated on spikes S1-S4 and licensing review. No in-house SFU.

## Alternatives
WebRTC mesh (does not scale); server-side decode and publish (licensing and cost risk); starting with LiveKit (complexity and legal risk before value is proven).

## Consequences
Low cost and complexity, original audio quality, audio delivery stays with the licensor. Sync quality depends on client clock handling (spike S5). Cannot play sources that lack a directly streamable URL. Backend design stays compatible with adding LiveKit token minting (`livekit-api`) later.
