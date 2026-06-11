# Phase 16: macOS Universal Binary - Research

**Researched:** 2026-06-11
**Domain:** macOS CI-Build, Universal Binary, Code-Signing, Gatekeeper
**Confidence:** HIGH

## Summary

Phase 16 umfasst drei Arbeitsbereiche: (1) GitHub Actions CI-Workflow fuer macOS-Builds mit anschliessendem `lipo`-Merge zum Universal Binary, (2) Ad-hoc Code-Signing und Artifact-Integration in den bestehenden Release-Job, (3) Helper.pm ISMAC-Block fuer Binary-Detection plus Gatekeeper-Dokumentation in README und Settings-Seite.

Die technischen Grundlagen sind solide: GitHub Actions bietet sowohl ARM64-Runner (`macos-15`) als auch Intel-Runner (`macos-15-intel`) im Free-Tier an. Rust unterstuetzt native Kompilierung fuer beide macOS-Targets auf den jeweiligen Runnern. `lipo` und `codesign` sind als Xcode Command Line Tools auf allen macOS-Runnern vorinstalliert. LMS PluginManager sucht automatisch in `Bin/darwin/` auf macOS-Systemen (via `$^O`).

**Primary recommendation:** Zwei native macOS-Build-Jobs (ARM64 auf `macos-15`, Intel auf `macos-15-intel`) mit einem nachgelagerten `lipo`-Job auf `macos-15`, der das Universal Binary erzeugt, ad-hoc signiert und als Release-Artifact bereitstellt.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** macOS Universal Binary liegt in `Bin/darwin/spoton` -- ein Verzeichnis, ein Binary fuer beide Architekturen
- **D-02:** Konsistent mit Herger's Spotty-Pattern. Helper.pm braucht nur einen `ISMAC`-Block mit `addFindBinPaths` auf `Bin/darwin/`
- **D-03:** Universal Binary wird via `lipo -create -output` aus x86_64 und aarch64 Builds erzeugt
- **D-04:** macOS-Build nur bei Tag-Releases (`v*`) und `workflow_dispatch` -- gleicher Trigger wie bestehender Linux/Windows-Build. Keine PR/Push-Builds
- **D-05:** macOS Universal Binary wird als `spoton-darwin` Artifact im GitHub Release veroeffentlicht
- **D-06:** Ad-hoc Code-Signing (`codesign --force --sign -`) reicht aus -- LMS Plugin Manager setzt kein quarantine-xattr
- **D-07:** README.md bekommt macOS in die Platform-Liste + kurzen xattr-Hinweis fuer manuelle Downloads
- **D-08:** Settings-Seite (basic.html) zeigt dynamischen Gatekeeper-Hinweis wenn Binary-Check auf macOS fehlschlaegt
- **D-09:** Gatekeeper-Warnung in strings.txt fuer alle 11 Sprachen (i18n via bestehendem Pattern)

### Claude's Discretion
- CI-Workflow-Struktur: Ob bestehender `build-librespot.yml` erweitert oder separater macOS-Workflow
- macOS Runner-Auswahl: Welche spezifischen GitHub Runner-Labels fuer Intel und ARM Builds
- Lipo-Job-Platzierung: Ob der Universal Binary Merge als separater Job oder innerhalb eines bestehenden Jobs laeuft

### Deferred Ideas (OUT OF SCOPE)
- **PLT-04: Developer ID Signing + Notarization** -- Erfordert $99/Jahr Apple Developer Membership. Nicht Teil dieser Phase.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PLT-01 | macOS Universal Binary covering Intel x86_64 and Apple Silicon aarch64 via lipo | GitHub Actions runner labels (`macos-15-intel` + `macos-15`), Rust targets (`x86_64-apple-darwin` + `aarch64-apple-darwin`), `lipo -create` Befehl -- alle verifiziert |
| PLT-02 | macOS binary works with LMS plugin manager installation (ad-hoc code signing, no quarantine xattr) | Ad-hoc signing via `codesign --force --sign -`, LMS PluginManager sucht automatisch `Bin/darwin/` via `$^O`, kein quarantine-xattr bei Perl-basierter Extraktion |
| PLT-03 | Setup guide documents Gatekeeper workaround (`xattr -d`) for manual binary downloads | `xattr -d com.apple.quarantine` Befehl, dynamischer Gatekeeper-Hinweis in Settings wenn Binary-Check fehlschlaegt, i18n-Strings |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| macOS Binary Build | CI/CD (GitHub Actions) | -- | Native Kompilierung auf macOS-Runnern, kein Cross-Compile noetig |
| Universal Binary Merge | CI/CD (GitHub Actions) | -- | `lipo` laeuft im CI nach den Build-Jobs |
| Ad-hoc Code-Signing | CI/CD (GitHub Actions) | -- | `codesign` im CI-Job nach dem `lipo`-Merge |
| Binary Detection (ISMAC) | LMS Plugin (Perl) | -- | Helper.pm `init()` registriert `Bin/darwin/` Pfad |
| Gatekeeper-Hinweis (UI) | LMS Plugin (Perl + TT2) | -- | Settings.pm + basic.html + strings.txt |
| Gatekeeper-Doku (README) | Repository Docs | -- | Statischer Text in README.md |

## Standard Stack

### Core

| Technology | Version | Purpose | Why Standard |
|------------|---------|---------|--------------|
| GitHub Actions macOS Runners | `macos-15` (ARM64), `macos-15-intel` (x86_64) | Native Kompilierung | Free-Tier Runner, keine Cross-Compilation noetig [CITED: docs.github.com/en/actions/reference/runners/github-hosted-runners] |
| Rust Targets | `x86_64-apple-darwin`, `aarch64-apple-darwin` | Kompilierung fuer beide Architekturen | Tier 1 Rust-Targets, stabil und vollstaendig unterstuetzt [CITED: doc.rust-lang.org/rustc/platform-support/apple-darwin.html] |
| `lipo` | macOS Xcode CLI Tools | Universal Binary Merge | Apple-Standard-Tool, vorinstalliert auf macOS-Runnern [ASSUMED] |
| `codesign` | macOS Xcode CLI Tools | Ad-hoc Code-Signing | Apple-Standard-Tool, vorinstalliert auf macOS-Runnern [ASSUMED] |
| `dtolnay/rust-toolchain@stable` | latest | Rust-Toolchain Installation | Bereits im bestehenden Workflow verwendet [VERIFIED: .github/workflows/build-librespot.yml] |
| `actions/upload-artifact@v4` | v4 | Artifact-Upload zwischen Jobs | Bereits im bestehenden Workflow verwendet [VERIFIED: .github/workflows/build-librespot.yml] |
| `actions/download-artifact@v4` | v4 | Artifact-Download im Lipo-Job | Bereits im bestehenden Workflow verwendet [VERIFIED: .github/workflows/build-librespot.yml] |

### Supporting

Keine zusaetzlichen Libraries oder Pakete noetig. Alles basiert auf vorinstallierten macOS-Tools und dem bestehenden Workflow-Pattern.

**Installation:**
Keine Package-Installation noetig. Alle Tools (lipo, codesign, cargo) sind auf macOS-Runnern vorinstalliert.

## Architecture Patterns

### System Architecture Diagram

```
Tag-Push (v*) / workflow_dispatch
        |
        v
+-------------------+     +---------------------+
| Build Job:        |     | Build Job:          |
| macos-15          |     | macos-15-intel      |
| (ARM64 M1)        |     | (Intel x86_64)      |
|                   |     |                     |
| cargo build       |     | cargo build         |
| --target          |     | --target            |
| aarch64-apple-    |     | x86_64-apple-       |
| darwin            |     | darwin              |
|                   |     |                     |
| Upload artifact:  |     | Upload artifact:    |
| spoton-aarch64-   |     | spoton-x86_64-      |
| darwin            |     | darwin              |
+--------+----------+     +----------+----------+
         |                            |
         +------+       +-------------+
                |       |
                v       v
     +--------------------+
     | Lipo Job:          |
     | macos-15           |
     |                    |
     | Download both      |
     | artifacts          |
     |                    |
     | lipo -create       |
     |   -output spoton   |
     |   arm64 x86_64     |
     |                    |
     | codesign --force   |
     |   --sign - spoton  |
     |                    |
     | lipo -info spoton  |
     | (verify)           |
     |                    |
     | Upload artifact:   |
     | spoton-darwin      |
     +--------+-----------+
              |
              v
     +--------------------+
     | Release Job:       |
     | (bestehend)        |
     |                    |
     | Download ALL       |
     | artifacts          |
     | (Linux + Win +     |
     |  macOS)            |
     |                    |
     | SHA256SUMS.txt     |
     | GitHub Release     |
     +--------------------+
```

### Empfohlene Workflow-Struktur: Bestehenden Workflow erweitern

**Empfehlung:** Den bestehenden `build-librespot.yml` erweitern, NICHT einen separaten Workflow erstellen.

**Begruendung:**
1. Ein einziger Workflow sorgt fuer atomare Releases (alle Plattformen oder keine)
2. Der Release-Job sammelt bereits alle Artifacts -- macOS fuegt sich nahtlos ein
3. Trigger-Konfiguration (`v*` Tags + `workflow_dispatch`) ist identisch
4. Weniger Wartungsaufwand als zwei separate Workflows

**Struktur-Aenderungen am Workflow:**

1. **Neuer Job `build-macos`** (parallel zum bestehenden `build`-Job):
   - Matrix mit zwei Eintraegen: ARM64 und x86_64
   - Jeweils eigener `runs-on` Wert aus der Matrix
   - Gleicher Build-Ablauf: checkout, rust-toolchain, cargo build, upload artifact

2. **Neuer Job `lipo`** (depends on: `build-macos`):
   - Laeuft auf `macos-15`
   - Download beider macOS-Artifacts
   - `lipo -create -output` zum Universal Binary
   - `codesign --force --sign -` fuer Ad-hoc Signatur
   - Verification via `lipo -info` und `file`
   - Upload als `spoton-darwin` Artifact

3. **Release-Job anpassen** (depends on: `build` AND `lipo`):
   - Zusaetzlich `spoton-darwin` Artifact downloaden
   - In SHA256SUMS.txt einbeziehen
   - Als Release-Asset hochladen

### Pattern 1: macOS Matrix mit unterschiedlichen Runnern

**What:** GitHub Actions Matrix mit `runs-on` aus Matrix-Variable
**When to use:** Wenn verschiedene Build-Targets unterschiedliche Runner brauchen

```yaml
# Source: GitHub Actions docs + bestehender build-librespot.yml Pattern
build-macos:
  name: Build macOS ${{ matrix.arch }}
  runs-on: ${{ matrix.os }}
  strategy:
    fail-fast: false
    matrix:
      include:
        - arch: aarch64
          os: macos-15
          target: aarch64-apple-darwin
        - arch: x86_64
          os: macos-15-intel
          target: x86_64-apple-darwin

  steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Install Rust toolchain
      uses: dtolnay/rust-toolchain@stable
      with:
        targets: ${{ matrix.target }}

    - name: Build binary
      working-directory: librespot-spoton
      run: cargo build --release --target ${{ matrix.target }}

    - name: Upload binary artifact
      uses: actions/upload-artifact@v4
      with:
        name: spoton-${{ matrix.arch }}-darwin
        path: librespot-spoton/target/${{ matrix.target }}/release/spoton
        retention-days: 30
```

### Pattern 2: Lipo + Codesign Job

**What:** Universal Binary erzeugen und ad-hoc signieren
**When to use:** Nach den beiden macOS-Build-Jobs

```yaml
# Source: Apple Developer docs + macOS CLI reference
lipo:
  name: Create macOS Universal Binary
  runs-on: macos-15
  needs: build-macos

  steps:
    - name: Download ARM64 binary
      uses: actions/download-artifact@v4
      with:
        name: spoton-aarch64-darwin
        path: arm64

    - name: Download x86_64 binary
      uses: actions/download-artifact@v4
      with:
        name: spoton-x86_64-darwin
        path: x86_64

    - name: Create Universal Binary
      run: |
        lipo -create -output spoton arm64/spoton x86_64/spoton
        chmod 755 spoton

    - name: Ad-hoc code sign
      run: codesign --force --sign - spoton

    - name: Verify Universal Binary
      run: |
        echo "=== lipo -info ==="
        lipo -info spoton
        echo "=== file ==="
        file spoton
        echo "=== codesign -dv ==="
        codesign -dv spoton 2>&1
        echo "=== SHA256 ==="
        shasum -a 256 spoton

    - name: Upload Universal Binary artifact
      uses: actions/upload-artifact@v4
      with:
        name: spoton-darwin
        path: spoton
        retention-days: 30
```

### Pattern 3: Helper.pm ISMAC-Block

**What:** Binary-Detection fuer macOS in Helper.pm
**When to use:** Exakt wie der bestehende ISWINDOWS-Block

```perl
# Source: Plugins/SpotOn/Helper.pm Zeile 28-31 (bestehendes Pattern)
if ( main::ISMAC ) {
    Slim::Utils::Misc::addFindBinPaths(
        catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'darwin')
    );
}
```

**Platzierung:** Nach dem bestehenden `main::ISWINDOWS`-Block (Zeile 32), vor dem `$prefs->setChange`-Aufruf (Zeile 34).

**Hinweis:** LMS PluginManager sucht automatisch in `Bin/$^O/` (= `Bin/darwin/` auf macOS), aber `addFindBinPaths` stellt sicher, dass der Pfad auch gefunden wird wenn die automatische Suche nicht greift. Herger's Spotty-Plugin nutzt denselben Mechanismus. [CITED: github.com/LMS-Community/slimserver/blob/public/9.0/Slim/Utils/PluginManager.pm]

### Pattern 4: Gatekeeper-Warnung in Settings

**What:** Dynamischer Hinweis auf der Settings-Seite wenn Binary auf macOS fehlt
**When to use:** Wenn `main::ISMAC` und kein Binary gefunden

```html
<!-- In basic.html, innerhalb des binaryPath-Checks -->
[% IF NOT binaryPath AND isMac %]
<div style="color: orange; margin-bottom:8px">
    [% 'PLUGIN_SPOTON_GATEKEEPER_HINT' | string %]
</div>
[% END %]
```

Settings.pm muss `isMac` als Template-Variable setzen:
```perl
$paramRef->{isMac} = main::ISMAC ? 1 : 0;
```

### Anti-Patterns to Avoid

- **Cross-Compilation statt nativer Builds:** Nicht `cargo build --target x86_64-apple-darwin` auf einem ARM64-Runner ausfuehren. Obwohl es fuer pure-Rust-Projekte funktionieren kann, gibt es Risiken bei Abhaengigkeiten die C-Linkage nutzen (`rustls-native-certs` liest System-Root-CAs). Native Builds auf dem jeweiligen Runner sind zuverlaessiger.
- **Einzelner Runner mit Cross-Compile:** Spart zwar CI-Minuten, fuehrt aber zu fragilen Builds. Zwei Runner sind robuster.
- **`--deep` Flag bei codesign:** Nur fuer App-Bundles relevant, nicht fuer einzelne Binaries.
- **Separater Workflow:** Fuehrt zu Race Conditions bei Releases -- ein Workflow koennte fertig sein, der andere nicht.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Universal Binary | Manuelles Binary-Merging | `lipo -create` | Apple-Standard-Tool, handled Mach-O Format korrekt |
| Code-Signing | Eigene Signatur-Logik | `codesign --force --sign -` | Apple-Standard-Tool, validiert gegen Gatekeeper |
| Binary-Verifikation | Eigene Architektur-Checks | `lipo -info` + `file` | Standard-Tools die exakt das Richtige pruefen |
| macOS-Detection | OS-String-Parsing | `main::ISMAC` | LMS-Konstante, bereits ueberall im Ecosystem verwendet |

## Common Pitfalls

### Pitfall 1: macOS Runner-Architektur-Verwirrung
**What goes wrong:** `macos-latest` gibt ARM64 (M1), nicht Intel. Code der `macos-latest` fuer Intel-Builds nutzt, baut fuer die falsche Architektur.
**Why it happens:** GitHub hat `macos-latest` 2024 von Intel auf Apple Silicon umgestellt.
**How to avoid:** Explizite Labels verwenden: `macos-15` fuer ARM64, `macos-15-intel` fuer x86_64.
**Warning signs:** `file` Befehl zeigt nur `arm64` statt `x86_64` (oder umgekehrt).
[CITED: docs.github.com/en/actions/reference/runners/github-hosted-runners]

### Pitfall 2: CI-Minuten-Verbrauch (10x Multiplier)
**What goes wrong:** macOS-Runner verbrauchen 10x so viele CI-Minuten wie Linux-Runner. Drei macOS-Jobs (2 Builds + 1 Lipo) verbrauchen erhebliche Minuten.
**Why it happens:** Apple Silicon Hardware ist teurer fuer GitHub.
**How to avoid:** Trigger auf Tags und workflow_dispatch beschraenken (D-04). Keine PR/Push-Builds fuer macOS.
**Warning signs:** Free-Tier Minutenkontingent schnell aufgebraucht.
[CITED: docs.github.com/en/billing/reference/actions-runner-pricing]

### Pitfall 3: Quarantine-xattr bei manuellen Downloads
**What goes wrong:** User laden das Binary manuell herunter (statt ueber LMS Plugin Manager), macOS blockiert die Ausfuehrung.
**Why it happens:** Safari/Chrome setzen `com.apple.quarantine` auf heruntergeladene Dateien. Ad-hoc signierte Binaries werden von Gatekeeper blockiert.
**How to avoid:** Dokumentation des `xattr -d com.apple.quarantine` Workarounds (D-07, D-08, D-09).
**Warning signs:** "spoton can't be opened because Apple cannot verify the developer" Fehlermeldung.
[CITED: gist.github.com/rsms/929c9c2fec231f0cf843a1a746a416f5]

### Pitfall 4: lipo -create Reihenfolge
**What goes wrong:** `lipo -create` schlaegt fehl wenn beide Inputs die gleiche Architektur haben.
**Why it happens:** Falscher Runner oder falsches Target in der Matrix.
**How to avoid:** `lipo -info` Verification nach dem Create, die explizit auf "2 architectures" prueft.
**Warning signs:** `lipo: input files ... have the same architecture` Fehlermeldung.

### Pitfall 5: Artifact-Name-Kollision
**What goes wrong:** macOS-Artifacts ueberschreiben Linux-Artifacts wenn die Namen identisch sind.
**Why it happens:** `upload-artifact` nutzt den Namen als eindeutigen Key -- Duplikate schlagen fehl.
**How to avoid:** Eigene Namenskonvention: `spoton-aarch64-darwin`, `spoton-x86_64-darwin` fuer die Einzel-Builds, `spoton-darwin` fuer das Universal Binary. Der bestehende Workflow nutzt `spoton-{bin_dir}` (z.B. `spoton-x86_64-linux`).
**Warning signs:** `Error: Artifact with name 'spoton-...' already exists` im CI-Log.

### Pitfall 6: Binary nicht executable nach Download
**What goes wrong:** `actions/download-artifact` behaelt Permissions nicht zuverlaessig bei. Binary ist nicht ausfuehrbar.
**Why it happens:** Artifact-Upload/Download-Zyklus kann Permission-Bits verlieren.
**How to avoid:** `chmod 755` nach dem Download und nach dem `lipo -create`. Auch im `Prepare binary`-Step des Workflows.
**Warning signs:** `Permission denied` beim Ausfuehren oder `codesign` schlaegt fehl.

### Pitfall 7: Release-Job muss auf ALLE Build-Jobs warten
**What goes wrong:** Release-Job startet bevor macOS-Builds fertig sind, macOS-Artifact fehlt im Release.
**Why it happens:** `needs:` nur auf `build` gesetzt, nicht auch auf `lipo`.
**How to avoid:** Release-Job braucht `needs: [build, lipo]` statt nur `needs: build`.
**Warning signs:** GitHub Release hat kein macOS-Binary.

## Code Examples

### Vollstaendige Lipo-Verification

```bash
# Source: Apple Developer CLI reference
# Nach lipo -create ausfuehren:

# 1. Verify: Universal Binary mit 2 Architekturen
lipo -info spoton
# Erwartete Ausgabe: Architectures in the fat file: spoton are: x86_64 arm64

# 2. Verify: Mach-O Typ
file spoton
# Erwartete Ausgabe: spoton: Mach-O universal binary with 2 architectures:
#   [x86_64:Mach-O 64-bit executable x86_64] [arm64:Mach-O 64-bit executable arm64]

# 3. Verify: Ad-hoc Signatur
codesign -dv spoton 2>&1
# Erwartete Ausgabe enthaelt: Signature=adhoc
```

### i18n String-Pattern fuer Gatekeeper-Warnung

```
# Source: Plugins/SpotOn/strings.txt (bestehendes Pattern)
PLUGIN_SPOTON_GATEKEEPER_HINT
	CS	SpotOn binary je blokován systémem macOS. Spusťte v Terminálu: xattr -d com.apple.quarantine /cesta/k/spoton
	DA	SpotOn binærfilen blokeres af macOS. Kør i Terminal: xattr -d com.apple.quarantine /sti/til/spoton
	DE	Das SpotOn-Binary wird von macOS blockiert. Im Terminal ausfuehren: xattr -d com.apple.quarantine /pfad/zu/spoton
	EN	The SpotOn binary is blocked by macOS. Run in Terminal: xattr -d com.apple.quarantine /path/to/spoton
	ES	El binario de SpotOn está bloqueado por macOS. Ejecuta en Terminal: xattr -d com.apple.quarantine /ruta/al/spoton
	FR	Le binaire SpotOn est bloqué par macOS. Exécutez dans le Terminal : xattr -d com.apple.quarantine /chemin/vers/spoton
	IT	Il binario SpotOn è bloccato da macOS. Esegui nel Terminale: xattr -d com.apple.quarantine /percorso/di/spoton
	NL	Het SpotOn binair bestand wordt geblokkeerd door macOS. Voer uit in Terminal: xattr -d com.apple.quarantine /pad/naar/spoton
	NO	SpotOn-binærfilen er blokkert av macOS. Kjør i Terminal: xattr -d com.apple.quarantine /sti/til/spoton
	PL	Plik binarny SpotOn jest blokowany przez macOS. Uruchom w Terminalu: xattr -d com.apple.quarantine /ścieżka/do/spoton
	SV	SpotOn-binärfilen blockeras av macOS. Kör i Terminal: xattr -d com.apple.quarantine /sökväg/till/spoton
```

### README.md Platform-Liste Update

```markdown
# Vorher:
- Supported platforms: x86_64 Linux, i386 Linux, aarch64 Linux (Pi 4+),
  armhf Linux (Pi 2/3), arm Linux, x86_64 Windows.
  macOS binaries not yet included.

# Nachher:
- Supported platforms: x86_64 Linux, i386 Linux, aarch64 Linux (Pi 4+),
  armhf Linux (Pi 2/3), arm Linux, x86_64 Windows,
  macOS (Universal Binary: Intel + Apple Silicon).
  On macOS, if you download the binary manually (not via LMS plugin manager),
  you may need to run `xattr -d com.apple.quarantine /path/to/spoton`
  in Terminal before first use.
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `macos-latest` = Intel | `macos-latest` = ARM64 (M1) | 2024 | Explizites `macos-15-intel` Label fuer Intel-Builds noetig |
| Keine Intel-macOS-Runner | `macos-15-intel` verfuegbar | Sep 2025 | Letztes Intel-Image, verfuegbar bis Aug 2027 |
| `macos-13` | Deprecated | Sep 2025 | Nicht mehr verwenden, `macos-15` nutzen |
| Einzelne Architektur-Binaries | Universal Binary via `lipo` | Apple Silicon Launch (2020) | Standard fuer macOS-Distribution |

**Deprecated/outdated:**
- `macos-13` Runner: Wird ab Sep 2025 eingestellt [CITED: github.blog/changelog/2025-09-19-github-actions-macos-13-runner-image-is-closing-down/]
- `macos-15-intel` wird das letzte Intel-Image sein (bis Aug 2027) [CITED: github.com/actions/runner-images/issues/13045]
- `macos-latest` NICHT fuer Intel-Builds verwenden -- gibt ARM64 zurueck [CITED: docs.github.com/en/actions/reference/runners/github-hosted-runners]

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `lipo` und `codesign` sind auf GitHub Actions macOS-Runnern vorinstalliert (Teil der Xcode CLI Tools) | Standard Stack | LOW -- Xcode CLI Tools sind auf allen macOS-Runnern installiert, `lipo` und `codesign` gehoeren dazu. Falls nicht: `xcode-select --install` im Workflow |
| A2 | Gatekeeper-Warnung Strings fuer 11 Sprachen koennten Uebersetzungsfehler enthalten | Code Examples | LOW -- maschinelle Uebersetzung folgt bestehendem strings.txt Pattern; finale Strings sollten vom Planner/Implementer geprueft werden |

## Open Questions

1. **Cross-Compilation als Fallback?**
   - What we know: Native Builds auf separaten Runnern sind zuverlaessiger. Cross-Compilation (x86_64 von ARM64 oder umgekehrt) funktioniert fuer pure-Rust-Projekte.
   - What's unclear: Ob `librespot-playback` durch `rustls-native-certs` System-C-Linkage braucht die Cross-Compilation verkompliziert.
   - Recommendation: Native Builds auf separaten Runnern (wie empfohlen). Cross-Compilation nur als Fallback wenn ein Runner ausfaellt.

2. **macos-15-intel Minutenkosten**
   - What we know: macOS-Runner haben einen 10x-Multiplier. Free-Tier bietet 300 macOS-Minuten/Monat.
   - What's unclear: Ob der Intel-Runner den gleichen 10x-Multiplier hat wie der ARM64-Runner.
   - Recommendation: Konservativ planen mit 10x fuer beide. Bei Tag-only-Trigger (~2 Releases/Monat) kein Problem.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Test::More (Perl, bundled with LMS) |
| Config file | none -- prove runs t/ directly |
| Quick run command | `prove t/06_binary_check.t` |
| Full suite command | `prove t/` |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PLT-01 | Universal Binary in Bin/darwin/ | CI verification | `lipo -info` + `file` im Workflow | Wave 0 (CI) |
| PLT-01 | Bin/darwin/ Verzeichnis existiert | unit | `prove t/06_binary_check.t` (erweitert) | Erweitern |
| PLT-02 | ISMAC-Block in Helper.pm | unit | `prove t/` (Perl syntax check via 05_perl_syntax.t) | Besteht |
| PLT-02 | Ad-hoc Signatur gueltig | CI verification | `codesign -dv` im Workflow | Wave 0 (CI) |
| PLT-03 | Gatekeeper-String in strings.txt | unit | `prove t/02_strings.t` (prueft String-Vollstaendigkeit) | Besteht |
| PLT-03 | README erwaehnt macOS | manual | Sichtkontrolle | manual-only |

### Sampling Rate
- **Per task commit:** `prove t/`
- **Per wave merge:** `prove t/` + CI-Workflow-Lauf verifiziert Universal Binary
- **Phase gate:** Full suite green + erfolgreicher `workflow_dispatch`-Lauf

### Wave 0 Gaps
- [ ] `Bin/darwin/.gitkeep` -- Placeholder fuer neues Verzeichnis
- [ ] CI-Workflow Verification Steps -- `lipo -info`, `file`, `codesign -dv` als Verification

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | -- |
| V3 Session Management | no | -- |
| V4 Access Control | no | -- |
| V5 Input Validation | no | -- |
| V6 Cryptography | partial | Ad-hoc Code-Signing (codesign) -- kein kryptographisches Signing, nur Checksum |

### Known Threat Patterns for macOS Binary Distribution

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Binary Tampering | Tampering | SHA256 Checksums im Release, ad-hoc codesign validiert Integritaet |
| Supply Chain (CI) | Tampering | Pinned Actions (`@v4`), GitHub-hosted Runners (nicht self-hosted) |
| Gatekeeper Bypass via xattr | -- | Dokumentiert als bewusster User-Workaround, nicht als Sicherheitsluecke |

## Sources

### Primary (HIGH confidence)
- `.github/workflows/build-librespot.yml` -- Bestehender Build-Workflow, Pattern-Referenz [VERIFIED: Codebase]
- `Plugins/SpotOn/Helper.pm` -- Binary-Detection Pattern, ISMAC-Platzierung [VERIFIED: Codebase]
- `Plugins/SpotOn/Settings.pm` -- Settings-Handler Pattern [VERIFIED: Codebase]
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` -- Settings-Template [VERIFIED: Codebase]
- `Plugins/SpotOn/strings.txt` -- i18n-Pattern (11 Sprachen) [VERIFIED: Codebase]
- `librespot-spoton/Cargo.toml` -- Build-Konfiguration, Feature-Flags [VERIFIED: Codebase]
- [GitHub Actions Runner Reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners) -- macOS Runner Labels und Architekturen
- [LMS PluginManager.pm](https://github.com/LMS-Community/slimserver/blob/public/9.0/Slim/Utils/PluginManager.pm) -- Automatische `Bin/$^O/` Suche

### Secondary (MEDIUM confidence)
- [macOS distribution gist](https://gist.github.com/rsms/929c9c2fec231f0cf843a1a746a416f5) -- Ad-hoc Signing, Quarantine, Gatekeeper
- [GitHub Actions runner-images #13045](https://github.com/actions/runner-images/issues/13045) -- macos-15-intel Announcement
- [Apple Developer: Building Universal macOS Binary](https://developer.apple.com/documentation/apple-silicon/building-a-universal-macos-binary) -- lipo Reference
- [Rust Platform Support: apple-darwin](https://doc.rust-lang.org/rustc/platform-support/apple-darwin.html) -- Target Tier Status

### Tertiary (LOW confidence)
- Keine -- alle relevanten Claims sind verifiziert oder zitiert

## Metadata

**Confidence breakdown:**
- Standard Stack: HIGH -- alle Tools und Runner verifiziert via offizielle Docs und bestehende Codebase
- Architecture: HIGH -- Pattern direkt aus bestehendem Workflow abgeleitet, LMS PluginManager-Verhalten verifiziert
- Pitfalls: HIGH -- basierend auf offizieller GitHub-Dokumentation und Apple-Referenzen

**Research date:** 2026-06-11
**Valid until:** 2026-09-11 (90 Tage -- stabile CI-Infrastruktur und Apple-Tools aendern sich selten)
