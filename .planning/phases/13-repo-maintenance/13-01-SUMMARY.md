---
phase: 13-repo-maintenance
plan: 01
subsystem: ci-infrastructure
tags: [ci, github-actions, gitignore, repo-hygiene]
dependency_graph:
  requires: []
  provides: [perl-ci-workflow, clean-gitignore, phase-cleanup]
  affects: [.github/workflows/perl-tests.yml, .gitignore]
tech_stack:
  added:
    - shogo82148/actions-setup-perl@v1 (GitHub Actions, Perl matrix)
    - actions/checkout@v4 (GitHub Actions)
    - prove (Perl core test runner)
  patterns:
    - GitHub Actions matrix strategy (fail-fast: false)
    - .gitignore pattern for release artifacts
key_files:
  created:
    - .github/workflows/perl-tests.yml
  modified:
    - .gitignore
  deleted:
    - .planning/phases/10-connect-dstm/ (9 files)
    - .planning/phases/11-track-history-metadata/ (7 files)
    - .planning/phases/12-protocol-handler-rename/ (8 files, incl. .gitkeep)
decisions:
  - Perl 5.36 and 5.38 matrix (LMS 9.x ships ~5.38; 5.36 as prior stable)
  - No cpanm step -- all test modules are Perl core (Test::More, etc.)
  - fail-fast: false to match existing build-librespot.yml convention
  - shogo82148/actions-setup-perl@v1 -- established action (400+ stars)
metrics:
  duration: "4 minutes"
  completed_date: "2026-06-07"
  tasks_completed: 2
  files_created: 1
  files_modified: 1
  files_deleted: 24
---

# Phase 13 Plan 01: Repo Hygiene and Perl CI Summary

**One-liner:** GitHub Actions Perl CI workflow (5.36+5.38 matrix via prove t/) with gitignore cleanup and stale phase-10/11/12 artifact removal.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Repo Hygiene -- .gitignore + artifact cleanup | 2ba24f9 | .gitignore (modified), 24 phase files deleted |
| 2 | GitHub Actions Perl CI Workflow | 23e20a4 | .github/workflows/perl-tests.yml (created) |

## What Was Built

**Task 1 -- Repo Hygiene:**
- Added `SpotOn-v*.zip` to `.gitignore` under a new `# Release ZIP archives` comment block
- Removed tracking of 24 stale planning files from completed phases 10, 11, 12
- Note: The 10 zip archives listed in the original git status were in the main repo's working tree, not in this worktree (worktree starts from committed HEAD at 5aba70c). The .gitignore entry ensures they will be ignored after merge.

**Task 2 -- Perl CI Workflow:**
- Created `.github/workflows/perl-tests.yml` with:
  - Triggers: push and pull_request to main
  - Matrix: Perl 5.36 and 5.38 (fail-fast: false)
  - Steps: checkout@v4, shogo82148/actions-setup-perl@v1, prove t/
  - No cpanm (all test modules are Perl core)
  - Follows build-librespot.yml conventions (step naming, fail-fast pattern)

## Deviations from Plan

**1. [Rule 3 - Context] Zip files absent from worktree**
- **Found during:** Task 1
- **Issue:** The 10 `SpotOn-v1.2.*.zip` files shown in the original git status are untracked files in the main repo's working tree. Git worktrees share the object store but have separate working trees -- the zip files were never committed, so they are not present in the worktree filesystem.
- **Fix:** Proceeded without `rm` (no-op -- files not present). The `.gitignore` entry is the correct fix and was applied. After merge back to main, the zip files will be ignored by git as intended.
- **Files modified:** .gitignore (no deviation from plan content)
- **Commit:** 2ba24f9

## Known Stubs

None.

## Threat Flags

None -- no new network endpoints, auth paths, or trust boundaries introduced. CI workflow uses pinned major versions for both actions (accepted in threat model T-13-01, T-13-02).

## Self-Check

Files created/modified:
- .github/workflows/perl-tests.yml: exists
- .gitignore: SpotOn-v*.zip entry present

Commits:
- 2ba24f9: chore(13-01): repo hygiene -- gitignore + phase cleanup
- 23e20a4: feat(13-01): add GitHub Actions Perl CI workflow
