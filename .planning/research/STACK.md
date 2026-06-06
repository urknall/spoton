# Technology Stack: SpotOn v1.3 (Polish & Publish)

**Project:** SpotOn — Spotify plugin for Lyrion Music Server
**Researched:** 2026-06-06
**Scope:** Stack additions/changes for v1.3 milestone only. v1.0 and v1.1 stacks are validated and unchanged.
**Confidence:** HIGH for CI, LMS repo submission format, and Spotify quota reality. MEDIUM for macOS signing approach (open-source precedent clear, Apple details verified).

---

## What This Document Covers

Four new capability areas for v1.3:

1. **macOS Universal Binary** — Cross-compilation for darwin-x86_64 and darwin-aarch64, lipo merging
2. **LMS Community Repo Submission** — Plugin registry format, include.json PR process
3. **GitHub Actions CI for Perl Tests** — Running `prove t/` on every push
4. **Spotify Extended Quota Application** — What is realistically achievable

Existing stack (Perl/LMS API, librespot 0.8.0, Spotify Web API, ZeroConf/Keymaster, cross-rs for Linux, cargo-xwin for Windows) is unchanged.

---

## Area 1: macOS Universal Binary

### Current State

No darwin binaries exist yet. `Plugins/SpotOn/Bin/` has 6 Linux directories and `x86_64-win64/` but no darwin directories. The existing `build-librespot.yml` CI handles 6 Linux musl targets + Windows via `ubuntu-latest` runners. macOS was deferred in v1.1.

### Strategy: Two Native Runners + lipo

Do NOT use osxcross/cross-rs for macOS. Apple's SDK licensing prevents distributable cross-compilation toolchains. The correct pattern is:

1. Build `x86_64-apple-darwin` on `macos-15-intel` runner
2. Build `aarch64-apple-darwin` on `macos-latest` runner (ARM64)
3. Combine with `lipo -create -output spoton spoton-x86_64 spoton-arm64`
4. Distribute the single universal binary

This is exactly how open-source Rust projects (librespot itself via Homebrew, Alacritty, dovi_tool) produce macOS binaries.

### GitHub Actions Runner Labels (as of June 2026)

| Target | Runner Label | Architecture | Cost on Public Repos |
|--------|-------------|--------------|----------------------|
| `x86_64-apple-darwin` | `macos-15-intel` | Intel x86_64 | Free |
| `aarch64-apple-darwin` | `macos-latest` (= macos-15 ARM) | Apple Silicon M1 | Free |

**Critical note on runner lifecycle:** `macos-13` was deprecated September 2025 and retired December 2025. `macos-15-intel` is the current Intel label. Apple has end-of-lifed x86_64 macOS runners — the last Intel runner image will be `macos-15-intel`, which GitHub will retire in approximately August 2027. For now it is available and free on public repos.

`macos-latest` currently maps to `macos-15` (ARM64). This is the right runner for `aarch64-apple-darwin`.

### Build Steps for CI

```yaml
jobs:
  build-macos:
    strategy:
      matrix:
        include:
          - target: x86_64-apple-darwin
            runner: macos-15-intel
            bin_dir: darwin-x86_64
          - target: aarch64-apple-darwin
            runner: macos-latest
            bin_dir: darwin-aarch64
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}
      - name: Build
        working-directory: librespot-spoton
        run: cargo build --release --target ${{ matrix.target }}
      - uses: actions/upload-artifact@v4
        with:
          name: spoton-${{ matrix.bin_dir }}
          path: librespot-spoton/target/${{ matrix.target }}/release/spoton

  lipo-universal:
    needs: build-macos
    runs-on: macos-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: artifacts
      - name: Create universal binary
        run: |
          lipo -create \
            artifacts/spoton-darwin-x86_64/spoton \
            artifacts/spoton-darwin-aarch64/spoton \
            -output spoton-universal
      - uses: actions/upload-artifact@v4
        with:
          name: spoton-darwin-universal
          path: spoton-universal
```

After lipo, copy to `Plugins/SpotOn/Bin/darwin-x86_64/spoton` AND `Plugins/SpotOn/Bin/darwin-aarch64/spoton` (LMS uses its own arch-detection; both directories get the universal binary). Alternatively, ship the universal binary in a single `darwin/` directory if Helper.pm is extended to look there.

### Code Signing and Gatekeeper

**Reality for open-source CLI tools distributed outside the Mac App Store:**

macOS Gatekeeper since Catalina (10.15) blocks unsigned binaries with "can't be opened because Apple cannot check it for malicious software." However:

- Full notarization (Apple Developer ID, $99/year) requires wrapping CLI binaries in a DMG or PKG — stapling to a raw Mach-O is not supported by Apple's tooling.
- **Ad-hoc signing** (`codesign --sign -`) is sufficient to prevent Gatekeeper quarantine for binaries that users download via `curl`/`wget` rather than a browser. Homebrew applies ad-hoc signatures to all poured bottles (including librespot).
- macOS Sequoia 15.x (released 2024) tightened Gatekeeper further, but the user workaround (`xattr -d com.apple.quarantine /path/to/binary`) remains functional.
- Since SpotOn binaries are distributed inside a ZIP installed via LMS's plugin manager (which uses its own HTTP client without quarantine attribution), the binary does NOT receive the quarantine xattr and runs without Gatekeeper interference.

**Recommendation:** Ad-hoc signing via `codesign --sign -` in CI is sufficient. Full Apple Developer ID notarization is not needed for LMS plugin distribution. The quarantine attribute is not set by LMS's plugin downloader.

Add to the lipo step:
```bash
codesign --sign - spoton-universal
```

**What NOT to do:** Do not apply for Apple Developer ID ($99/year) solely for this use case. Do not ship a DMG wrapper — LMS plugin installs from ZIP only.

### Bin Directory Changes Needed

Two new directories must be created:
```
Plugins/SpotOn/Bin/darwin-x86_64/spoton
Plugins/SpotOn/Bin/darwin-aarch64/spoton
```

Helper.pm already knows these paths from the v1.1 research (`Bin/darwin-x86_64/` and `Bin/darwin-aarch64/`). The universal binary can be copied to both.

---

## Area 2: LMS Community Repo Submission

### How the Registry Works

The LMS Community repository aggregates plugin metadata from plugin authors' own hosted repo.xml files. The flow is:

1. Each plugin author hosts their own `repo.xml` at a stable URL (e.g., raw GitHub URL or web server).
2. The central `include.json` at `LMS-Community/lms-plugin-repository` contains a list of these URLs.
3. A GitHub Actions workflow runs automatically every few hours, fetches all listed repo.xml files, and merges them into the public `extensions.xml` shown in LMS's "Plugin Library" UI.

### repo.xml Format

SpotOn already has a `repo.xml` (used in v1.0/v1.1). The format follows this structure, confirmed against Spotty's production entry:

```xml
<?xml version="1.0" encoding="utf-8"?>
<extensions>
  <plugins>
    <plugin name="SpotOn" version="1.3.0" minTarget="8.0" maxTarget="*">
      <title lang="EN">SpotOn</title>
      <title lang="DE">SpotOn</title>
      <desc lang="EN">Spotify plugin for Lyrion Music Server. Streaming, Connect, Library, Browse.</desc>
      <desc lang="DE">Spotify-Plugin für Lyrion Music Server. Streaming, Connect, Bibliothek, Browse.</desc>
      <changes>v1.3.0: macOS binaries, Like button, Connect credential isolation.</changes>
      <category>musicservices</category>
      <url>https://github.com/OWNER/spoton/releases/download/v1.3.0/SpotOn-v1.3.0.zip</url>
      <icon>https://raw.githubusercontent.com/OWNER/spoton/main/Plugins/SpotOn/HTML/EN/plugins/SpotOn/html/images/icon.png</icon>
      <creator>Marek Stiefenhofer</creator>
      <email>sti@posteo.de</email>
      <sha>SHA1_CHECKSUM_HERE</sha>
    </plugin>
  </plugins>
</extensions>
```

**Required attributes:**
- `name` — must match Perl package naming (`Slim::Plugin::SpotOn` → `name="SpotOn"`)
- `version` — determines upgrade eligibility in LMS
- `minTarget` — minimum LMS version (use `"8.0"` for SpotOn)
- `maxTarget` — `"*"` means all future versions

**Required elements:**
- `url` — points to the zip file (GitHub Release URL is ideal)
- `sha` — SHA1 checksum of the zip (NOT SHA256 — LMS uses SHA1)
- `category` — use `"musicservices"` (confirmed from Spotty entry)

**Optional but strongly recommended:**
- `title` and `desc` in at least EN (multilingual supported)
- `changes` — release notes (single `<changes>` element, no lang attribute in Spotty's format)
- `icon` — plugin icon URL
- `creator` and `email`

### Submission Process

1. Ensure SpotOn's `repo.xml` is hosted at a stable public URL (already done — GitHub raw URL or releases).
2. Submit a PR to `LMS-Community/lms-plugin-repository` that adds the repo.xml URL to `include.json`:
   ```json
   "repositories": [
     ...existing entries...,
     "https://raw.githubusercontent.com/OWNER/spoton/main/repo.xml"
   ]
   ```
3. The CI auto-validates and merges. No formal code review documented; the repo appears to operate on a "add and the bot does the rest" model.
4. After merge, LMS users see SpotOn in their Plugin Library within a few hours.

**What NOT to do:**
- Do not edit `extensions.xml` directly — it is auto-generated.
- Do not host repo.xml at a URL that changes per release — the URL in include.json must be stable and always point to the current version.

---

## Area 3: GitHub CI for Perl Tests

### Current State

`build-librespot.yml` exists and handles Rust binary builds. No Perl test CI exists yet.

### What the Tests Need

All 12 test files (`t/01_install_xml.t` through `t/12_protocol_rename.t`) use inline LMS module stubs written into `File::Temp` directories. No real LMS installation is needed. Tests run with `prove t/` from the project root — verified locally: 230 tests pass in ~0.6 seconds.

The tests require:
- Perl with `Test::More` (standard Perl core since 5.8)
- `File::Temp`, `File::Path`, `File::Basename`, `Cwd` — all Perl core modules
- No CPAN installs needed

### Recommended Workflow: `.github/workflows/perl-tests.yml`

```yaml
name: Perl Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    name: prove t/ (Perl ${{ matrix.perl-version }})
    runs-on: ubuntu-latest
    strategy:
      matrix:
        perl-version: ['5.36', '5.38']

    steps:
      - uses: actions/checkout@v4

      - name: Set up Perl
        uses: shogo82148/actions-setup-perl@v1
        with:
          perl-version: ${{ matrix.perl-version }}

      - name: Run tests
        run: prove -lv t/
```

**Key choices:**
- `shogo82148/actions-setup-perl@v1` — the standard action for Perl CI, widely used (MEDIUM confidence from multiple community sources). Installs any Perl version without CPAN setup.
- `prove -lv t/` — `-l` adds lib/ to `@INC`, `-v` verbose output. SpotOn tests don't use lib/ but `-l` is harmless.
- Perl versions `5.36` and `5.38`: LMS 9.x ships approximately Perl 5.38. Testing both `5.36` (LMS 8.x era) and `5.38` (LMS 9.x) covers the compatibility range without running ancient Perl.
- `ubuntu-latest` only — no macOS or Windows runners needed for Perl tests.

**What NOT to add:**
- Perl::Critic linting — not bundled with LMS, adds CPAN dependency, overkill for a plugin project
- cpanm/carton dependency install steps — no external deps needed
- Windows or macOS Perl runners — unnecessary; LMS plugin logic is OS-agnostic Perl

---

## Area 4: Spotify Extended Quota Application

### Current Reality (HIGH confidence — verified against official Spotify docs, May 2025 policy update)

Extended Quota Mode requirements as of May 15, 2025:

| Requirement | SpotOn Status |
|-------------|---------------|
| Legally registered business or organisation | **Fails** — individual developer |
| Operating an active, launched service | Passes (plugin is public) |
| Minimum 250,000 Monthly Active Users | **Fails** — LMS community is ~10K-50K users total |
| Available in key Spotify markets | Partially passes |
| Company email address for application | **Fails** — individual |

**The application path is currently blocked for SpotOn.** Spotify tightened requirements in March 2025, explicitly requiring established businesses with 250K MAUs. Less than 1% of applications previously granted extended access are affected, and new applications from individuals are not accepted.

The development mode restrictions (5 test users, lower rate limits, removed batch endpoints) will persist unless Extended Quota is granted.

### What "Preparing the Application" Actually Means for v1.3

Given the above, "Spotify Extended Quota Client-ID Antrag vorbereiten" for v1.3 should be scoped as **documentation and groundwork only**, not an actual application submission:

1. **Document what Extended Quota would unlock:** Higher rate limits, unlimited test users, restored batch endpoints (currently removed in dev mode since Feb 2026).
2. **Track user adoption:** If SpotOn reaches significant adoption (Community Repo listing helps), document it. MAU tracking is not available without Extended Quota, creating a chicken-and-egg problem.
3. **Research alternative Client IDs:** The memory context mentions `d420a117...` (ncspot's extended-quota Client ID, used by ncspot as bundled). This is another project's approved ID — using it without permission violates Spotify ToS. Do not pursue this path.
4. **Write the application draft:** Prepare a `SPOTIFY_QUOTA_APPLICATION.md` with app description, use case, user benefit narrative, ready to submit if circumstances change (e.g., LMS Community Repo adoption boosts visibility, a supporting organization emerges).

**What NOT to do:**
- Do not submit the application now — it will be auto-rejected.
- Do not bundle another project's Client ID to bypass dev mode restrictions.
- Do not wait for Extended Quota before shipping v1.3 — dev mode is functional for all currently implemented features (search, library, player control, all use single-fetch endpoints that still work).

### Dev Mode Workaround That IS Available

The Feb 2026 restrictions removed batch endpoints and browse/categories but did NOT remove the endpoints SpotOn actually uses for its core features. The dual-token routing (own Client ID for me/* endpoints, bundled Keymaster ID for browse) already addresses the rate-limit pressure. Dev mode is sufficient for SpotOn's feature set.

---

## Version Summary for v1.3 New Tooling

| Tool | Version | Purpose | Notes |
|------|---------|---------|-------|
| `shogo82148/actions-setup-perl` | `@v1` (current) | Perl test CI | No CPAN deps needed |
| `macos-15-intel` runner | GitHub-hosted | x86_64 macOS build | Last Intel runner; retires ~Aug 2027 |
| `macos-latest` runner | GitHub-hosted (M1) | aarch64 macOS build | Free on public repos |
| `lipo` | macOS system tool | Universal binary merger | Always available on macOS runners |
| `codesign --sign -` | macOS system tool | Ad-hoc signing | Prevents quarantine without paid cert |

---

## Integration with Existing Build System

The existing `build-librespot.yml` handles 6 Linux targets + Windows on `ubuntu-latest`. macOS is additive:

- Add macOS matrix entries to `build-librespot.yml` (or create separate `build-macos.yml`)
- Add a lipo job that depends on both macOS build jobs
- Add `Plugins/SpotOn/Bin/darwin-x86_64/.gitkeep` and `darwin-aarch64/.gitkeep` as placeholders (matching existing pattern for other Bin dirs)
- Release job already uses `softprops/action-gh-release@v2` and artifact collection — no changes needed there

Add `perl-tests.yml` as a completely separate workflow (not part of the binary build) triggered on push/PR.

---

## What NOT to Add

| Category | Avoid | Reason |
|----------|-------|--------|
| macOS signing | Apple Developer ID ($99/yr) | Not needed; LMS installer doesn't set quarantine xattr |
| macOS signing | Full notarization | Requires DMG/PKG wrapper; incompatible with LMS plugin ZIP format |
| macOS cross-compile | osxcross on ubuntu-latest | Apple SDK licensing; native runners are simpler and free |
| macOS cross-compile | setup-osxcross action | Same issue; first-run build takes ~20min; native runners avoid it |
| Perl CI | Perl::Critic | Not bundled with LMS; CPAN dependency; not in project constraints |
| Perl CI | Windows/macOS Perl runners | Plugin Perl logic is OS-agnostic; ubuntu-only sufficient |
| Perl CI | Perl 5.10 testing | LMS floor but no GitHub runner ships 5.10; 5.36 is the practical minimum |
| LMS repo | Direct extensions.xml edit | Auto-generated; only include.json PR needed |
| Spotify | Extended Quota application (now) | Hard-fails org/MAU requirements; prepare doc only |
| Spotify | Third-party Client ID bundling | ToS violation; use own dev-mode Client ID |

---

## Sources

- GitHub Actions runner labels: https://docs.github.com/en/actions/reference/runners/github-hosted-runners (HIGH — official)
- macos-13 deprecation timeline: https://github.blog/changelog/2025-09-19-github-actions-macos-13-runner-image-is-closing-down/ (HIGH)
- macos-15-intel availability: https://github.com/actions/runner-images/issues/13045 (HIGH)
- macOS x86_64 end-of-life roadmap: https://github.com/actions/runner-images/issues/13027 (HIGH)
- lipo universal binary pattern: multiple Rust project issues/PRs (MEDIUM — community-verified pattern)
- macOS Gatekeeper / code signing requirements: https://tuist.dev/blog/2024/12/31/signing-macos-clis (MEDIUM)
- macOS Sequoia Gatekeeper tightening: https://hackaday.com/2024/11/01/apple-forces-the-signing-of-applications-in-macos-sequoia-15-1/ (MEDIUM)
- Homebrew ad-hoc codesign: https://github.com/Homebrew/brew/issues/9082 (MEDIUM)
- LMS plugin repo format: https://lyrion.org/reference/repository-dev/ (HIGH — official)
- LMS include.json structure: https://github.com/LMS-Community/lms-plugin-repository/blob/master/include.json (HIGH)
- Spotty repo.xml reference: http://www.herger.net/slim-plugins/repo.xml (HIGH — production example)
- shogo82148/actions-setup-perl: https://github.com/marketplace/actions/setup-perl-environment (MEDIUM)
- Spotify Extended Quota requirements: https://developer.spotify.com/documentation/web-api/concepts/quota-modes (HIGH — official)
- Spotify April 2025 criteria update: https://developer.spotify.com/blog/2025-04-15-updating-the-criteria-for-web-api-extended-access (HIGH — official)
- SpotOn codebase: /home/sti/spoton (HIGH — direct inspection of Bin structure, existing CI, test suite)

---

*Stack research for: SpotOn v1.3 — Polish & Publish*
*Researched: 2026-06-06*
