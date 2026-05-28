---
phase: 03-browse-navigation
verified: 2026-05-28T16:00:00Z
status: human_needed
score: 9/11 must-haves verified
overrides_applied: 0
gaps:
  - truth: "Album page 2+ shows track artwork (album images passed through passthrough)"
    status: failed
    reason: "_albumItem only puts albumId in passthrough. _albumFeed page >0 reads passthrough->{albumImages} which is always undef because _albumItem never sets it. _albumTrackItem then calls _largestImage(undef) -> '' -> no artwork on page 2+ of multi-page albums (>50 tracks)."
    artifacts:
      - path: "Plugins/SpotOn/Plugin.pm"
        issue: "_albumItem passthrough only contains {albumId}, missing albumImages and albumArtist. Lines 291-298 vs _albumFeed lines 790-791."
    missing:
      - "In _albumItem: passthrough => [{ albumId => $album->{id}, albumImages => $album->{images}, albumArtist => $firstArtist }]"
      - "This is CR-04 from the code review (03-REVIEW.md) — not addressed by the fix commits (735148d, c4f7254)"
  - truth: "Search pagination handles limit=10 per request (NAV-11)"
    status: failed
    reason: "_searchTypeFeed is the paginated drill-down feed for search type results. Per fix commit c4f7254, it now hardcodes offset=0 and limit=10 and returns no total. This means search results are capped at 10 items with no way to load more. NAV-11 requires search pagination to handle limit=10 per request — the API cap is respected but multi-page loading is disabled. This was a deliberate choice for the LMS XMLBrowser sub-item bug, but it means NAV-11 (pagination) is structurally incomplete."
    artifacts:
      - path: "Plugins/SpotOn/Plugin.pm"
        issue: "_searchTypeFeed: offset hardcoded to 0, no total returned (lines 661-701). LMS cannot request subsequent pages."
    missing:
      - "NAV-11 requires pagination. The current implementation shows max 10 results per type with no way to page forward. This conflicts with NAV-11's intent even if the Dev Mode limit=10 per request is respected."
      - "If the LMS XMLBrowser sub-item bug applies here, this may be an accepted limitation. Requires human decision."
human_verification:
  - test: "Top-level SpotOn menu shows Home, Search, Library after account switcher"
    expected: "Account Switcher item, then Home, Search (with text input), Library as three navigable items"
    why_human: "LMS menu rendering and type='search' behavior cannot be verified without a running LMS instance"
  - test: "Home -> Recently Played shows track list with artwork"
    expected: "List of recently played tracks with album art, artist name, and track title"
    why_human: "Requires live Spotify API call with valid OAuth token"
  - test: "Home -> Made For You shows Spotify-generated playlists (Discover Weekly, Daily Mix, etc.)"
    expected: "Playlists owned by 'spotify' appear; user-created playlists do not appear"
    why_human: "Depends on user's Spotify account content and the owner.id='spotify' filter working correctly at runtime"
  - test: "Home -> Top Tracks shows user's top tracks (medium_term)"
    expected: "User's listening history top tracks list"
    why_human: "Requires live Spotify API call"
  - test: "Library -> Liked Songs shows paginated track list"
    expected: "User's saved tracks in recently-added order with track/artist/album metadata"
    why_human: "Requires live Spotify API, pagination verification needs actual scroll"
  - test: "Library -> Playlists does NOT show Made-For-You playlists"
    expected: "Only user-created playlists, no Discover Weekly or Daily Mix"
    why_human: "Filter correctness requires runtime verification with actual playlist data"
  - test: "Search 'Radiohead' returns Top Result + Tracks/Albums/Artists/Playlists sections"
    expected: "Top Result (first track), then 4 category sub-menus with result counts"
    why_human: "Requires live search API call and LMS menu rendering"
  - test: "Artist detail shows 4 sections: Albums, Singles, Compilations, Appears On — NO Top Tracks"
    expected: "Exactly 4 navigable sections; no 'Top Tracks' or 'Related Artists' menu items"
    why_human: "LMS menu rendering verification"
  - test: "Album detail shows track numbers and duration"
    expected: "line1 = '1. Track Name', duration visible, track is playable via spotify:// URI"
    why_human: "Requires LMS rendering and interaction verification"
  - test: "Track context navigation: 'View Artist' and 'View Album' appear in context menu"
    expected: "Long-press or context action on a track shows artist and album navigation links"
    why_human: "LMS context menu behavior varies by controller type (Jive, web UI, iPeng)"
  - test: "Re-authentication required after scope change (user-follow-read added)"
    expected: "Existing sessions with old tokens get 403 on /me/following; no scope mismatch prompt exists (WR-03 known gap)"
    why_human: "Scope detection gap (WR-03) means re-auth is not prompted automatically — verify 403 fallback shows NO_RESULTS gracefully"
---

# Phase 3: Browse + Navigation Verification Report

**Phase Goal:** Users can navigate the full Spotify content hierarchy — Home, Search, Library — via LMS OPML menus
**Verified:** 2026-05-28T16:00:00Z
**Status:** human_needed (1 confirmed code gap, 1 contested gap requiring human decision, 11 human verification items)
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|---------|
| 1 | Top-level SpotOn menu shows Home, Search, and Library; all three are navigable | ? UNCERTAIN | Code wired: handleFeed pushes Home/Search/Library items inside `if ($activeName)` block (Plugin.pm lines 127-141). Search uses type='search'. Functional verification requires running LMS. |
| 2 | Home displays Recently Played items and at least one Made For You mix | ? UNCERTAIN | _homeFeed returns 3 items. _recentlyPlayedFeed calls getRecentlyPlayed(limit=50). _madeForYouFeed filters getUserPlaylists by owner.id='spotify'. Correctness depends on account data. |
| 3 | Liked Songs appears in Library without special configuration | ✓ VERIFIED | _libraryFeed includes _savedTracksFeed unconditionally. _savedTracksFeed calls getSavedTracks with offset/limit pagination and returns total. No gating code present. (NAV-08) |
| 4 | Searching "Radiohead" returns results grouped into Tracks, Albums, Artists, Playlists sections | ? UNCERTAIN | _searchFeed calls Client->search with type='track,album,artist,playlist'. Categories with 0 results skipped. Top Result from first track. Structure correct; result depends on live API. |
| 5 | Navigating into an artist shows Albums/Singles/Compilations/Appears On | ✓ VERIFIED | _artistFeed returns exactly 4 items pointing to _artistAlbumsFeed with single includeGroups value each: album, single, compilation, appears_on. No Top Tracks, no Related Artists. (NAV-05, D-09) |
| 6 | Navigating into an album shows paginated tracklist with track number, duration, and featuring artists | ✓ VERIFIED (with gap) | _albumFeed: page 1 uses getAlbum (embeds tracks), page 2+ uses getAlbumTracks with correct offset. _albumTrackItem builds "N. Title" in line1. Duration from duration_ms/1000. Featuring artists shown when different from album artist. BUT: page 2+ loses artwork (CR-04 gap). |
| 7 | Dev Mode removed endpoints (Artist Top Tracks, Related Artists, Browse Categories, New Releases) are silently hidden | ✓ VERIFIED | No menu items created for these endpoints anywhere in Plugin.pm. No API methods for these paths in Client.pm. Comment at Plugin.pm line 710 explicitly confirms omission. (NAV-10, D-11) |
| 8 | All 12 Spotify API methods exist in Client.pm following _request() pipeline | ✓ VERIFIED | All 12 subs present: search, getRecentlyPlayed, getTopTracks, getSavedTracks, getSavedAlbums, getFollowedArtists, getUserPlaylists, getArtist, getArtistAlbums, getAlbum, getAlbumTracks, getPlaylistItems. Each calls _request('get', path, params, $cb). |
| 9 | TokenManager includes user-follow-read and playlist-read-collaborative scopes | ✓ VERIFIED | TokenManager.pm lines 32-33 show both scopes in REQUIRED_SCOPES constant. Used in startOAuthFlow. |
| 10 | All 18 i18n strings for Phase 3 exist in EN and DE | ✓ VERIFIED | All 18 keys verified present in strings.txt: 16 base keys + PLUGIN_SPOTON_ARTIST_VIEW + PLUGIN_SPOTON_ALBUM_VIEW. German translations use proper UTF-8 umlauts (Kürzlich, Für, Künstler, Gefällt). SON entry remains last. |
| 11 | Search pagination handles limit=10 per Dev Mode constraint (NAV-11) | ✗ FAILED | _searchTypeFeed hardcodes offset=0 and returns no total. Deliberate per c4f7254 commit (LMS XMLBrowser bug workaround), but means search type results are capped at max 10 with no pagination. NAV-11 intent (handle limit=10 per request) is technically satisfied at the API level but pagination is functionally disabled. See gap section. |

**Score: 9/11 truths verified** (6 VERIFIED, 3 UNCERTAIN pending human test, 2 FAILED)

---

### Known Limitations (not bugs per submission brief)

The following deliberate design choices are flagged as known limitations, not gaps:

- **Lists limited to 50 items** for navigable sub-item feeds (_savedAlbumsFeed, _userPlaylistsFeed, _artistAlbumsFeed): hardcoded offset=0, limit=50, no total returned. This is the LMS XMLBrowser sub-item resolution bug workaround (commit c4f7254). Feeds with navigable sub-items (link type) cannot use LMS pagination without breaking sub-item navigation.
- **getArtist() unused**: Client.pm has getArtist() method that is never called in Plugin.pm. _artistFeed shows discography sections without calling getArtist for artist metadata. This is WR-01 from the code review — dead code, not a blocker.
- **Made For You may be empty**: _madeForYouFeed fetches first 50 playlists and filters. Users with no Spotify-generated playlists or with all Made-For-You content beyond position 50 will see empty results.

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/API/Client.pm` | 12 new API endpoint methods | ✓ VERIFIED | All 12 methods present, follow _request() pattern. Committed as 5f096aa. |
| `Plugins/SpotOn/API/TokenManager.pm` | user-follow-read + playlist-read-collaborative scopes | ✓ VERIFIED | Both scopes in REQUIRED_SCOPES constant (lines 32-33). Committed as 71ddc1d. |
| `Plugins/SpotOn/strings.txt` | 18 new i18n keys (16 + 2 context nav) | ✓ VERIFIED | 53 total PLUGIN_SPOTON_ entries. All 18 Phase 3 keys present with DE/EN, proper umlauts, SON last. |
| `Plugins/SpotOn/Plugin.pm` | handleFeed, Home/Library feeds, Search, detail pages | ✓ VERIFIED (partial) | 28 subs total. All required subs present and wired. CR-04 gap: _albumItem passthrough incomplete. |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| Plugin.pm handleFeed | API/Client.pm | none (navigation only) | ✓ WIRED | handleFeed wires Home/Search/Library to sub-feed handlers |
| Plugin.pm _recentlyPlayedFeed | Client->getRecentlyPlayed | API call | ✓ WIRED | Line 369: direct call with limit=50, _noCache=1 |
| Plugin.pm _madeForYouFeed | Client->getUserPlaylists | API call + owner.id filter | ✓ WIRED | Lines 391, 397: getUserPlaylists then _isMadeForYou filter |
| Plugin.pm _topTracksFeed | Client->getTopTracks | API call | ✓ WIRED | Line 410: time_range=medium_term, limit=50 |
| Plugin.pm _savedTracksFeed | Client->getSavedTracks | API call + LMS pagination | ✓ WIRED | Line 474: offset/limit from $args, total returned |
| Plugin.pm _savedAlbumsFeed | Client->getSavedAlbums | API call | ✓ WIRED (limited) | Line 495: hardcoded offset=0, limit=50, no total (LMS bug workaround) |
| Plugin.pm _followedArtistsFeed | Client->getFollowedArtists | API call | ✓ WIRED | Line 518: cursor-based, no offset, type=artist hardcoded |
| Plugin.pm _userPlaylistsFeed | Client->getUserPlaylists | API call + MFY filter | ✓ WIRED (limited) | Line 538: hardcoded offset=0, excludes Made-For-You |
| Plugin.pm _searchFeed | Client->search | API call | ✓ WIRED | Line 574: type='track,album,artist,playlist', limit=10 |
| Plugin.pm _searchTypeFeed | Client->search | API call (no pagination) | PARTIAL | Line 661: correct single-type search, but offset=0 hardcoded, no total |
| Plugin.pm _artistFeed | _artistAlbumsFeed x4 | passthrough includeGroups | ✓ WIRED | 4 sections each with single include_groups value |
| Plugin.pm _artistAlbumsFeed | Client->getArtistAlbums | API call | ✓ WIRED (limited) | Line 758: correct single include_groups, offset=0 hardcoded |
| Plugin.pm _albumFeed | Client->getAlbum / getAlbumTracks | API calls | ✓ WIRED | Line 801 (page 1 getAlbum), line 823 (page N getAlbumTracks) |
| Plugin.pm _albumItem | _albumFeed | passthrough | PARTIAL | passthrough only has {albumId} — missing albumImages and albumArtist for page 2+ |
| Plugin.pm _playlistFeed | Client->getPlaylistItems | API call + pagination | ✓ WIRED | Line 908: offset/limit from $args, cap 100, total returned, null track guard present |
| Plugin.pm _trackItem | spotify:// URI | play and url fields | ✓ WIRED | Lines 268-269: 'spotify://' . $track->{uri} in url and play fields |

---

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| _savedTracksFeed | $data->{items} | getSavedTracks -> /me/tracks | Yes — paginated Spotify library query | ✓ FLOWING |
| _savedAlbumsFeed | $data->{items} | getSavedAlbums -> /me/albums | Yes (first 50 only — LMS bug workaround) | ✓ FLOWING (limited) |
| _recentlyPlayedFeed | $data->{items} | getRecentlyPlayed -> /me/player/recently-played | Yes — live, no cache | ✓ FLOWING |
| _madeForYouFeed | @mfy (filtered) | getUserPlaylists -> /me/playlists | Yes — filtered by owner.id='spotify' | ✓ FLOWING |
| _searchFeed | $data->{tracks}{items}[] etc | search -> /search | Yes — live search results | ✓ FLOWING |
| _albumFeed (page 2+) | $albumImages | _albumItem passthrough->{albumImages} | No — always undef (passthrough incomplete) | ✗ HOLLOW_PROP |
| _searchTypeFeed | $resultItems | search -> /search (single type) | Yes — but always offset=0, max 10 items | STATIC (offset) |

---

### Behavioral Spot-Checks

Step 7b: SKIPPED — no runnable entry points outside LMS runtime. All code requires LMS modules (Slim::*) not available in this environment. The SUMMARY.md notes `perl -c` fails outside LMS due to missing JSON::XS::VersionOneAndTwo module. This is expected behavior.

---

### Probe Execution

Step 7c: No probe scripts found in scripts/ directory for this phase. Phase uses manual human-verify checkpoint (03-03-PLAN.md Task 2).

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|---------|
| NAV-01 | 03-01, 03-02 | Top-level menu: Home, Search, Library | ? NEEDS HUMAN | Code wired in handleFeed; functional verification requires LMS |
| NAV-02 | 03-02 | Home: Recently Played, Made For You, Top Tracks | ? NEEDS HUMAN | _homeFeed returns 3 items; content depends on account data |
| NAV-03 | 03-01, 03-02 | Library: Liked Songs, Saved Albums, Followed Artists, User Playlists | ✓ SATISFIED | _libraryFeed wires all 4 items; "Saved Albums"=_savedAlbumsFeed, "Followed Artists"=_followedArtistsFeed |
| NAV-04 | 03-01, 03-03 | Search with categorized results | ? NEEDS HUMAN | _searchFeed structure correct; result verification needs live API |
| NAV-05 | 03-02, 03-03 | Artist detail: Discography (Albums, Singles, Compilations) | ✓ SATISFIED | _artistFeed returns 4 sections (Albums, Singles, Compilations, Appears On); no Top Tracks |
| NAV-06 | 03-03 | Album detail: track number, duration, featuring artists | ✓ SATISFIED (partial) | _albumTrackItem builds correct items; page 2+ lacks artwork (CR-04) |
| NAV-07 | 03-03 | Playlist detail: paginated tracks, description, creator | ✓ SATISFIED | _playlistFeed: paginated, null guard, total returned |
| NAV-08 | 03-02 | Liked Songs unconditional access | ✓ SATISFIED | _savedTracksFeed has no gating code; called directly from _libraryFeed |
| NAV-09 | 03-02 | Library items: recently added as default | ✓ SATISFIED | Spotify API default sort for /me/tracks is added_at desc; no override applied |
| NAV-10 | 03-01, 03-02, 03-03 | Dev Mode removed endpoints gracefully hidden | ✓ SATISFIED | No menu items or API methods for Top Tracks (artist), Related Artists, Browse Categories, New Releases |
| NAV-11 | 03-01, 03-03 | Search pagination: limit=10 per request | ✗ BLOCKED | _searchTypeFeed caps at 10 (Dev Mode) but hardcodes offset=0 and returns no total — multi-page search disabled |

**Note on NAV-02 vs REQUIREMENTS.md:** NAV-02 says "via category ID trick" but implementation uses getUserPlaylists + owner.id='spotify' filter (D-04 decision in CONTEXT.md). The CONTEXT.md explicitly superseded the "category ID trick" approach because Browse Categories endpoint was removed in Feb 2026 Dev Mode. The implementation satisfies the intent of NAV-02 (Made For You mixes visible in Home) through an alternative method.

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|---------|--------|
| Plugin.pm | 290-298 | _albumItem passthrough missing albumImages/albumArtist | BLOCKER | Album page 2+ shows tracks without artwork |
| Plugin.pm | 661-701 | _searchTypeFeed offset=0 hardcoded, no total | WARNING | Search type results capped at 10, no pagination |
| Client.pm | 347-354 | Retry-After: 0 bypasses rate limit cache | WARNING | WR-04: $retryAfter=0 overrides default; cache TTL=0 means no protection if Spotify sends Retry-After: 0 |
| Plugin.pm | 375, 483 | _recentlyPlayedFeed and _savedTracksFeed lack null track guard | WARNING | WR-02: potential crash if Spotify returns null track object in these endpoints |
| TokenManager.pm | 26-38 | REQUIRED_SCOPES not compared to stored token scopes | WARNING | WR-03: users with old tokens get silent 403 on getFollowedArtists, no re-auth prompt |

---

### Human Verification Required

#### 1. Top-Level Menu (NAV-01)

**Test:** Restart LMS (or reload plugin). Note: re-auth required because OAuth scopes changed (user-follow-read, playlist-read-collaborative added). Open SpotOn menu.
**Expected:** Account Switcher, then Home, Search (with text input box), Library as top-level items.
**Why human:** LMS menu rendering and type='search' behavior cannot be verified without running LMS.

#### 2. Home Feed (NAV-02)

**Test:** Navigate Home -> verify 3 sub-items: Recently Played, Made For You, Top Tracks. Open each.
**Expected:** Recently Played shows track list with artwork. Made For You shows Spotify-generated playlists (Discover Weekly, Daily Mix, etc.). Top Tracks shows user's top tracks.
**Why human:** Content depends on user's Spotify account and live API responses.

#### 3. Library Navigation (NAV-03)

**Test:** Navigate Library -> verify 4 sub-items. Open Playlists -> confirm Made-For-You playlists are NOT in this list.
**Expected:** Liked Songs, Albums, Artists, Playlists. Playlists section excludes Discover Weekly / Daily Mix.
**Why human:** Filter correctness (owner.id='spotify') needs runtime verification with actual data.

#### 4. Search Results (NAV-04)

**Test:** Use Search -> type "Radiohead" -> verify result structure.
**Expected:** Top Result section (first track inline), then Tracks/Albums/Artists/Playlists sections with result counts. Categories with 0 results absent.
**Why human:** Requires live search API and LMS search input rendering.

#### 5. Search Pagination (NAV-11)

**Test:** From search Tracks category, scroll through results.
**Expected:** With current implementation, max 10 results shown per type. No "load more" pagination. Verify this is acceptable UX or escalate NAV-11 gap.
**Why human:** This is a deliberate design trade-off (LMS XMLBrowser bug). Human must decide if NAV-11 is considered satisfied by "limiting to 10 per type" or if pagination must be implemented differently.

#### 6. Artist Detail (NAV-05, NAV-10)

**Test:** From search results, navigate into an Artist. Verify sections.
**Expected:** Albums, Singles, Compilations, Appears On — no Top Tracks, no Related Artists.
**Why human:** Menu structure verification requires LMS UI.

#### 7. Album Detail (NAV-06) — including page 2+ artwork

**Test:** Open an album with >50 tracks (e.g., a compilation). Navigate to page 2 of tracks.
**Expected:** Track numbers visible. Page 1 artwork correct. Page 2 artwork: EXPECTED TO BE MISSING (CR-04 gap). Confirm whether this is acceptable.
**Why human:** Multi-page album testing requires actual LMS interaction. This test targets the CR-04 gap.

#### 8. Playlist Detail (NAV-07)

**Test:** Navigate to a playlist. Scroll through tracks.
**Expected:** Paginated track list. Null track entries (local files) skipped. Made-For-You playlists may show NO_RESULTS if 403 returned.
**Why human:** Pagination and error-case behavior need runtime verification.

#### 9. Track Context Navigation (D-07)

**Test:** On a track item, access context menu / long-press. Look for "View Artist" and "View Album" options.
**Expected:** Both context items appear when track has artist and album IDs. Tapping navigates to artist/album feed.
**Why human:** LMS context menu behavior varies by controller type (Jive remote, web UI, iPeng, etc.).

#### 10. Scope Re-Authentication (WR-03)

**Test:** If existing Spotify session predates Phase 3, test getFollowedArtists (Library -> Artists).
**Expected:** Either re-auth happened (new scopes granted) OR Library Artists shows NO_RESULTS gracefully (403 fallback). No crash. No error in LMS log.
**Why human:** Scope mismatch detection (WR-03) is not implemented — old tokens cause silent 403 with NO_RESULTS fallback.

#### 11. LMS Log Check

**Test:** `grep -i 'spoton\|error\|warn' /path/to/lms/server.log` after navigating through all feeds.
**Expected:** No uncaught exceptions. Warning-level messages for 403/429 acceptable. No "Can't use string as HASH reference" errors.
**Why human:** Runtime error detection requires actual LMS session.

---

### Gaps Summary

**Two gaps identified:**

**GAP 1 — BLOCKER: Album page 2+ artwork missing (CR-04)**

`_albumItem` builds passthrough as `[{ albumId => $id }]` only. When `_albumFeed` handles page 2+ (offset > 0), it reads `$passthrough->{albumImages}` and `$passthrough->{albumArtist}` which are undef because `_albumItem` never sets them. This was identified as CR-04 in the code review and the fix commits (735148d, c4f7254) did NOT address it — they fixed passthrough dereference syntax and feed total/pagination logic respectively.

Fix: Add `albumImages => $album->{images}, albumArtist => $firstArtist` to `_albumItem`'s passthrough hashref.

This gap affects NAV-06 (album tracklist). For most albums (<50 tracks), page 2 is never reached so the bug is invisible. Only prolific artists (classical, jazz compilations, etc.) expose this.

**GAP 2 — WARNING: Search type pagination disabled (NAV-11)**

`_searchTypeFeed` hardcodes `offset=0, limit=10` and returns no `total`. This means drilling into a search category (e.g., all Tracks matching "Radiohead") shows max 10 items with no way to load more. The fix commit c4f7254 deliberately removed pagination from this feed citing the LMS XMLBrowser sub-item bug.

However: search type results are navigable sub-items (link type), so the same LMS bug that prevents pagination on album/artist feeds may apply here too. The current behavior (max 10 items, no pagination) may be the correct trade-off. This requires human decision on whether NAV-11 is satisfied.

---

_Verified: 2026-05-28T16:00:00Z_
_Verifier: Claude (gsd-verifier)_
