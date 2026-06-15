---
phase: quick-260615-khq
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - Plugins/SpotOn/Connect/DaemonManager.pm
autonomous: true
---

<objective>
Fix two Connect daemon hygiene issues in DaemonManager.pm:
1. Prevent daemon start for players without cached credentials (eliminates crash-loop-detection → 30min disable → retry cycle)
2. Clean up orphaned connect log files at startup for players no longer connected to LMS

Purpose: Stop pointless crash loops that fill logs with "No cached credentials" errors and bloat diagnostic bundles with stale log files.
Output: Modified DaemonManager.pm with credential pre-check and delayed orphan log cleanup.
</objective>

<execution_context>
@.planning/quick/260615-khq-daemon-credential-check-orphan-log-clean/260615-khq-PLAN.md
</execution_context>

<context>
@Plugins/SpotOn/Connect/DaemonManager.pm
@Plugins/SpotOn/Connect/Daemon.pm
</context>

<tasks>

<task type="auto">
  <name>Task 1: Credential pre-check before daemon start</name>
  <files>Plugins/SpotOn/Connect/DaemonManager.pm</files>
  <action>
In DaemonManager.pm, add `use File::Spec::Functions qw(catdir catfile);` to the imports (top of file, near the existing `use` statements). Also add `my $serverPrefs = preferences('server');` alongside the existing `my $prefs = preferences('plugin.spoton');`.

In the `startHelper()` method, BEFORE the existing `my $helper = $helperInstances{$clientId};` line (around line 220), add a credential existence check:

1. Resolve the client object from `$clientId` (it is already a plain string at this point, after the `blessed` extraction on line 217).
2. Construct the credential path using the SAME logic as Daemon.pm::start() lines 89-92:
   - Read `$prefs->get('activeAccount')` for the active account ID.
   - Build the base cache dir: if activeAccountId is set, use `catdir($serverPrefs->get('cachedir'), 'spoton', $activeAccountId)`, otherwise `catdir($serverPrefs->get('cachedir'), 'spoton')`.
   - Derive mac_no_colons: `(my $macClean = $clientId) =~ s/://g;`
   - Build credential file path: `catfile($cacheDir, 'credentials.json')` — this is the credentials.json that librespot writes directly into the cache dir passed via `-c`. Note: the `-c` flag in Daemon.pm passes the entire account-scoped cache dir, and librespot stores credentials.json at the root of that cache dir, NOT in a per-device subdirectory.

Wait — re-reading the user's description, they specify `catdir($cacheDir, 'spoton', 'connect-' . $mac_no_colons, 'credentials.json')`. Honor this path exactly: the full credential check path is `catfile($serverPrefs->get('cachedir'), 'spoton', 'connect-' . $macClean, 'credentials.json')`.

3. If the credential file does NOT exist (`! -f $credFile`):
   - Log at INFO level: `"Skipping Connect daemon for $clientId - no cached credentials (expected: $credFile)"`
   - Return immediately (do not proceed to create or start a daemon).

This check MUST come before any Daemon->new() or $helper->start() call so the crash loop never begins.
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/Connect/DaemonManager.pm 2>&1 | grep -q 'syntax OK' && echo PASS || echo FAIL</automated>
  </verify>
  <done>DaemonManager.pm skips daemon start when no credentials.json exists for the player MAC, with INFO-level log message. Syntax check passes.</done>
</task>

<task type="auto">
  <name>Task 2: Delayed orphaned log file cleanup at startup</name>
  <files>Plugins/SpotOn/Connect/DaemonManager.pm</files>
  <action>
Add a constant at the top of DaemonManager.pm (near the existing constants):
`use constant ORPHAN_LOG_CLEANUP_DELAY => 30;`

In the `init()` method, AFTER the existing immediate initHelpers timer (line 80: `Slim::Utils::Timers::setTimer($class, Time::HiRes::time() + 0.5, \&initHelpers);`), add a delayed timer for orphaned log cleanup:

```
Slim::Utils::Timers::setTimer($class, Time::HiRes::time() + ORPHAN_LOG_CLEANUP_DELAY, \&_cleanupOrphanedLogs);
```

Create a new private subroutine `_cleanupOrphanedLogs`:

1. Build the spoton cache base dir: `my $baseDir = catdir($serverPrefs->get('cachedir'), 'spoton');`
2. Use `glob` to find all files matching `*-connect.log` in `$baseDir`: `my @logFiles = glob(catfile($baseDir, '*-connect.log'));`
3. For each log file:
   a. Extract the MAC portion from the filename. The filename format is `{MAC_NO_COLONS}-connect.log`. Use a regex on the basename: `if (basename($f) =~ /^([0-9a-f]{12})-connect\.log$/i)` to capture the 12-hex-char MAC.
   b. Convert to colon-separated format: insert colons every 2 chars to get `aa:bb:cc:dd:ee:ff` format. Use: `my $mac = join(':', $macClean =~ /../g);`
   c. Check if a connected player exists: `my $client = Slim::Player::Client::getClient($mac);`
   d. If NO client is found (player not connected to LMS): `unlink $f;` and log at INFO level: `"Cleaned up orphaned Connect log: " . basename($f)`
   e. If client IS found: skip (player is connected, log file is active).

Add `use File::Basename qw(basename);` to the imports at the top of the file (File::Spec::Functions was already added in Task 1).

The 30s delay ensures players have had time to connect to LMS after a restart. Only log FILES are deleted — credential directories are never touched (players may reconnect later).
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/Connect/DaemonManager.pm 2>&1 | grep -q 'syntax OK' && echo PASS || echo FAIL</automated>
  </verify>
  <done>DaemonManager.pm cleans up orphaned *-connect.log files 30s after init, only for players not connected to LMS. Credential directories are never touched. Syntax check passes.</done>
</task>

</tasks>

<verification>
1. `perl -c Plugins/SpotOn/Connect/DaemonManager.pm` passes syntax check
2. Credential check is positioned BEFORE Daemon->new() / $helper->start() in startHelper()
3. Orphaned log cleanup uses a 30s delay timer, not immediate execution
4. Only log files matching `*-connect.log` are deleted, never credential dirs
5. Both features log at INFO level for observability
</verification>

<success_criteria>
- Players without credentials.json are silently skipped (no crash loop, no 30min disable cycle)
- Orphaned log files from disconnected players are cleaned up on restart
- Existing behavior for players WITH credentials is completely unchanged
- No new CPAN dependencies introduced
</success_criteria>

<output>
Create `.planning/quick/260615-khq-daemon-credential-check-orphan-log-clean/260615-khq-SUMMARY.md` when done
</output>
