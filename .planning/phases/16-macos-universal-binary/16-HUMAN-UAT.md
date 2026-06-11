---
status: partial
phase: 16-macos-universal-binary
source: [16-VERIFICATION.md]
started: 2026-06-11
updated: 2026-06-11
---

## Current Test

[awaiting human testing]

## Tests

### 1. LMS Plugin Manager macOS Binary Detection
expected: Helper.pm ISMAC-Block registriert Bin/darwin/ Pfad wenn main::ISMAC wahr ist. LMS findet das Universal Binary auf macOS automatisch via addFindBinPaths.
result: [pending]

### 2. Gatekeeper-Hint sichtbar wenn Binary fehlt auf macOS
expected: Settings-Seite zeigt orangen Gatekeeper-Hinweis mit xattr-Befehl wenn binaryPath leer und isMac=1. Hinweis erscheint nur auf macOS, nicht auf Linux/Windows.
result: [pending]

### 3. GitHub Actions CI produziert gueltiges Universal Binary
expected: Tag-Push oder workflow_dispatch erzeugt macOS Universal Binary (lipo -info zeigt x86_64 + arm64), ad-hoc signiert (codesign -dv zeigt Signature=adhoc), als spoton-darwin im GitHub Release.
result: [pending]

## Summary

total: 3
passed: 0
issues: 0
pending: 3
skipped: 0
blocked: 0

## Gaps
