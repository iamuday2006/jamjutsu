# Audio Architecture Evaluation

Status: **Decision proposed in ADR-0003 (requires owner sign-off).** Evidence gaps are listed in section 6; resolve them with spikes before building SFU features.

## 1. Problem

The host picks a track; all participants must hear it in near-sync. Two fundamentally different models exist:

- **Model A, Synchronized playback:** every client plays the same licensed stream URL itself, driven by a shared playback clock (state + server time). No audio flows through our infrastructure.
- **Model B, Audio broadcasting:** the host's browser (or a server) captures/decodes audio and publishes it as a live audio track through WebRTC; participants subscribe to that single stream.

## 2. Options evaluated

### 2.1 Plain WebRTC mesh
Host connects peer-to-peer to each participant. Host upload scales linearly with participants; NAT traversal needs STUN/TURN; unsuitable beyond a handful of participants. **Rejected.**

### 2.2 SFU architecture (general)
Host publishes once; the SFU forwards to N subscribers. Scales server-side, handles congestion and reconnection. Cost: infrastructure, operational complexity, and bandwidth egress.

### 2.3 LiveKit (open-source SFU, cloud or self-hosted)
- Open source server; cloud and self-hosted options.
- **Rust support:** `livekit-api` (server APIs, access-token generation) and `livekit` (real-time SDK, Tokio required). Backend can mint room tokens in Rust with fine-grained grants (publish only for host, subscribe only for participants), which maps directly to BR-03.
- Browser SDK for capture/publish/subscribe; reconnection and connection management built in.
- Server-side publishing from Rust is possible via the real-time SDK (there are examples like publishing audio from a file), which would allow a "server radio" mode, but the Rust real-time SDK is heavier (native libwebrtc bindings) and a reported stereo-publishing issue (rust-sdks #1018) means stereo must be verified.
- Token-based auth fits our JWT model: backend validates the Neon Auth user, checks host/participant role, then issues a LiveKit token.

### 2.4 Audio capture options for the host (Model B)
| Method | Notes |
|---|---|
| `getDisplayMedia` tab audio | User-gesture + permission prompt; browser support varies (best in Chromium); poor UX. |
| `HTMLAudioElement.captureStream()` / Web Audio `MediaStreamAudioDestinationNode` | Clean for our own player. **Cross-origin audio without CORS headers is tainted/silent** → depends on Jamendo CDN CORS behavior (unverified). |
| Server-side decode + publish | Backend downloads and decodes track, publishes via LiveKit Rust SDK. Raises licensing questions (server-side copying/redistribution) and adds CPU/bandwidth cost. |

### 2.5 Participant connection management
Model B needs: token issuance, per-participant subscribe permission, reconnect handling, TURN fallback, room-lifecycle sync (delete LiveKit room on BR-07). Model A needs none of this.

## 3. Comparison

| Criterion | A: Sync playback | B: LiveKit SFU broadcast |
|---|---|---|
| Infra cost | Very low | Medium to high (SFU + egress) |
| Complexity | Low | High |
| Audio quality | Original stream quality | Opus re-encode, bitrate-limited |
| Sync accuracy | Depends on clock sync (typ. 50-250 ms with correction) | Inherently one stream; small jitter differences |
| Host failure | Participants keep local playback until state says otherwise | Audio stops with host |
| Licensing posture | Each listener streams from the licensor's own endpoint (easier to reason about) | Our system redistributes audio (higher legal risk) |
| Works with non-streamable sources | No | Yes (host-side capture) |
| Scale | Excellent | Depends on SFU capacity |

## 4. Recommendation (ADR-0003)

1. **MVP: Model A (synchronized playback)** with Jamendo stream URLs. It satisfies the product ("everyone hears the same track together, host controls it") at the lowest cost and risk, and keeps audio delivery with the licensor's CDN.
2. **Keep Model B as a gated phase** behind a feature flag, implemented with **LiveKit**, only if (a) Spike S2 shows tracks can be captured legitimately, and (b) licensing review (LICENSING.md) confirms broadcast redistribution is allowed. Backend work for B is mostly `livekit-api` token minting plus room lifecycle hooks, so the architecture does not need rework to add it.
3. Do **not** build an in-house SFU.

## 5. Synchronized playback design (Model A)

State (Redis `playback` hash): `track_id`, `track_url`, `status` (`playing`|`paused`|`idle`), `position_ms` (position at `updated_at_ms`), `updated_at_ms` (server clock), `seq`.

Clients:
1. On connect, run time sync: send `time.sync {client_t0}` → server replies `{client_t0, server_t}`; compute offset using RTT/2; repeat 3-5 times, take the lowest-RTT sample.
2. When playing, target position = `position_ms + (serverNow() - updated_at_ms)`.
3. Compare with `audio.currentTime`; if drift > 300 ms, seek; if 50-300 ms, adjust `playbackRate` slightly (0.97-1.03) until aligned; re-check every 5 s.
4. Autoplay policy: browsers require a user gesture; show "Tap to join audio" before first play.
5. Track end: host client sends `playback.set_track` for the next track (no server queue persisted, BR-10). Optional host-local queue lives only in host's browser memory.

## 6. Spikes required before dependent work

| ID | Question | Output |
|---|---|---|
| S1 | Does Jamendo's CDN serve audio with CORS headers allowing `captureStream`/Web Audio? Are stream URLs time-limited or hotlink-restricted? | Findings appended here |
| S2 | Can host capture produce clean audio in Chrome/Firefox/Safari? | Compatibility matrix |
| S3 | Does Jamendo (licensing@jamendo.com) permit server-mediated rebroadcast and public group listening? | Written answer in LICENSING.md |
| S4 | LiveKit self-host vs cloud cost at target scale (e.g., 100 rooms x 20 listeners) | Cost table |
| S5 | Measured sync drift of Model A across browsers/networks | Report and tuned constants |
