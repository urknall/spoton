---
phase: 16
slug: macos-universal-binary
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-06-11
---

# Phase 16 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Test::More (Perl, bundled with LMS) + CI Verification (GitHub Actions) |
| **Config file** | none — prove runs t/ directly |
| **Quick run command** | `prove t/06_binary_check.t` |
| **Full suite command** | `prove t/` |
| **Estimated runtime** | ~5 seconds |

---

## Sampling Rate

- **After every task commit:** Run `prove t/`
- **After every plan wave:** Run `prove t/` + CI-Workflow-Lauf verifiziert Universal Binary
- **Before `/gsd:verify-work`:** Full suite must be green + erfolgreicher `workflow_dispatch`-Lauf
- **Max feedback latency:** 10 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 16-01-01 | 01 | 1 | PLT-01 | — | N/A | CI | `lipo -info` + `file` in workflow | ❌ W0 | ⬜ pending |
| 16-01-02 | 01 | 1 | PLT-02 | T-16-01 | SHA256 checksum validates binary integrity | CI | `codesign -dv` in workflow | ❌ W0 | ⬜ pending |
| 16-02-01 | 02 | 2 | PLT-02 | — | N/A | unit | `prove t/05_perl_syntax.t` | ✅ | ⬜ pending |
| 16-02-02 | 02 | 2 | PLT-03 | — | N/A | unit | `prove t/02_strings.t` | ✅ | ⬜ pending |
| 16-02-03 | 02 | 2 | PLT-03 | — | N/A | manual | README sichtkontrolle | manual-only | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `Bin/darwin/.gitkeep` — Placeholder für neues Verzeichnis
- [ ] CI-Workflow Verification Steps — `lipo -info`, `file`, `codesign -dv` als Verification

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| README.md erwähnt macOS in Platform-Liste | PLT-03 | Prosa-Text, kein automatischer Test sinnvoll | Sichtkontrolle: README.md enthält "macOS (Universal Binary: Intel + Apple Silicon)" |
| Settings-Seite zeigt Gatekeeper-Hinweis | PLT-03 | Erfordert LMS-Laufzeit mit main::ISMAC=true | LMS auf macOS starten, Settings-Seite ohne Binary prüfen |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 10s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
