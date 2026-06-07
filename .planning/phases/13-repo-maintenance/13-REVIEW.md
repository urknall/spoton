---
phase: 13-repo-maintenance
reviewed: 2026-06-07T00:00:00Z
depth: standard
files_reviewed: 8
files_reviewed_list:
  - .github/workflows/perl-tests.yml
  - .github/ISSUE_TEMPLATE/bug_report.yml
  - .github/ISSUE_TEMPLATE/feature_request.yml
  - .github/ISSUE_TEMPLATE/config.yml
  - .gitignore
  - CONTRIBUTING.md
  - LICENSE
  - README.md
findings:
  critical: 0
  warning: 4
  info: 3
  total: 7
status: issues_found
---

# Phase 13: Code Review Report

**Reviewed:** 2026-06-07
**Depth:** standard
**Files Reviewed:** 8
**Status:** issues_found

## Summary

These are repository maintenance artifacts — CI workflow, GitHub issue templates, gitignore, contributing guide, license, and README. No source code logic is under review. The files are generally well-structured. Four warnings stand out: two factual errors in CONTRIBUTING.md (wrong Bin path and wrong armv7 directory name), a missing SpotOn plugin version field in the bug report template, and a stale LICENSE copyright year. Three info items cover minor omissions and the missing macOS build target.

## Warnings

### WR-01: CONTRIBUTING.md lists wrong top-level path for Bin directory

**File:** `CONTRIBUTING.md:49`
**Issue:** The project structure listing shows `Bin/` at the repository root, but no such directory exists. The binaries live at `Plugins/SpotOn/Bin/`. A contributor looking here to add or inspect binaries will search the wrong location.
**Fix:** Change line 49 from:
```
Bin/                     librespot binaries (per architecture, auto-selected)
```
to:
```
Plugins/SpotOn/Bin/      librespot binaries (per architecture, auto-selected)
```

---

### WR-02: CONTRIBUTING.md shows wrong armv7 bin-dir name

**File:** `CONTRIBUTING.md:52`
**Issue:** The example lists `armv7-linux/` as the ARM 32-bit directory, but the actual on-disk name (and the `bin_dir` value in `build-librespot.yml:30`) is `armhf-linux`. A contributor cross-referencing this table when debugging ARM deployment will be confused.
**Fix:** Change line 52 from:
```
  armv7-linux/
```
to:
```
  armhf-linux/
```

---

### WR-03: Bug report template has no SpotOn plugin version field

**File:** `.github/ISSUE_TEMPLATE/bug_report.yml:1`
**Issue:** The template collects LMS version, OS, and player type, but omits the SpotOn plugin version. Bug triaging requires knowing which release is affected — a regression in v1.2.3 needs a different fix path than one in v1.1. Without this field, reporters skip it, making every bug report version-ambiguous.
**Fix:** Add a required version field after the `lms-version` block (approximately after line 11):
```yaml
  - type: input
    id: spoton-version
    attributes:
      label: SpotOn Version
      placeholder: "e.g. 1.2.4"
    validations:
      required: true
```

---

### WR-04: LICENSE copyright year is stale

**File:** `LICENSE:3`
**Issue:** Copyright year is `2024`. The project has shipped releases through at least 2026 (v1.2.x). While not a legal defect under MIT, a frozen year is misleading and is the first thing users/lawyers check when assessing active maintenance.
**Fix:** Update line 3 to:
```
Copyright (c) 2024–2026 Marek Stiefenhofer
```

---

## Info

### IN-01: CONTRIBUTING.md's Settings entry is wrong (directory vs file)

**File:** `CONTRIBUTING.md:38`
**Issue:** The project structure table lists `Settings/` (implying a subdirectory) but the actual path is `Plugins/SpotOn/Settings.pm` — a single file, not a directory. Also, several top-level plugin files are omitted from the table (`Connect.pm`, `Helper.pm`, `DontStopTheMusic.pm`).
**Fix:** Change `Settings/` to `Settings.pm` and add the missing top-level files to the table.

---

### IN-02: README.md has no link to CONTRIBUTING.md

**File:** `README.md:1`
**Issue:** The README has no Contributing section or link pointing newcomers to `CONTRIBUTING.md`. GitHub renders a Contributing link automatically when the file exists, but explicit mention in the README reduces friction for first-time contributors who read the README end-to-end.
**Fix:** Add a brief section at the end:
```markdown
## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for setup and pull request guidelines.
```

---

### IN-03: macOS build target absent from build-librespot.yml

**File:** `.github/workflows/build-librespot.yml:21`
**Issue:** The CLAUDE.md stack table lists macOS (`x86_64-apple-darwin`, `aarch64-apple-darwin`) as a supported platform, but there is no macOS matrix entry in `build-librespot.yml`. Users on macOS will either get no binary or have to build from source without documentation on how to do so.

This is an info-level finding because LMS on macOS has a small user base and the omission may be intentional (e.g., cross-compilation to macOS from Linux requires osxcross, which is non-trivial). Worth a deliberate decision either way.
**Fix:** Either add macOS targets to the matrix, or explicitly document in `CONTRIBUTING.md` that macOS binaries are not distributed pre-built.

---

_Reviewed: 2026-06-07_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
