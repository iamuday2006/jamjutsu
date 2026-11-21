# Music Licensing and Source Policy

> Not legal advice. Have a lawyer review before any public or commercial launch.

## 1. Hard rules

1. **No YouTube / YouTube Music audio extraction, proxying, restreaming, or downloading.** Platform terms do not grant unrestricted extraction or redistribution. Do not add libraries or code for this.
2. Only play tracks from a source with an explicit licensing basis recorded here.
3. Always display required attribution for each track (title, artist, license, link).
4. License must be evaluated **per track**, never assumed catalog-wide.

## 2. Initial source: Jamendo

Verified 2026-11-21 from Jamendo's public documentation:

| Topic | Finding |
|---|---|
| API use | Jamendo API Terms of Use: the API may be used freely for non-commercial uses. Any other use, including commercial, requires contacting licensing@jamendo.com for a quote. |
| Future changes | Jamendo reserves the right to charge for API use or insert ads in the future, and may monitor API usage for compliance. |
| Catalog licensing | Tracks are Creative Commons licensed. Terms differ per track: some permit commercial use, some are NC (non-commercial), some require share-alike; all require attribution. The API exposes a per-track license field. |
| Rate limit | Third-party documentation cites up to 35,000 requests/month for non-commercial apps; confirm in the Jamendo developer portal. |
| Auth | `client_id` query parameter from the Jamendo developer console; keep it server-side. |
| Commercial path | Jamendo Licensing (separate service) sells commercial licenses; Jamendo Music itself is positioned for personal use. |

### Implications for Jam Room

- **Non-commercial / hobby / portfolio deployment:** allowed under the API terms, with attribution and license-aware filtering.
- **Commercial deployment (ads, subscriptions, paid tiers):** requires a Jamendo licensing agreement first. Treat as **blocked** (PRODUCT.md OQ-3).
- **Group listening / rebroadcast:** whether a synchronized group session is a personal use or a public performance, and whether server-mediated audio is permitted, is **unresolved**; see spike S3. Model A (each client streams from Jamendo directly) is the more conservative posture.

## 3. Implementation requirements

- The `tracks` module normalizes a Jamendo track into `Track { id, title, artist, album, duration_s, audio_url, license: LicenseInfo, attribution_url }`.
- `LicenseInfo` parses Creative Commons license URL into flags: `commercial_allowed`, `share_alike`, `no_derivatives`.
- Config `COMMERCIAL_MODE=false` (default) allows NC tracks; `true` filters to `commercial_allowed` only. (BR-15)
- Search endpoint must be license-aware and must always return attribution fields.
- Never proxy audio bytes through our servers in Model A.
- The Jamendo `client_id` never ships to the browser; the backend proxies metadata calls.
- Cache metadata only as terms allow; default no persistent cache; short in-memory TTL (<= 5 min).
- UI shows an attribution panel for the current track.

## 4. Adding another source

Requires an ADR that records: license basis, API terms, attribution rules, commercial status, rate limits, and a review of redistribution rights.
