---
phase: 04-single-track-streaming
fixed_at: 2026-05-28T00:00:00Z
review_path: .planning/phases/04-single-track-streaming/04-REVIEW.md
iteration: 1
findings_in_scope: 7
fixed: 7
skipped: 0
status: all_fixed
---

# Phase 04: Code Review Fix Report

**Fixed at:** 2026-05-28
**Source review:** .planning/phases/04-single-track-streaming/04-REVIEW.md
**Iteration:** 1

**Summary:**
- Findings in scope: 7 (3 Critical + 4 Warning)
- Fixed: 7
- Skipped: 0

## Fixed Issues

### CR-01: Fehlender `require` fur Helper in ProtocolHandler.pm

**Files modified:** `Plugins/SpotOn/ProtocolHandler.pm`
**Commit:** af66517
**Applied fix:** `require Plugins::SpotOn::Helper;` direkt vor dem `getCapability('passthrough')`-Aufruf im `if (grep { $_ eq 'ogg' })` Block eingefugt — analog zum vorhandenen `require Plugins::SpotOn::Plugin;` im gleichen Scope.

---

### CR-02: `Slim::Utils::Versions` in ProtocolHandler.pm nicht importiert

**Files modified:** `Plugins/SpotOn/ProtocolHandler.pm`
**Commit:** 166f444
**Applied fix:** `use Slim::Utils::Versions;` in die `use`-Sektion am Anfang von ProtocolHandler.pm eingefugt, zwischen `Slim::Utils::Prefs` und `Slim::Player::CapabilitiesHelper`.

---

### CR-03: Shell-Injection-Risiko bei `pkill -f "$helper"`

**Files modified:** `Plugins/SpotOn/Plugin.pm`
**Commit:** 51f7289
**Applied fix:** Unix-Pfad mit Single-Quote-Escaping abgesichert (`s/'/'\\''/g` + `pkill -f '$safeHelper'`), passend zum Muster in `Helper.pm::helperCheck()`. Windows-Pfad mit Double-Quote-Wrapping und Bereinigung von Anführungszeichen im Namen abgesichert.

---

### WR-01: Fehlender HTML-Escape bei `helperMissing` in basic.html

**Files modified:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html`
**Commit:** 17ee52b
**Applied fix:** `[% helperMissing %]` zu `[% helperMissing | html %]` geandert — konsistent mit allen anderen Template-Ausgaben in der Datei.

---

### WR-02: Doppelter Binary-Status-Block in basic.html

**Files modified:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html`
**Commit:** 9b314be
**Applied fix:** Block 1 (den redundanten `IF helperMissing; WRAPPER`-Block) vollstandig entfernt. Block 2 deckt beide Zustande (Binary vorhanden / fehlend) bereits vollstandig ab.

---

### WR-03: Aufruf einer privaten Methode `_buildRedirectUri` aus einer anderen Klasse

**Files modified:** `Plugins/SpotOn/API/TokenManager.pm`, `Plugins/SpotOn/Settings.pm`
**Commit:** 08d515e
**Applied fix:** Offentliche Wrapper-Methode `buildRedirectUri` in TokenManager.pm hinzugefugt (delegiert an `_buildRedirectUri`). Settings.pm aktualisiert, um die offentliche Methode aufzurufen.

---

### WR-04: Normalisierungs-Flag-Injektion ohne Fehlerbehandlung

**Files modified:** `Plugins/SpotOn/Plugin.pm`
**Commit:** 6df07a6
**Applied fix:** Normalisierungsblock umstrukturiert: Vorherigen Wert in `$before` speichern, Regex anwenden, dann per `eq`-Vergleich prufen ob der Match erfolgreich war. Bei fehlendem Match wird `$log->warn(...)` ausgegeben.

## Skipped Issues

Keine — alle Findings wurden erfolgreich angewendet.

---

_Fixed: 2026-05-28_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
