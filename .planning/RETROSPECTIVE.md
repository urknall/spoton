# Project Retrospective

*A living document updated after each milestone. Lessons feed forward into future planning.*

## Milestone: v1.0 — Foundation

**Shipped:** 2026-06-03
**Phases:** 15 | **Plans:** 50

### What Was Built
- Complete Spotify plugin from scratch: Browse, Search, Library, Connect, DSTM, Settings
- ZeroConf + Dual-Token auth (one-click setup)
- 5-option format dropdown with per-player prefs
- 62 requirements fulfilled

### What Worked
- Pitfall documentation (P-01 through P-20) from Spotty-NG prevented re-learning expensive lessons
- Inserted decimal phases (04.1, 04.2, etc.) kept scope manageable without replanning
- HTTP-streaming decision for Connect audio eliminated FIFO pain

### What Was Inefficient
- Multiple auth pivots (PKCE → ZeroConf → Keymaster) before finding the right approach
- Connect phases required 5 sub-phases (05.1–05.4) — initial estimate was 1 phase

### Patterns Established
- Decimal phase insertion for scope creep within a milestone
- Wave-based plan execution (dependencies between plans)
- TDD for metadata and utility functions

### Key Lessons
1. librespot credential management is opaque — ZeroConf is the only reliable auth path
2. Connect mode is fundamentally different from Browse mode — plan for separate phases
3. Sync groups need dedicated testing — discovered daemon duplication bug only in multi-player setup

---

## Milestone: v1.1 — Hardening & Reach

**Shipped:** 2026-06-06
**Phases:** 7 | **Plans:** 13

### What Was Built
- Multi-arch binaries (6 Linux musl-static + Windows GNU via cross-rs)
- Connect-DSTM via Spirc-native autoplay
- Stream metadata in Songinfo
- Production deployment (GitHub, repo.xml, Pi monitoring)
- Track history with async re-fetch and Connect-to-Browse translation
- spoton:// protocol handler rename for Spotty coexistence

### What Worked
- cross-rs Docker targets made multi-arch compilation reliable and repeatable
- Spirc-native autoplay was far simpler than LMS-side DSTM — the spike validated the approach before committing
- Phase 12 (protocol rename) was requested by Herger and shipped same day — good community response

### What Was Inefficient
- REQUIREMENTS.md traceability fell out of sync — many items marked "Pending" that were actually complete
- Phase summaries had inconsistent one-liner extraction (some returned verifier rule violations)
- macOS binaries (ARCH-05/06) blocked on missing native CI runner — should have been flagged as deferred earlier

### Patterns Established
- Binary rebuild as a separate plan (wave 2) whenever Rust source changes
- 7-day TTL for metadata cache — balances freshness with API quota
- `--check` JSON manifest for binary capability discovery

### Key Lessons
1. Connect daemon shares cache dir with Browse token — multi-user household exposes credential overwrite bug (discovered post-ship)
2. Slim::Utils::Cache is SQLite-backed and persists across LMS restarts — in-memory assumptions about cache clearing are wrong
3. Spike-gated phases (Phase 10) save massive rework — validate unknowns before committing architecture

---

## Cross-Milestone Trends

### Process Evolution

| Milestone | Phases | Plans | Days | Key Change |
|-----------|--------|-------|------|------------|
| v1.0 | 15 | 50 | 9 | Established GSD workflow, decimal phase insertion |
| v1.1 | 7 | 13 | 3 | Spike-gated phases, wave-based binary rebuilds |

### Cumulative Quality

| Milestone | Tests | LOC (Perl) | LOC (Tests) |
|-----------|-------|------------|-------------|
| v1.0 | ~180 | ~5.000 | ~2.500 |
| v1.1 | 230 | 5.559 | 3.654 |

### Top Lessons (Verified Across Milestones)

1. librespot credential/session management requires defensive design — every assumption about credential persistence will eventually break
2. Decimal phase insertion is better than scope inflation — keeps original phase goals clean
3. Binary rebuilds are mechanical but critical — always test `--check` output after rebuild
