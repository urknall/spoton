---
phase: 01
slug: plugin-skeleton-binary-foundation
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-05-27
updated: 2026-05-27
---

# Phase 01 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Perl `Test::More` (bundled with Perl >= 5.10) |
| **Config file** | none — Wave 0 installs |
| **Quick run command** | `prove -v t/` |
| **Full suite command** | `prove -rv t/` |
| **Estimated runtime** | ~5 seconds |

---

## Sampling Rate

- **After every task commit:** Run `prove -v t/`
- **After every plan wave:** Run `prove -rv t/`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 5 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 01-03-T1 | 01-03 | 3 | LMS-04 | — | N/A | unit | `prove -v t/01_install_xml.t` | W0 | pending |
| 01-03-T1 | 01-03 | 3 | LMS-03 | — | N/A | unit | `prove -v t/02_strings.t` | W0 | pending |
| 01-03-T1 | 01-03 | 3 | LMS-05 | — | N/A | unit | `prove -v t/03_convert_conf.t` | W0 | pending |
| 01-03-T1 | 01-03 | 3 | — | — | N/A | unit | `prove -v t/04_types_conf.t` | W0 | pending |
| 01-03-T1 | 01-03 | 3 | LMS-01, LMS-02 | — | N/A | unit | `prove -v t/05_perl_syntax.t` | W0 | pending |
| 01-03-T1 | 01-03 | 3 | LMS-07 | — | N/A | integration | `prove -v t/06_binary_check.t` | W0 | pending |

*Status: pending / green / red / flaky*

---

## Wave 0 Requirements

Test files created in Plan 01-03 Task 1 (synchronized with plan):

- [ ] `t/01_install_xml.t` — install.xml structure: UUID, minVersion, maxVersion, module, category (LMS-04)
- [ ] `t/02_strings.t` — i18n string completeness: EN + DE for all PLUGIN_SPOTON_* keys (LMS-03)
- [ ] `t/03_convert_conf.t` — custom-convert.conf: 4 son pipelines, [spoton] refs, --passthrough (LMS-05)
- [ ] `t/04_types_conf.t` — custom-types.conf: son format, audio/x-sb-spoton MIME
- [ ] `t/05_perl_syntax.t` — perl -c on Plugin.pm, ProtocolHandler.pm, Helper.pm, Settings.pm (LMS-01, LMS-02)
- [ ] `t/06_binary_check.t` — x86_64 binary --check JSON validation; skip_all if binary not yet built (LMS-07)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| LMS plugin manager loads SpotOn after symlink | LMS-01 | Requires running LMS instance | sudo ln -s, restart, check server.log |
| Settings page renders under LMS Settings | LMS-02 | Requires LMS web UI | Navigate to LMS Settings > SpotOn |
| OPML menu shows Binary-Missing hint | LMS-01 | Requires LMS + client | Open SpotOn in LMS menu |
| Binary --check returns JSON on ARM architecture | LMS-06, LMS-07 | Requires ARM hardware or QEMU | Run on target platform after CI build |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 5s
- [ ] `nyquist_compliant: true` set in frontmatter (set, awaiting execution)

**Approval:** pending execution
