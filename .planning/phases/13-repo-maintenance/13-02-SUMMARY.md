---
phase: 13-repo-maintenance
plan: "02"
subsystem: contributor-infrastructure
tags: [github, issue-templates, contributing, license, readme, documentation]
dependency_graph:
  requires: []
  provides: [REPO-03, REPO-04, REPO-05]
  affects: []
tech_stack:
  added: []
  patterns: [github-yaml-issue-forms, mit-license, ci-badge]
key_files:
  created:
    - .github/ISSUE_TEMPLATE/bug_report.yml
    - .github/ISSUE_TEMPLATE/feature_request.yml
    - .github/ISSUE_TEMPLATE/config.yml
    - CONTRIBUTING.md
    - LICENSE
    - README.md
  modified: []
decisions:
  - "YAML Forms format (.yml) chosen over Markdown templates for structured validation"
  - "blank_issues_enabled: false to steer contributors toward structured forms"
  - "Lyrion community forum contact link added in config.yml for general LMS questions"
metrics:
  duration: "~10 minutes"
  completed: "2026-06-07"
---

# Phase 13 Plan 02: Issue Templates, CONTRIBUTING, LICENSE, README Summary

**One-liner:** Structured GitHub issue forms (bug/feature), MIT license, CONTRIBUTING guide with standalone test instructions, and README with CI badge and install instructions.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Issue Templates (Bug Report + Feature Request + config.yml) | d360369 | `.github/ISSUE_TEMPLATE/bug_report.yml`, `feature_request.yml`, `config.yml` |
| 2 | CONTRIBUTING.md + LICENSE + README.md | c424313 | `CONTRIBUTING.md`, `LICENSE`, `README.md` |

## What Was Built

### Issue Templates

**bug_report.yml** — Structured bug report form with 5 fields:
- `lms-version` (input, required) — LMS version e.g. 9.1.1
- `os` (dropdown, required) — Linux x86_64 / Linux ARM-Raspberry Pi / macOS / Windows / Other
- `reproduction` (textarea, required) — Steps to reproduce
- `logs` (textarea, optional) — Log excerpt from server.log
- `player-type` (dropdown, optional) — squeezelite / UPnPBridge-BridgeSDK / piCorePlayer / Other

**feature_request.yml** — Feature request form with 3 fields:
- `problem` (textarea, required) — Problem statement
- `solution` (textarea, required) — Proposed solution
- `alternatives` (textarea, optional) — Alternatives considered

**config.yml** — Template chooser: blank issues disabled, Lyrion forum contact link.

### CONTRIBUTING.md

English, ~1 page. Sections:
1. Prerequisites — Perl 5.36+, git; no LMS installation needed; no CPAN setup
2. Running Tests — `prove t/`, expected 12 files / 230 tests / under 1 second
3. Project Structure — brief map of `Plugins/SpotOn/`, `t/`, `.github/workflows/`, `Bin/`
4. Pull Request Guidelines — branch from main, CI must pass (5.36 + 5.38), focused changes

### LICENSE

MIT License, copyright 2024 Marek Stiefenhofer.

### README.md

- CI badge (perl-tests.yml) as first line
- Features: Browse, Streaming (5 format modes), Spotify Connect, Library
- Requirements: LMS 8.0+ (9.x recommended), Spotify Premium
- Installation via repo.xml URL in LMS Settings

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all files contain final, wired content.

## Threat Flags

None — these are static documentation files with no trust boundary exposure.

## Self-Check: PASSED

- `.github/ISSUE_TEMPLATE/bug_report.yml` — exists, contains `lms-version`, 3 required fields
- `.github/ISSUE_TEMPLATE/feature_request.yml` — exists, contains `problem`, 2 required fields
- `.github/ISSUE_TEMPLATE/config.yml` — exists, `blank_issues_enabled: false`
- `CONTRIBUTING.md` — exists, contains `prove t/`, mentions "No CPAN", "Perl core"
- `LICENSE` — exists, `MIT License`, `Marek Stiefenhofer`, `2024`
- `README.md` — exists, CI badge with `perl-tests.yml/badge.svg`, repo.xml URL, LMS 8.0+, Spotify Premium
- Commits d360369 and c424313 verified in git log
