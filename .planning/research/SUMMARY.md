# Project Research Summary

**Project:** SpotOn — Spotify plugin for Lyrion Music Server
**Domain:** Streaming service plugin (Perl + Rust binary + Spotify Web API)
**Researched:** 2026-05-26
**Confidence:** HIGH

## Executive Summary

SpotOn is an LMS plugin that integrates Spotify streaming and Connect into the Lyrion Music Server ecosystem. All four research dimensions converge on the same architectural truth: the project is structurally similar to Herger's Spotty-Plugin but must diverge in three specific areas — authentication strategy (Keymaster-only via login5, no PKCE), audio transport (HTTP streaming over FIFO), and API module design (split, centralized vs. monolithic). LMS's single-threaded event loop is the dominant constraint: every design decision flows from it.

The recommended approach is a bottom-up layered build: plugin skeleton first, then auth + API foundation, then browse navigation, then single-track streaming, then Spotify Connect with FIFO transport, and finally HTTP transport as an upgrade. This order matches the dependency graph exactly.

The top systemic risks are: (1) Keymaster/login5 protocol instability — the only past occurrence caused a 24-hour outage for all librespot users; (2) Spotify Development Mode restrictions expanding further — Spotify restricted the API in November 2024 and February 2026, a third wave is plausible; (3) the Connect audio transport gap — FIFO has known UX defects that HTTP streaming solves, but HTTP streaming requires non-trivial binary work.

## Key Findings

### Stack

- **LMS Plugin API is stable.** Four core modules (`OPMLBased`, `SimpleAsyncHTTP`, `Cache`, `Prefs`) with clear APIs. ALL HTTP in server context must use `SimpleAsyncHTTP` — LMS is single-threaded, blocking calls kill audio.
- **librespot 0.8.0** is current upstream. Missing `--single-track`, `--lms`, `--get-token`, `--check` flags (Herger's fork). SpotOn requires its own fork (Plan B from AD-04).
- **Spotify API February 2026 removals:** Artist Top Tracks, Browse Categories, Related Artists, New Releases, batch endpoints — all removed for dev-mode apps. Search limit halved to 10/request. New unified library endpoints (`PUT/DELETE/GET /me/library`).
- **Made For You mixes accessible** via hardcoded category ID `0JQ5DAt0tbjZptfcdMSKl3` — Herger's undocumented trick, works in dev mode.

### Features

- **#1 user complaint:** Playback stops randomly (every third Spotty issue)
- **#2 user complaint:** 429 rate limit errors from parallel pagination
- **Spotty's biggest design mistake:** Liked Songs gated behind custom Client ID. SpotOn must expose unconditionally.
- **Connect is the emotional core:** "When Connect was disabled, my family stopped using multiroom entirely." (Spotty issue #224)
- **`recommendations` endpoint still works** in dev-mode — DSTM survives.

### Architecture

- Build order is strictly: Helper/AccountHelper → API/Auth+Client → OPML/Browse → ProtocolHandler+Transcoding → Connect/EventHandler+Manager+Daemon → HTTP streaming upgrade
- `API/Client.pm` as sole HTTP egress point with central throttle — non-negotiable
- Connect IPC is HTTP POST to LMS JSON-RPC — well-established pattern
- HTTP streaming server belongs in the binary, not Perl — from Perl's perspective it's just a standard HTTP radio URL

### Pitfalls (NEW — beyond P-01 through P-20)

- **P-21:** Keymaster already died once (August 2025, 24h outage). Binary must be >= 0.6.0 (login5).
- **P-22:** February 2026 API removals cripple Browse in Dev Mode. NAV-05, NAV-06 need redesign.
- **P-25/P-30:** Multi-daemon port collisions. Manager.pm must assign unique ports from pool.
- **P-26:** Token expiry at 1h silently degrades Connect. Proactive restart at 50 minutes.
- **P-32:** Transcoding table race — two players starting simultaneously overwrite each other's cache path.

## Top 5 Risks

| # | Risk | Impact | Mitigation |
|---|------|--------|------------|
| 1 | Keymaster/login5 protocol change | Total plugin failure | Pin binary >= 0.6.0; design Auth.pm with PKCE stub |
| 2 | Further Spotify API restrictions | Browse features break | Verify every endpoint against dev mode; graceful degradation |
| 3 | Connect token expiry (1h) | Silent session degradation | Proactive daemon restart at 50 min |
| 4 | Multi-daemon port collisions | Connect fails for multi-player | Manager.pm allocates unique ports from pool |
| 5 | Transcoding table race condition | Wrong audio for wrong player | Encode player params in URI, not global state |

## Suggested Phase Structure

| Phase | Name | Depends On | Research Needed |
|-------|------|------------|-----------------|
| 1 | Plugin Skeleton + Binary Foundation | — | No |
| 2 | Auth + API Foundation | Phase 1 | Yes (Keymaster binary interface) |
| 3 | Browse + Navigation | Phase 2 | No |
| 4 | Single-Track Streaming | Phase 2+3 | No |
| 5 | Spotify Connect (FIFO Transport) | Phase 4 | Yes (Connect state machine) |
| 6 | Polish + DSTM + Settings | Phase 3-5 | No |
| 7 | HTTP Streaming Transport (v2) | Phase 5 | Yes (binary HTTP server design) |

## Gaps to Address

- Keymaster login5 binary interface details — verify against actual forked binary
- Extended Quota Mode runtime detection — probe behavior needs live verification
- librespot issue #1377 (token expiry) status — monitor for resolution
- B&O format support matrix via UPnPBridge — needed for OGG-Direct defaults

---
*Research completed: 2026-05-26*
*Ready for roadmap: yes*
