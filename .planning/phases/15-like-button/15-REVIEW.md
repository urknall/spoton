---
status: findings
phase: 15-like-button
reviewed_files:
  - Plugins/SpotOn/API/Client.pm
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/API/TokenManager.pm
  - Plugins/SpotOn/strings.txt
  - t/02_strings.t
  - t/08_api_client.t
finding_count: 3
severity_counts:
  high: 1
  medium: 1
  low: 1
---

# Code Review — Phase 15: Like Button

## Findings

### 1. [HIGH] SpotOnLike/SpotOnUnlike: Network errors (code=0) treated as success

**File:** `Plugins/SpotOn/Plugin.pm` lines 451, 480
**Category:** correctness

The error check `if ($err && ref $err eq 'HASH' && $err->{code} && $err->{code} >= 400)` fails when `$err->{code}` is `0` — which is false in Perl's boolean context.

Client.pm `_doFlavouredRequest` error callback (line 614) calls `$userCb->(undef, { error => $error, code => $code })` where `$code` can be `0` for network errors (timeout, connection refused, DNS failure — when `$response->code` returns 0 or undef).

**Failure scenario:** User clicks "Like", network is down → Client.pm calls callback with `{error => 'Timeout', code => 0}` → SpotOnLike's error check evaluates false (0 is falsy) → falls through to success path → `$cache->remove($cacheKey)` + shows "Liked!" → user believes save succeeded but it didn't.

**Fix:** Invert the logic — check for success (absence of error) rather than specific error shapes:

```perl
# Instead of checking for specific error codes:
if ($err) {
    my $msg = (ref $err eq 'HASH' && $err->{code} && $err->{code} == 403)
        ? cstring($client, 'PLUGIN_SPOTON_LIKE_ERROR_SCOPE')
        : cstring($client, 'PLUGIN_SPOTON_LIKE_ERROR');
    $cb->({ items => [{ name => $msg, showBriefly => 1 }] });
    return;
}
# success path only reached when $err is undef
```

Same fix applies to SpotOnUnlike (line 480).

---

### 2. [MEDIUM] SpotOnManageLike caches API errors as "not liked" for 60 seconds

**File:** `Plugins/SpotOn/Plugin.pm` lines 429-432
**Category:** correctness

When `checkTracks` returns an error (429 rate-limited, network timeout, 401 unauthorized), `$result` is undef. The callback unconditionally computes `$isLiked = 0` and caches it for 60s via `$cache->set($cacheKey, $isLiked, 60)`.

**Failure scenario:** User has a liked track. Spotify returns 429 rate-limited. `$isLiked` is set to 0 and cached. For the next 60s, the menu shows "Like" instead of "Unlike". If the user clicks "Like" (thinking it's not liked), the track is already liked — no harm from the API call, but the UX is incorrect.

**Fix:** Only cache on success (when `$err` is undef):

```perl
my $isLiked = ($result && ref $result eq 'ARRAY' && $result->[0]) ? 1 : 0;
$cache->set($cacheKey, $isLiked, 60) unless $err;  # don't cache error state
$buildMenu->($isLiked);
```

---

### 3. [LOW] SpotOnLike and SpotOnUnlike are near-identical (~23 lines)

**File:** `Plugins/SpotOn/Plugin.pm` lines 441-494
**Category:** simplification

The two subs differ only in: API method (`saveTracks` vs `removeTracks`) and success string (`PLUGIN_SPOTON_LIKED` vs `PLUGIN_SPOTON_UNLIKED`). Error handling, cache invalidation, callback structure, and 403 branching are identical.

**Cost:** Bug fixes to error handling (like finding #1) must be applied in both locations. Currently low risk with only 2 copies, but could diverge if feature grows.

**Fix (optional):** Extract a shared helper:

```perl
sub _doLibraryAction {
    my ($client, $cb, $args, $apiMethod, $successKey) = @_;
    # shared logic here
}
```

Not blocking — acceptable at current scale.
