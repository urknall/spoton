---
phase: 19-podcast-browse
verified: 2026-06-14T17:32:44Z
status: human_needed
score: 7/7 must-haves verified
overrides_applied: 0
human_verification:
  - test: "Open LMS browser, navigate to SpotOn plugin — confirm 'Podcasts' appears as top-level entry after 'Bibliothek'"
    expected: "Podcasts entry visible at the same level as Home, Suche, Bibliothek"
    why_human: "OPMLBased menu rendering is browser-side — grep confirms the push @items block exists but visual ordering requires live LMS"
  - test: "Enter 'Podcasts > Meine Podcasts' — confirm saved shows appear with publisher name on line2"
    expected: "Show list, each showing publisher below the show title; most-recently-added first"
    why_human: "Requires a live Spotify account with saved shows to exercise getSavedShows API"
  - test: "Select a show — confirm episode list loads with 'NN Min · DD. Mon' on line2"
    expected: "Episode list with title on line1 and duration+date on line2, newest episode first"
    why_human: "Requires live API call to getShowEpisodes; German relative date ('Heute'/'Gestern') only verifiable at runtime"
  - test: "Select an episode — confirm it begins playback"
    expected: "Player shows episode title and artwork; audio plays; Now Playing shows show name as artist"
    why_human: "Requires ProtocolHandler to process spoton://episode:ID — can only be verified on a running LMS with librespot binary"
  - test: "Enter 'Podcasts > Podcast-Suche', type a query — confirm results show Shows and Episodes as separate sections"
    expected: "Two link items labelled 'Shows' and 'Episoden' with result counts; each drills into paginated list"
    why_human: "LMS type=search rendering requires browser interaction; result structure requires live Spotify API call"
---

# Phase 19: Podcast Browse Verification Report

**Phase Goal:** Users can navigate to Podcasts, browse their saved shows, open a show, play episodes, and search for shows/episodes
**Verified:** 2026-06-14T17:32:44Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | A "Podcasts" entry appears in the top-level SpotOn menu alongside Home, Suche, Bibliothek | VERIFIED | `Plugin.pm:302-306` — `push @items { name => cstring($client, 'PLUGIN_SPOTON_PODCASTS'), url => \&_podcastsFeed, type => 'link' }` after PLUGIN_SPOTON_LIBRARY block |
| 2 | "Meine Podcasts" lists all saved shows sorted by add date (most recently added first) | VERIFIED | `_savedShowsFeed` (Plugin.pm:1012) calls `getSavedShows` with `offset`/`limit` only — no sort applied; Spotify GET /me/shows returns newest-added first by default. No alphabetical sort in implementation. |
| 3 | Selecting a show opens its episode list with episode title, duration, and release date visible | VERIFIED | `_showFeed` (Plugin.pm:1058) calls `getShowEpisodes`; maps results via `_episodeItem` which sets `line2 = _formatEpisodeLine2($duration_sec, $release_date)` |
| 4 | Selecting an episode begins playback via the existing ProtocolHandler (spoton:// URI) | VERIFIED | `_episodeItem` (Plugin.pm:1110-1112) constructs `spoton://episode:ID` via same regex/pattern as `_trackItem`; metadata cache populated identically |
| 5 | The Podcasts menu contains a "Podcast-Suche" entry as a distinct submenu item | VERIFIED | `_podcastsFeed` (Plugin.pm:997-1003) pushes `PLUGIN_SPOTON_PODCAST_SEARCH` with `type => 'search'` so LMS renders text input |
| 6 | Entering a query under "Podcast-Suche" returns matching shows and episodes as separate result sections | VERIFIED | `_podcastSearchFeed` (Plugin.pm:1214) does combined `type=show,episode` search; pushes separate SHOWS/EPISODES link items only if total > 0 |
| 7 | Show results and episode results each display up to 10 items (Dev Mode limit) | VERIFIED | `_podcastSearchTypeFeed` (Plugin.pm:1282): `$limit = $qty > 10 ? 10 : $qty` — hard cap at 10 per D-12 |

**Score:** 7/7 truths verified

### Required Artifacts (Plan 19-01)

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/strings.txt` | 5 podcast string keys with 11-language translations | VERIFIED | Lines 287-350: PLUGIN_SPOTON_PODCASTS, MY_PODCASTS, PODCAST_SEARCH, SHOWS, EPISODES each with CS/DA/DE/EN/ES/FR/IT/NL/NO/PL/SV |
| `t/02_strings.t` | String key validation including 5 new podcast keys | VERIFIED | Lines 62-66: all 5 keys in @bilingual_keys; `prove t/02_strings.t` passes |
| `t/08_api_client.t` | Corrected cache version assertion (version 4) | VERIFIED | Lines 610/621/622/628/629: version 4 in comment, both regexes, and both descriptions |

### Required Artifacts (Plan 19-02)

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/Plugin.pm` — `sub _podcastsFeed` | Top-level Podcasts menu container | VERIFIED | Plugin.pm:989 — static menu with MY_PODCASTS (link) and PODCAST_SEARCH (search) |
| `Plugins/SpotOn/Plugin.pm` — `sub _savedShowsFeed` | Paginated saved shows | VERIFIED | Plugin.pm:1012 — calls `getSavedShows`, unwraps `{show}`, OPMLBased pagination |
| `Plugins/SpotOn/Plugin.pm` — `sub _showItem` | Show link item builder | VERIFIED | Plugin.pm:1040 — type=link, publisher on line2, `_largestImage`, showImages in passthrough |
| `Plugins/SpotOn/Plugin.pm` — `sub _showFeed` | Paginated episode list | VERIFIED | Plugin.pm:1058 — calls `getShowEpisodes`, maps via `_episodeItem` |
| `Plugins/SpotOn/Plugin.pm` — `sub _episodeItem` | Episode audio item builder | VERIFIED | Plugin.pm:1093 — spoton://episode:ID URI, 3-tier image fallback, metadata cache, type=audio |
| `Plugins/SpotOn/Plugin.pm` — `sub _formatEpisodeLine2` | Duration + date formatting | VERIFIED | Plugin.pm:1145 — German "Min"/"Std" units, middle-dot U+00B7 separator |
| `Plugins/SpotOn/Plugin.pm` — `sub _formatRelativeDate` | Relative date formatting | VERIFIED | Plugin.pm:1174 — Heute/Gestern/Vor N Tagen (0-6 days), absolute "14. Jun [YYYY]", eval-guarded timelocal |
| `Plugins/SpotOn/Plugin.pm` — `sub _podcastSearchFeed` | Podcast search entry | VERIFIED | Plugin.pm:1214 — empty-query guard, combined type=show,episode call, separate SHOWS/EPISODES sections |
| `Plugins/SpotOn/Plugin.pm` — `sub _podcastSearchTypeFeed` | Typed search results | VERIFIED | Plugin.pm:1274 — typeToKey map, limit capped at 10, dispatches to _showItem or _episodeItem |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `Plugin.pm::_podcastsFeed` | `Plugin.pm::_savedShowsFeed` | `url => \&_savedShowsFeed` in menu item | WIRED | Plugin.pm:994 |
| `Plugin.pm::_savedShowsFeed` | `API::Client::getSavedShows` | async API call | WIRED | Plugin.pm:1021 |
| `Plugin.pm::_showItem` | `Plugin.pm::_showFeed` | `url => \&_showFeed` in item | WIRED | Plugin.pm:1044 |
| `Plugin.pm::_showFeed` | `API::Client::getShowEpisodes` | async API call | WIRED | Plugin.pm:1070 |
| `Plugin.pm::_episodeItem` | ProtocolHandler (spoton://episode:ID) | `spoton_url` in url/play fields | WIRED | Plugin.pm:1112/1131-1132 |
| `Plugin.pm::_podcastSearchFeed` | `API::Client::search` | async call with `type=show,episode` | WIRED | Plugin.pm:1226-1230 |
| `Plugin.pm::_podcastSearchTypeFeed` | `API::Client::search` | async call with type passthrough | WIRED | Plugin.pm:1286-1290 |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `_savedShowsFeed` | `$data->{items}` | `getSavedShows` -> `GET /me/shows` | Yes — real API call via `_request` | FLOWING |
| `_showFeed` | `$data->{items}` | `getShowEpisodes` -> `GET /shows/{id}/episodes` | Yes — real API call | FLOWING |
| `_podcastSearchFeed` | `$data->{shows}`, `$data->{episodes}` | `search` -> `GET /search?type=show,episode` | Yes — real API call | FLOWING |
| `_podcastSearchTypeFeed` | `$typeData->{items}` | `search` -> `GET /search?type={type}` | Yes — real API call with passthrough query | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Perl syntax check | `prove t/05_perl_syntax.t` | All tests successful (8 tests) | PASS |
| String key validation | `prove t/02_strings.t` | All tests successful (105 tests) | PASS |
| Cache version assertion | `prove t/08_api_client.t` | All tests successful (35 tests) | PASS |
| Full test suite | `prove t/` | All tests successful (278 tests, 12 files) | PASS |

### Probe Execution

No declared probes for this phase. Step 7c: SKIPPED (no probe-*.sh files declared or found).

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| NAV-01 | 19-01, 19-02 | "Podcasts" als eigener Top-Level-Menüpunkt | SATISFIED | Plugin.pm:302-306: push @items with PLUGIN_SPOTON_PODCASTS |
| NAV-02 | 19-02 | "Meine Podcasts" zeigt gespeicherte Shows | SATISFIED (with note) | _savedShowsFeed returns API-ordered shows; REQUIREMENTS.md says "alphabetisch" but ROADMAP SC2 and CONTEXT.md D-03 override this to "added_at desc" — implementation matches ROADMAP |
| NAV-03 | 19-01, 19-02 | "Podcast-Suche" als eigener Untermenüpunkt | SATISFIED | _podcastsFeed:997 pushes PODCAST_SEARCH as type=search item |
| POD-01 | 19-02 | Saved Podcast-Shows browsen unter "Meine Podcasts" | SATISFIED | _savedShowsFeed calls getSavedShows with pagination |
| POD-02 | 19-02 | Show öffnen und Episodenliste sehen | SATISFIED | _showFeed calls getShowEpisodes, maps via _episodeItem |
| POD-03 | 19-02 | Episode aus Episodenliste abspielen | SATISFIED | _episodeItem produces spoton://episode:ID audio items |
| SRC-01 | 19-02 | Nach Shows suchen unter "Podcast-Suche" | SATISFIED | _podcastSearchTypeFeed handles type=show |
| SRC-02 | 19-02 | Nach Episoden suchen unter "Podcast-Suche" | SATISFIED | _podcastSearchTypeFeed handles type=episode |
| SRC-03 | 19-02 | Suchergebnisse zeigen Shows und Episoden getrennt | SATISFIED | _podcastSearchFeed pushes separate SHOWS and EPISODES link items |

**Note on NAV-02:** REQUIREMENTS.md NAV-02 still reads "alphabetisch sortiert" but CONTEXT.md D-03 decision explicitly changed this to API-order (added_at desc), and ROADMAP.md SC2 was updated accordingly. The implementation correctly reflects the ROADMAP decision. REQUIREMENTS.md is stale but this is a documentation gap only, not a functional gap.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| None found | — | — | — | — |

No TODO/FIXME/TBD/XXX markers found in any modified file. No stub returns (return null/\[\]/\{\}) in the new podcast subs. All podcast subs dispatch real API calls.

### Human Verification Required

#### 1. Top-Level Menu Placement

**Test:** Open the SpotOn plugin in LMS browser; inspect the top-level menu listing.
**Expected:** "Podcasts" appears as a menu item after "Bibliothek", at the same level as Home, Suche, Bibliothek.
**Why human:** OPMLBased menu rendering is browser-side — the push @items block exists in code but visual ordering and label rendering require a live LMS instance.

#### 2. Saved Shows Browse

**Test:** Navigate to "Podcasts > Meine Podcasts" with a logged-in Spotify account that has saved shows.
**Expected:** List of saved shows with publisher name on line2 (below title); most-recently-added show first.
**Why human:** Requires a live Spotify Premium account with saved shows to exercise getSavedShows API. D-01 publisher display and D-03 ordering are only verifiable against real API data.

#### 3. Episode List with Duration and Date

**Test:** Select a show in "Meine Podcasts".
**Expected:** Episode list loads; each episode shows title on line1 and e.g. "45 Min · Gestern" or "1 Std 23 Min · 12. Jun" on line2; episodes ordered newest first.
**Why human:** Requires live API call to getShowEpisodes; German relative date ("Heute"/"Gestern") output depends on current date and release_date from API response — can only be fully verified at runtime.

#### 4. Episode Playback

**Test:** Select an episode from the episode list.
**Expected:** Player starts the episode; Now Playing screen shows episode title and artwork; show name appears as "artist" in the metadata panel.
**Why human:** Requires ProtocolHandler to process spoton://episode:ID URI, librespot binary to be present and functional, and LMS player to be connected.

#### 5. Podcast Search

**Test:** Navigate to "Podcasts > Podcast-Suche", enter a search query (e.g. "Radiolab").
**Expected:** Results page shows two link items: "Shows" (with count) and "Episoden" (with count); drilling into either shows a paginated list of max 10 results.
**Why human:** type=search item rendering (LMS text input box) and result display require browser interaction; two-section layout requires live Spotify API call to confirm both sections appear.

---

## Stale REQUIREMENTS.md Note

REQUIREMENTS.md NAV-02 reads: "Meine Podcasts zeigt gespeicherte Shows alphabetisch sortiert"

The implementation and ROADMAP.md SC2 instead specify "sorted by add date (most recently added first)". This was an explicit design decision (CONTEXT.md D-03). The REQUIREMENTS.md NAV-02 line was not updated. This is a documentation stale — not a functional gap. The ROADMAP is the authoritative success criteria source per the verification process.

---

_Verified: 2026-06-14T17:32:44Z_
_Verifier: Claude (gsd-verifier)_
