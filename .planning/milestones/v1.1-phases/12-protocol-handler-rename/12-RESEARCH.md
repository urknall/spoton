# Phase 12: Protocol Handler Rename - Research

**Researched:** 2026-06-05
**Domain:** LMS Protocol Handler registration, Perl URL scheme substitution, Rust binary URI normalization
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-01:** Beim Plugin-Update: kompletter Cache-Flush aller `spoton_*` Keys. Kein granulares Vorgehen — sauberer Neustart.
**D-02:** Trigger via `cacheSchemaVersion` in Prefs. In `initPlugin()`: wenn Marker fehlt oder veraltet → alle `spoton_*` Cache-Einträge löschen, Marker auf neue Version setzen. Idempotent.
**D-03:** Keine automatische History-Migration. Alte `spotify://` Einträge in der LMS-DB werden ignoriert. Manueller Cleanup auf dev und raspi.
**D-04:** Importer.pm wird komplett auf `spoton://` umgestellt. Da Importer.pm nicht existiert, ist dieser Task ein No-Op.
**D-05:** Clean Break — nur `spoton://` wird akzeptiert. Kein Dual-Schema-Support, kein Fallback auf `spotify://`.
**D-06:** Spotify-API-URI (`spotify:track:ABC123`) ist nur Eingabe. Die LMS-URL (`spoton://track:ABC123`) ist ein internes Routing-Schema. Klare Trennung: API-URIs bleiben `spotify:`, LMS-URLs werden `spoton://`.
**D-07:** Akzeptanzkriterium: Beide Plugins (SpotOn + Spotty) können gleichzeitig in LMS aktiviert werden. Browse funktioniert in beiden unabhängig. Manueller Test auf raspi.
**D-08:** Kein shared state zwischen SpotOn und Spotty.
**D-09:** Spotty hat aktuell keinen Connect-Mode — keine Gerätenamen-Kollision.

### Claude's Discretion

- Reihenfolge der Datei-Änderungen
- Exakter Wert für `cacheSchemaVersion`
- Ob `_isDeadHistoryUrl` in Connect.pm das neue Schema-Pattern berücksichtigen muss
- Regex-Optimierungen

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PROTO-01 | All URL constructions in Plugin.pm, ProtocolHandler.pm, Connect.pm use `spoton://` scheme | Verified: 42 lines total need changing across 4 files — full location map in Architecture Patterns |
| PROTO-02 | ProtocolHandler registered as `spoton` not `spotify` in LMS | Verified: single line Plugin.pm:82, LMS `registerHandler` is a plain hash write keyed on protocol string |
| PROTO-03 | custom-convert.conf uses content types matching `spoton://` scheme | Verified: `son`/`soc` content types already SpotOn-specific — NO change needed. `$URL$` substitution passes full URL including scheme |
| PROTO-04 | Connect URLs use `spoton://connect-` prefix | Verified: 3 construction sites in Connect.pm (lines 630, 722, 828) |
| PROTO-05 | Spotty and SpotOn can be enabled simultaneously without URI handler conflict | Verified: LMS ProtocolHandlers uses separate hash keys per scheme; `spotify` and `spoton` are independent |
| PROTO-06 | Cached metadata under old `spotify://` keys invalidated or migrated on first start | Verified via LMS Cache internals: keys are stored as MD5 integers — prefix enumeration is not possible. Implementation: named cache namespace OR `cacheSchemaVersion` pref marker with natural expiry |
</phase_requirements>

---

## Summary

Phase 12 is a pure string-substitution refactor — no new libraries, no new architecture. Every occurrence of `spotify://` in LMS-internal URLs (routing URLs, not Spotify API URIs) must be replaced with `spoton://`. The ProtocolHandler registration key changes from `'spotify'` to `'spoton'`. There are no architectural changes to the pipeline.

**Critical non-obvious finding:** The librespot binary (`librespot-spoton/src/main.rs` line 667) has a hardcoded normalization: `track_uri.replace("spotify://", "spotify:")`. After the rename, LMS will pass `spoton://track:ID` via `$URL$` in `custom-convert.conf`. The binary will receive `spoton://track:ID` and its current normalization will fail silently, producing an invalid URI. **The Rust binary must also be modified and all platform binaries must be rebuilt.**

**Cache flush implementation:** The LMS `Slim::Utils::Cache` stores all keys as 60-bit MD5 integer hashes in SQLite (`Slim/Utils/DbCache.pm:_key()`). There is no prefix-based enumeration or deletion. Prefix-based "flush all `spoton_*` keys" is not directly achievable. Two clean implementations exist:
1. **Named namespace** (recommended): Switch all SpotOn modules to `Slim::Utils::Cache->new('spoton', $VERSION)`. Bumping `$VERSION` auto-triggers `$cache->clear()` on only the SpotOn SQLite db file (`spoton.db`), leaving all other LMS cache untouched.
2. **Natural expiry** (simpler but not D-01-compliant): Old `spotify://`-keyed entries are dead after the rename (never looked up under the new scheme) and expire within 7 days. No code required.

**Primary recommendation:** Modify Perl sources (4 files), add `cacheSchemaVersion` pref with named-namespace flush, modify Rust binary normalization line, rebuild all 6 platform binaries, register handler as `'spoton'`.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Protocol handler registration | LMS Plugin API (Plugin.pm initPlugin) | — | `registerHandler` maps URL scheme to handler class at startup |
| URL scheme in routing URLs | All Perl modules that construct/match URLs | Rust binary (receives URL via `$URL$`) | URL scheme is part of internal routing, not API wire format |
| URI normalization (scheme→Spotify format) | Rust binary (main.rs run_single_track) | — | Binary converts LMS routing URL back to Spotify API URI format |
| Cache key invalidation | Plugin.pm initPlugin | Slim::Utils::Cache | Cache keys are hashed; namespace version bump is the standard LMS pattern |
| LMS DB history entries | Not in scope (manual cleanup) | — | D-03: no code migration for history |

---

## Standard Stack

### Core (no new dependencies)

This phase installs no external packages. All required modules are already in the LMS bundle.

| Module | Purpose | Status |
|--------|---------|--------|
| `Slim::Player::ProtocolHandlers` | Handler registration | Already used — `registerHandler('spoton', ...)` |
| `Slim::Utils::Cache` | Cache with namespace + version | Already used — add named namespace support |
| `Slim::Utils::Prefs` | `cacheSchemaVersion` marker | Already used — add new pref key |

**Installation:** No new packages.

---

## Package Legitimacy Audit

No external packages are installed in this phase. Section not applicable.

---

## Architecture Patterns

### System Architecture Diagram

```
Spotify App → librespot (Connect) → spottyconnect JSON-RPC → Connect.pm
                                                                    ↓
                                                        spoton://connect-<ts> URL
                                                                    ↓
LMS Browse → Plugin.pm/_trackItem → spoton://track:ID URL → ProtocolHandler.pm
                                                                    ↓
                                                    custom-convert.conf $URL$ substitution
                                                                    ↓
                                            binary --single-track "spoton://track:ID"
                                                                    ↓
                                            normalize: replace("spoton://", "spotify:")
                                                                    ↓
                                            SpotifyUri::from_uri("spotify:track:ID") → PCM/OGG
```

### Recommended Project Structure

No structural changes. Modified files:

```
librespot-spoton/src/
└── main.rs              # Line 667: normalize "spoton://" → "spotify:"

Plugins/SpotOn/
├── Plugin.pm            # registerHandler('spoton'), URL constructions, initPlugin cache flush
├── ProtocolHandler.pm   # All spotify:// regex matches and URL strings
├── Connect.pm           # All spotify:// regex matches and connect-URL constructions
└── DontStopTheMusic.pm  # One URL construction (line 259)
```

### Pattern 1: Protocol Handler Registration Change

**What:** Single registration call in `Plugin.pm::initPlugin()`.
**When to use:** Only once — `initPlugin()`.

```perl
# BEFORE:
Slim::Player::ProtocolHandlers->registerHandler(
    'spotify',
    'Plugins::SpotOn::ProtocolHandler'
);

# AFTER:
Slim::Player::ProtocolHandlers->registerHandler(
    'spoton',
    'Plugins::SpotOn::ProtocolHandler'
);
```

**LMS mechanism:** `ProtocolHandlers.pm:registerHandler` stores `$protocolHandlers{'spoton'} = $class`. `handlerForURL` extracts scheme via `$url =~ /^([a-zA-Z0-9\-]+):/` and does `$protocolHandlers{lc $protocol}` lookup. `spoton` and `spotify` are independent hash entries — no collision possible. [VERIFIED: /usr/share/perl5/Slim/Player/ProtocolHandlers.pm lines 75-98]

### Pattern 2: URL Construction Change

**What:** All 5 construction sites where `spotify://` prefix is concatenated to a track path.

```perl
# Plugin.pm lines 418, 1155 — BEFORE:
my $spotify_url = 'spotify://' . $track_path;

# Plugin.pm lines 418, 1155 — AFTER:
my $spoton_url = 'spoton://' . $track_path;
# NOTE: variable rename is at Claude's discretion — functionally identical

# DontStopTheMusic.pm line 259 — BEFORE:
my $uri = "spotify://$1";

# DontStopTheMusic.pm line 259 — AFTER:
my $uri = "spoton://$1";

# Connect.pm lines 630, 722, 828 — BEFORE:
sprintf("spotify://connect-%u", $ts)

# Connect.pm lines 630, 722, 828 — AFTER:
sprintf("spoton://connect-%u", $ts)
```

### Pattern 3: Regex Match Changes

All regex patterns matching the LMS routing scheme must change. Pattern is mechanical: every `spotify://` in a regex becomes `spoton://`.

**Full match location map:**

| File | Line | Current | After |
|------|------|---------|-------|
| Plugin.pm | 182 | `m{^spotify://}` | `m{^spoton://}` |
| ProtocolHandler.pm | 36 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 65 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 95 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 209 | `m{^spotify://(?!connect-)}` | `m{^spoton://(?!connect-)}` |
| ProtocolHandler.pm | 224 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 257 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 267 | `"spotify://track:$1"` | `"spoton://track:$1"` |
| ProtocolHandler.pm | 298 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 306 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 316 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 337 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 348 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 353 | `"spotify://track:$trackId"` | `"spoton://track:$trackId"` |
| ProtocolHandler.pm | 371 | `s{^spotify:}{spotify://}` | `s{^spoton:}{spoton://}` (see Pitfall 3) |
| ProtocolHandler.pm | 403 | `m{spotify://track:}` | `m{spoton://track:}` |
| ProtocolHandler.pm | 426 | `m{spotify://track:([A-Za-z0-9]+)}` | `m{spoton://track:([A-Za-z0-9]+)}` |
| ProtocolHandler.pm | 428 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| ProtocolHandler.pm | 477-478 | `m{spotify://connect-}` / `"spotify://track:$trackId"` | `m{spoton://connect-}` / `"spoton://track:$trackId"` |
| Connect.pm | 107 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| Connect.pm | 285 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| Connect.pm | 592 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| Connect.pm | 598 | `m{spotify://connect-}` | `m{spoton://connect-}` |
| Connect.pm | 617 | `m{^spotify://} && !~ m{spotify://connect-}` | `m{^spoton://} && !~ m{spoton://connect-}` |
| Connect.pm | 695 | `m{^spotify://} && !~ m{spotify://connect-}` | `m{^spoton://} && !~ m{spoton://connect-}` |

**Lines that must NOT change** (Spotify API URIs per D-06):

| File | Lines | Value | Reason |
|------|-------|-------|--------|
| ProtocolHandler.pm | 266, 351, 431 | `m/^spotify:track:([A-Za-z0-9]+)$/` | `spotifyUri` field from Spotify API — always `spotify:track:` |
| Connect.pm | 624, 714, 771 | `"spotify:track:$trackId"` | `eventTrackUri` pluginData — stores Spotify API URI |
| Connect.pm | 918 | `spotifyUri => $trackInfo->{uri}` | Spotify API response field |

### Pattern 4: Cache Flush on Schema Version Change

**D-01/D-02 implementation using named namespace** (recommended):

```perl
# In each module — CHANGE cache initialization:
# BEFORE:
my $cache = Slim::Utils::Cache->new();

# AFTER (all 6 modules: Plugin.pm, ProtocolHandler.pm, Connect.pm,
#        DontStopTheMusic.pm, API/Client.pm, API/TokenManager.pm):
use constant SPOTON_CACHE_VERSION => 2;
my $cache = Slim::Utils::Cache->new('spoton', SPOTON_CACHE_VERSION);
```

This creates `spoton.db` (separate from the default `cache.db`). Bumping `SPOTON_CACHE_VERSION` from `1` (current state using default namespace) to `2` triggers automatic `clear()` on plugin load — standard LMS cache versioning pattern. [VERIFIED: /usr/share/perl5/Slim/Utils/Cache.pm lines 130-138]

**cacheSchemaVersion pref** (D-02) can be used as a belt-and-suspenders guard for the one-time version bump to version 2. After that, the namespace version constant handles future migrations.

**Alternative: natural expiry** — old `spotify://`-keyed entries become unreachable after rename (lookups use `spoton://` keys). They expire in ≤7 days. No code required. This satisfies the practical goal of D-01 but not the letter ("flush"). Planner should discuss with user if named-namespace approach is acceptable.

**Note on cacheSchemaVersion in prefs:**

```perl
# Plugin.pm::initPlugin() — add to $prefs->init():
$prefs->init({
    ...existing prefs...,
    cacheSchemaVersion => 0,  # default: migration not done
});

# In initPlugin(), after prefs init:
if ( ($prefs->get('cacheSchemaVersion') || 0) < SPOTON_CACHE_VERSION ) {
    $log->info("SpotOn cache schema version changed — cache cleared by namespace version bump");
    $prefs->set('cacheSchemaVersion', SPOTON_CACHE_VERSION);
}
# (Actual cache clear is handled by Slim::Utils::Cache->new('spoton', SPOTON_CACHE_VERSION))
```

### Pattern 5: Rust Binary Normalization Fix (CRITICAL)

**What:** `librespot-spoton/src/main.rs` function `run_single_track` receives the URL from `$URL$` substitution in `custom-convert.conf`. After rename, this will be `spoton://track:ID`. The current normalization converts `"spotify://"` → `"spotify:"` for `SpotifyUri::from_uri`. Must be updated to recognize the new scheme.

```rust
// src/main.rs line 666-667 — BEFORE:
// Normalize URI: LMS passes spotify://track:ID, librespot needs spotify:track:ID
let normalized_uri = track_uri.replace("spotify://", "spotify:");

// AFTER:
// Normalize URI: LMS passes spoton://track:ID, librespot needs spotify:track:ID
let normalized_uri = track_uri.replace("spoton://", "spotify:");
```

**Build implications:** After modifying `main.rs`, all 6 platform binaries must be rebuilt:
- 5 Linux targets via GitHub CI workflow (`workflow_dispatch` trigger on `.github/workflows/build-librespot.yml`)
- 1 Windows target (`x86_64-pc-windows-gnu`) via local `cross build` (not in CI)

The `$URL$` substitution passes `$self->streamUrl()` from `Song.pm:577` to `tokenizeConvertCommand2`, which sets `$subs{'URL'} = '"' . $fullpath . '"'`. After rename, `$self->streamUrl()` will return `spoton://track:ID`, which the binary must handle. [VERIFIED: /usr/share/perl5/Slim/Player/Song.pm:577, /usr/share/perl5/Slim/Player/TranscodingHelper.pm:586]

### Pattern 6: getFormatForURL Comment Update

`ProtocolHandler.pm` comment line 32 refers to `spotify://connect-*` — update to `spoton://connect-*` (cosmetic but keeps code consistent with D-06 rule).

### Anti-Patterns to Avoid

- **Changing `spotify:track:ID` Spotify API URIs:** The `spotifyUri` field, `eventTrackUri` pluginData values, and `SpotifyUri::from_uri()` call target all use `spotify:` prefix. These are Spotify protocol identifiers, not LMS routing URLs. Do NOT change these (D-06).
- **Touching `custom-convert.conf`:** Content types `son`/`soc` and the `$URL$` placeholder require no changes. The `soc pcm * *` pipeline's `-` passthrough is also unchanged.
- **Touching `custom-types.conf`:** `son`/`soc` content types are already SpotOn-specific and have no `spotify://` references.
- **Forgetting the binary rebuild:** The Rust normalization fix with no binary rebuild will cause silent failures — librespot receives `spoton://track:ID`, produces invalid URI, and exits non-zero. LMS will report the track as unplayable.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Cache prefix deletion | SQL LIKE query on LMS cache DB | `Slim::Utils::Cache->new('spoton', $VERSION)` version bump | LMS cache stores keys as 60-bit MD5 integers — no original key strings exist for prefix matching |
| Protocol handler conflict detection | Runtime scheme collision check | Separate registration keys (`spoton` vs `spotify`) | LMS ProtocolHandlers is a simple hash; separate keys cannot collide |

---

## Runtime State Inventory

This is a rename/migration phase. All 5 categories explicitly checked:

| Category | Items Found | Action Required |
|----------|-------------|-----------------|
| Stored data | LMS SQLite track DB (e.g., `squeezebox.db`): tracks table stores `url` field as `spotify://track:ID` for history entries | Code edit: no migration. D-03: manual DB cleanup on dev and raspi after deploy. Old entries become unplayable (expected). |
| Live service config | LMS `Slim::Utils::Cache` default namespace: `spoton_meta_*` entries keyed on `md5_hex("spotify://track:ID")` — unreachable after rename | Named namespace version bump (Pattern 4) triggers auto-clear; OR natural TTL expiry (7 days max) |
| OS-registered state | None — no cron jobs, Task Scheduler, launchd, or systemd units reference `spotify://` | None |
| Secrets/env vars | None — no `.env`, SOPS, or CI/CD variables reference the URL scheme string | None |
| Build artifacts | `librespot-spoton/target/` — stale build artifacts using old normalization; `Plugins/SpotOn/Bin/*/spoton` — pre-built binaries need replacement | Binary rebuild required (all 6 platforms); `cross build` or CI workflow dispatch |

---

## Common Pitfalls

### Pitfall 1: Forgetting the Rust Binary

**What goes wrong:** All Perl files are updated but the binary still runs `track_uri.replace("spotify://", "spotify:")`. When LMS passes `spoton://track:ID`, the normalization produces `spoton:track:ID` (not `spotify:track:ID`). `SpotifyUri::from_uri("spoton:track:ID")` fails. Binary exits non-zero. All Browse playback is broken.

**Why it happens:** The binary normalization is a single Rust line in a separate source tree not touched by Perl search-and-replace.

**How to avoid:** Include `librespot-spoton/src/main.rs` line 667 in the changeset. Trigger CI for Linux targets; do local cross-build for Windows.

**Warning signs:** LMS log shows `PROBLEM_OPENING` errors, binary exits non-zero, no audio on Browse tracks after deploy.

### Pitfall 2: Changing `spotify:track:` Spotify API URI Strings

**What goes wrong:** Overzealous replacement changes `spotifyUri => $trackInfo->{uri}` or `m/^spotify:track:([A-Za-z0-9]+)$/` matches. The Connect→Browse translation in `getMetadataFor` and `getNextTrack` fails because the cached `spotifyUri` field no longer matches the regex.

**Why it happens:** A blanket `s/spotify/spoton/g` replace across files hits both LMS routing URLs and Spotify API protocol strings.

**How to avoid:** Only replace occurrences of `spotify://` (double-slash). Never replace `spotify:track:`, `spotify:episode:`, or bare `spotify:` (colon notation for Spotify API URIs). Review each changed line.

**Warning signs:** Connect→Browse translation always fails. History replay (`_isDeadHistoryUrl`) never finds a match. `getNextTrack` falls through to error callback for history URLs.

### Pitfall 3: The Canonical Normalization in ProtocolHandler.pm Line 371

**What goes wrong:** Line 370-371 normalizes LMS-style colon notation to double-slash notation for cache key lookup:
```perl
if ($canonical && $canonical =~ m{^spotify:(?!//)}) {
    $canonical =~ s{^spotify:}{spotify://};
}
```
After rename, if this is left unchanged, it still normalizes `spotify:track:ID` → `spotify://track:ID`. But the cache keys are now `spoton://track:ID`. The normalization must become:
```perl
if ($canonical && $canonical =~ m{^spoton:(?!//)}) {
    $canonical =~ s{^spoton:}{spoton://};
}
```

**Why it happens:** LMS internally may strip the double-slash and pass `spoton:track:ID` to `getMetadataFor`. The normalization converts this back to `spoton://track:ID` for cache lookup.

**How to avoid:** Update line 370-371 as part of the ProtocolHandler.pm changeset.

**Warning signs:** NowPlaying metadata shows "Loading..." permanently for tracks that should have cache hits.

### Pitfall 4: Cache Namespace Shared Between Modules

**What goes wrong:** If the named namespace change (`Slim::Utils::Cache->new('spoton', VERSION)`) is applied to only some modules but not others, different modules use different cache namespaces and cache misses occur silently.

**Why it happens:** There are 6 files using `Slim::Utils::Cache->new()` without arguments.

**How to avoid:** Apply the named namespace change atomically across all 6 files in one plan task, or verify with grep after.

**Warning signs:** Token caches or API response caches miss after the change; API calls spike due to re-fetching cached data.

### Pitfall 5: LMS History Still Plays with Old `spotify://` Scheme

**What goes wrong:** After deploy, pressing play on a history entry with `spotify://track:ID` URL causes LMS to look up handler for scheme `spotify`, which is now owned by Spotty (if installed) or unregistered. Playback fails or routes to Spotty.

**Why it happens:** D-03 accepts this as known behavior. LMS stores track URLs in its database; no code migration updates them.

**How to avoid:** D-03 decision is already correct. Communicate to user: manual `UPDATE tracks SET url = REPLACE(url, 'spotify://', 'spoton://')` on dev and raspi LMS databases (or delete history). Provide SQL command in plan.

---

## Code Examples

### Rust Normalization (the critical change)

```rust
// librespot-spoton/src/main.rs — run_single_track function
// Source: /home/sti/spoton/librespot-spoton/src/main.rs line 666-670

// BEFORE:
// Normalize URI: LMS passes spotify://track:ID, librespot needs spotify:track:ID
let normalized_uri = track_uri.replace("spotify://", "spotify:");

// AFTER:
// Normalize URI: LMS passes spoton://track:ID, librespot needs spotify:track:ID
let normalized_uri = track_uri.replace("spoton://", "spotify:");

// SpotifyUri::from_uri validates the URI format; malformed URIs return Err (T-04.1-05)
let track_id = SpotifyUri::from_uri(&normalized_uri)?;
```

### Named Cache Namespace

```perl
# Source: /usr/share/perl5/Slim/Utils/Cache.pm lines 104-146
# The namespace version auto-clears on version mismatch.

# Applied to all 6 modules:
use constant SPOTON_CACHE_VERSION => 2;
my $cache = Slim::Utils::Cache->new('spoton', SPOTON_CACHE_VERSION);
```

### Canonical Normalization (ProtocolHandler.pm lines 370-371)

```perl
# Source: /home/sti/spoton/Plugins/SpotOn/ProtocolHandler.pm lines 368-371
# Normalize: cache is keyed on spoton://track:ID but LMS may pass spoton:track:ID

# BEFORE:
my $canonical = $url;
if ($canonical && $canonical =~ m{^spotify:(?!//)}) {
    $canonical =~ s{^spotify:}{spotify://};
}

# AFTER:
my $canonical = $url;
if ($canonical && $canonical =~ m{^spoton:(?!//)}) {
    $canonical =~ s{^spoton:}{spoton://};
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|-----------------|--------------|--------|
| `spotify://` as LMS routing scheme | `spoton://` as LMS routing scheme | Phase 12 | Enables simultaneous Spotty + SpotOn activation |
| Default LMS cache namespace | Named `spoton` cache namespace | Phase 12 | Allows atomic cache invalidation on schema version change |

**No deprecations in this phase.**

---

## Open Questions

1. **Named namespace vs. natural expiry for D-01**
   - What we know: D-01 requests "all spoton_* key flush." Prefix-based flush is not possible in LMS cache (keys are MD5 hashes). Named namespace version bump achieves the same goal (clears all SpotOn-owned cache entries) but requires changing all 6 `Cache->new()` calls. Natural expiry is simpler but doesn't satisfy D-01 literally.
   - What's unclear: User's priority — clean code boundary or minimal changeset?
   - Recommendation: Planner should default to the named namespace approach. It is the standard LMS pattern, has a clear mechanical implementation, and provides a clean separation. If user wants minimal diff, note that natural expiry is also acceptable.

2. **Windows binary rebuild path**
   - What we know: Windows binary (`x86_64-win64/spoton.exe`) is not included in the CI workflow. It was built via local `cross build --target x86_64-pc-windows-gnu`.
   - What's unclear: Is the local cross-rs toolchain still configured and working for Windows?
   - Recommendation: Include a local cross-build step for Windows binary as a plan task. CI handles all 5 Linux targets.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Perl 5 | Syntax check of modified .pm files | ✓ | 5.38.2 | — |
| cargo / Rust | Binary rebuild | ✓ | 1.96.0 | — |
| cross (cargo install cross) | Linux cross-compile | Assumed | — | Build only x86_64-linux locally if cross unavailable |
| LMS Perl modules | Test suite (t/05_perl_syntax.t) | ✓ | /usr/share/perl5/Slim/ | — |
| GitHub CI (build-librespot.yml) | Linux platform binaries | ✓ (workflow_dispatch) | — | Local musl build |

**Missing dependencies with no fallback:** None.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Test::More (Perl) |
| Config file | none — direct `prove` invocation |
| Quick run command | `prove t/05_perl_syntax.t t/03_convert_conf.t t/11_track_history.t` |
| Full suite command | `prove t/` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|--------------|
| PROTO-01 | All `spotify://` LMS URLs replaced with `spoton://` | unit (grep) | `prove t/12_protocol_rename.t` | ❌ Wave 0 |
| PROTO-02 | ProtocolHandler registered as `spoton` | unit (grep) | `prove t/12_protocol_rename.t` | ❌ Wave 0 |
| PROTO-03 | custom-convert.conf unchanged (content types `son`/`soc`) | unit | `prove t/03_convert_conf.t` | ✅ |
| PROTO-04 | Connect URLs use `spoton://connect-` prefix | unit (grep) | `prove t/12_protocol_rename.t` | ❌ Wave 0 |
| PROTO-05 | No remaining `registerHandler('spotify')` in SpotOn | unit (grep) | `prove t/12_protocol_rename.t` | ❌ Wave 0 |
| PROTO-06 | `cacheSchemaVersion` pref added to `initPlugin` | unit (grep) | `prove t/12_protocol_rename.t` | ❌ Wave 0 |
| PROTO-01 | Perl syntax still valid after rename | unit | `prove t/05_perl_syntax.t` | ✅ |
| All | History test still passes after rename | regression | `prove t/11_track_history.t` | ✅ |

### Sampling Rate

- **Per task commit:** `prove t/05_perl_syntax.t`
- **Per wave merge:** `prove t/`
- **Phase gate:** `prove t/` green before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `t/12_protocol_rename.t` — covers PROTO-01 through PROTO-06 via grep assertions on source files and binary --check

---

## Security Domain

> No new attack surface introduced. This is a string substitution of internal routing URLs. The URL scheme string is not user-controlled and not reachable from network input.

ASVS categories reviewed:

| ASVS Category | Applies | Rationale |
|---------------|---------|-----------|
| V5 Input Validation | No | URL scheme is a constant string, not user input |
| V6 Cryptography | No | No cryptographic changes |
| All others | No | Internal refactor only |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `Importer.pm` does not exist — D-04 is a no-op | Architecture Patterns | If Importer.pm was added in Phase 11 (which added library metadata features), it may need changes. Verify with `ls Plugins/SpotOn/Importer.pm`. |
| A2 | Windows cross-build toolchain is still functional locally | Environment Availability | Windows binary rebuild blocked; Windows users get broken playback after deploy |
| A3 | Named namespace approach is acceptable to user for D-01/D-02 | Architecture Patterns | If user wants the simpler natural-expiry approach, the named-namespace change across 6 files is unnecessary |

---

## Project Constraints (from CLAUDE.md)

- **Language:** Perl only for LMS plugin modules. No new dependencies.
- **No external CPAN deps:** `Slim::Utils::Cache` named namespace is a built-in LMS feature, not an external dep.
- **LMS Plugin API:** `ProtocolHandlers->registerHandler`, `Slim::Utils::Cache->new($namespace, $version)` — both are standard LMS API.
- **GSD Workflow Enforcement:** All changes via GSD execute-phase workflow.

---

## Sources

### Primary (HIGH confidence)
- `/usr/share/perl5/Slim/Player/ProtocolHandlers.pm` — `registerHandler` implementation, `handlerForURL` scheme extraction
- `/usr/share/perl5/Slim/Utils/Cache.pm` — namespace/version auto-clear pattern (lines 130-138)
- `/usr/share/perl5/Slim/Utils/DbCache.pm` — MD5 key storage (`_key()` line 152), `wipe()`/`clear()`, separate `.db` file per namespace
- `/usr/share/perl5/Slim/Player/Song.pm:577` — `$self->streamUrl()` passed as `$fullpath` to `tokenizeConvertCommand2`
- `/usr/share/perl5/Slim/Player/TranscodingHelper.pm:586` — `$subs{'URL'} = '"' . $fullpath . '"'` confirms `$URL$` gets the full routing URL
- `/home/sti/spoton/librespot-spoton/src/main.rs:666-670` — binary normalization logic
- `/home/sti/spoton/Plugins/SpotOn/Plugin.pm` — registerHandler location (line 82-83), URL constructions (lines 418, 1155, 182)
- `/home/sti/spoton/Plugins/SpotOn/ProtocolHandler.pm` — all 25 regex/string occurrences audited
- `/home/sti/spoton/Plugins/SpotOn/Connect.pm` — all 12 regex/string occurrences audited
- `/home/sti/spoton/Plugins/SpotOn/DontStopTheMusic.pm` — 1 URL construction (line 259)
- `/home/sti/spoton/.github/workflows/build-librespot.yml` — CI build matrix (5 Linux targets, no Windows)

### Secondary (MEDIUM confidence)
- `/home/sti/spoton/t/` — existing test suite baseline verified via `prove`
- `/home/sti/spoton/.planning/phases/12-protocol-handler-rename/12-DISCUSSION-LOG.md` — user decision audit trail

---

## Metadata

**Confidence breakdown:**
- Rename scope (which lines change): HIGH — full grep audit of codebase
- LMS ProtocolHandlers mechanism: HIGH — verified from installed LMS source
- Cache flush implementation: HIGH — verified from DbCache/Cache source
- Binary normalization impact: HIGH — read actual Rust source
- Binary rebuild complexity: MEDIUM — CI workflow verified; Windows local build assumed working

**Research date:** 2026-06-05
**Valid until:** 2026-07-05 (stable platform, no upstream dependencies)
