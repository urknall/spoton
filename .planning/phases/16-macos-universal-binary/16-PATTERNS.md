# Phase 16: macOS Universal Binary - Pattern Map

**Mapped:** 2026-06-11
**Files analyzed:** 6 (new/modified files)
**Analogs found:** 6 / 6

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `.github/workflows/build-librespot.yml` (modify) | config | batch | Self (existing workflow) | exact |
| `Plugins/SpotOn/Helper.pm` (modify) | utility | request-response | Self (lines 28-32 ISWINDOWS block) | exact |
| `Plugins/SpotOn/Settings.pm` (modify) | controller | request-response | Self (line 59 helperMissing pattern) | exact |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (modify) | component | request-response | Self (lines 29-33 crashLoop warning) | exact |
| `Plugins/SpotOn/strings.txt` (modify) | config | N/A | Self (existing i18n entries) | exact |
| `README.md` (modify) | config | N/A | Self (line 36 platform list) | exact |
| `Plugins/SpotOn/Bin/darwin/.gitkeep` (new) | config | N/A | Any existing `Bin/*/.gitkeep` | exact |

## Pattern Assignments

### `.github/workflows/build-librespot.yml` (config, batch) -- MODIFY

**Analog:** Self -- the existing workflow IS the pattern. Three additions: `build-macos` job, `lipo` job, release job update.

**Matrix strategy pattern** (lines 20-39):
```yaml
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: x86_64-unknown-linux-musl
            bin_dir: x86_64-linux
            use_cross: true
          - target: aarch64-unknown-linux-musl
            bin_dir: aarch64-linux
            use_cross: true
          # ... more entries
          - target: x86_64-pc-windows-gnu
            bin_dir: x86_64-win64
            binary_ext: .exe
            use_cross: false
```

New `build-macos` job follows same matrix pattern but with `runs-on` from matrix (not hardcoded `ubuntu-latest`). Uses `os` matrix field instead of hardcoded runner.

**Build steps pattern** (lines 42-49 -- checkout + rust-toolchain):
```yaml
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}
```

macOS build uses identical checkout + rust-toolchain steps but with `cargo build` directly (no cross, no MinGW).

**Verify + upload pattern** (lines 91-107):
```yaml
      - name: Verify binary
        run: |
          EXT="${{ matrix.binary_ext }}"
          BINARY="Plugins/SpotOn/Bin/${{ matrix.bin_dir }}/${{ env.BINARY_NAME }}${EXT}"
          echo "=== Binary info ==="
          file "$BINARY"
          echo "=== Binary size ==="
          ls -la "$BINARY"
          echo "=== SHA256 checksum ==="
          sha256sum "$BINARY"

      - name: Upload binary artifact
        uses: actions/upload-artifact@v4
        with:
          name: spoton-${{ matrix.bin_dir }}
          path: Plugins/SpotOn/Bin/${{ matrix.bin_dir }}/spoton${{ matrix.binary_ext }}
          retention-days: 30
```

macOS individual builds use similar verify + upload, with artifact names `spoton-aarch64-darwin` and `spoton-x86_64-darwin`.

**Release job pattern** (lines 109-145):
```yaml
  release:
    name: Create GitHub Release
    runs-on: ubuntu-latest
    needs: build
    if: startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Download all binary artifacts
        uses: actions/download-artifact@v4
        with:
          path: release-artifacts

      - name: Create checksums
        run: |
          cd release-artifacts
          find . \( -name "spoton" -o -name "spoton.exe" \) -exec sha256sum {} \; > SHA256SUMS.txt
          cat SHA256SUMS.txt
```

Release job must change `needs: build` to `needs: [build, lipo]` to wait for macOS Universal Binary.

---

### `Plugins/SpotOn/Helper.pm` (utility, request-response) -- MODIFY

**Analog:** Self -- lines 28-32 (ISWINDOWS block)

**Platform-specific addFindBinPaths pattern** (lines 28-32):
```perl
    if ( main::ISWINDOWS ) {
        Slim::Utils::Misc::addFindBinPaths(
            catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'x86_64-win64')
        );
    }
```

New ISMAC block is structurally identical -- insert after line 32, before line 34 (`$prefs->setChange`):
```perl
    if ( main::ISMAC ) {
        Slim::Utils::Misc::addFindBinPaths(
            catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'darwin')
        );
    }
```

**Placement context** (lines 19-37):
```perl
sub init {
    # aarch64 can fall back to armhf binaries
    if ( !main::ISWINDOWS && !main::ISMAC
         && Slim::Utils::OSDetect::details()->{osArch} =~ /^aarch64/i ) {
        Slim::Utils::Misc::addFindBinPaths(
            catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'armhf-linux')
        );
    }

    if ( main::ISWINDOWS ) {
        Slim::Utils::Misc::addFindBinPaths(
            catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'x86_64-win64')
        );
    }

    # >>> INSERT ISMAC BLOCK HERE <<<

    $prefs->setChange( sub {
        $helper = $helperVersion = $helperCapabilities = undef;
    }, 'binary') if !main::SCANNER;
}
```

---

### `Plugins/SpotOn/Settings.pm` (controller, request-response) -- MODIFY

**Analog:** Self -- line 59 (helperMissing pattern) and line 217 (degradedMode pattern)

**Template variable passing pattern** (lines 56-61):
```perl
    my ($helperPath, $helperVersion) = Plugins::SpotOn::Helper->get();

    # Pass binary status to template
    $paramRef->{helperMissing} = string('PLUGIN_SPOTON_BINARY_MISSING') unless $helperPath;
    $paramRef->{binaryVersion} = $helperVersion || '';
    $paramRef->{binaryPath}    = $helperPath    || '';
```

New `isMac` template variable follows the same pattern as `degradedMode` (line 217):
```perl
    $paramRef->{degradedMode}   = _isDegradedMode();
```

Add near the helperMissing block:
```perl
    $paramRef->{isMac} = main::ISMAC ? 1 : 0;
```

---

### `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (component, request-response) -- MODIFY

**Analog:** Self -- lines 29-33 (crashLoop warning div pattern)

**Conditional warning div pattern** (lines 29-33):
```html
		[% IF discoveryByCrashLoop %]
		<div style="color: orange; margin-bottom:8px">
			[% 'PLUGIN_SPOTON_DISCOVERY_CRASH_LOOP_WARNING' | string %]
		</div>
		[% END %]
```

Also the degraded-mode warning pattern (lines 99-105):
```html
		[% IF degradedMode %]
		<div style="color: orange; margin-bottom:10px">
			[% 'PLUGIN_SPOTON_DEGRADED_MODE_WARNING' | string %]
			<a href="https://developer.spotify.com/dashboard" target="_blank">
				[% 'PLUGIN_SPOTON_REGISTER_APP_LINK' | string %]
			</a>
		</div>
		[% END %]
```

**Binary status section** (lines 76-82) -- this is where the Gatekeeper hint goes:
```html
	[% WRAPPER setting title="PLUGIN_SPOTON_BINARY_STATUS" desc="" %]
		[% IF binaryPath %]
			<p>[% binaryPath | html %] (v[% binaryVersion | html %])</p>
		[% ELSE %]
			<div style="color: red">[% 'PLUGIN_SPOTON_BINARY_MISSING' | string %]</div>
		[% END %]
	[% END %]
```

New Gatekeeper warning inserts inside the `[% ELSE %]` block when `isMac` is true:
```html
		[% ELSE %]
			<div style="color: red">[% 'PLUGIN_SPOTON_BINARY_MISSING' | string %]</div>
			[% IF isMac %]
			<div style="color: orange; margin-top:8px">
				[% 'PLUGIN_SPOTON_GATEKEEPER_HINT' | string %]
			</div>
			[% END %]
		[% END %]
```

---

### `Plugins/SpotOn/strings.txt` (config, N/A) -- MODIFY

**Analog:** Self -- all existing string entries

**i18n string block pattern** (lines 27-38 as example -- PLUGIN_SPOTON_BINARY_MISSING):
```
PLUGIN_SPOTON_BINARY_MISSING
	CS	Binární soubor SpotOn chybí. Prosím přeinstalujte plugin nebo zadejte cestu v nastavení.
	DA	SpotOn binærfilen mangler. Geninstallér venligst pluginet eller angiv stien i indstillingerne.
	DE	Das SpotOn-Binary fehlt. Bitte das Plugin neu installieren oder den Pfad in den Einstellungen angeben.
	EN	The SpotOn binary is missing. Please reinstall the plugin or specify the path in settings.
	ES	Falta el binario de SpotOn. Por favor reinstala el plugin o especifica la ruta en los ajustes.
	FR	Le binaire SpotOn est manquant. Veuillez réinstaller le plugin ou spécifier le chemin dans les paramètres.
	IT	Il binario SpotOn è mancante. Reinstalla il plugin o specifica il percorso nelle impostazioni.
	NL	Het SpotOn binair bestand ontbreekt. Herinstalleer de plugin of geef het pad op in de instellingen.
	NO	SpotOn-binærfilen mangler. Installer pluginen på nytt eller angi stien i innstillingene.
	PL	Brak pliku binarnego SpotOn. Zainstaluj ponownie wtyczkę lub podaj ścieżkę w ustawieniach.
	SV	SpotOn-binärfilen saknas. Installera om pluginet eller ange sökvägen i inställningarna.
```

New `PLUGIN_SPOTON_GATEKEEPER_HINT` follows identical format: key on first line, then tab-indented language code + tab + translation for all 11 languages (CS, DA, DE, EN, ES, FR, IT, NL, NO, PL, SV). Append after the last existing string block.

---

### `README.md` (config, N/A) -- MODIFY

**Analog:** Self -- line 36 (platform list)

**Current platform list** (line 36):
```markdown
- Supported platforms: x86_64 Linux, i386 Linux, aarch64 Linux (Pi 4+), armhf Linux (Pi 2/3), arm Linux, x86_64 Windows. macOS binaries not yet included.
```

Replace with updated list that adds macOS Universal Binary and xattr hint.

---

### `Plugins/SpotOn/Bin/darwin/.gitkeep` (new file)

**Analog:** Any existing `Bin/*/` directory. Each contains either a `spoton` binary or a `.gitkeep` placeholder.

```bash
$ ls Plugins/SpotOn/Bin/x86_64-linux/
spoton
```

The CI workflow `Prepare binary` step (lines 80-89) removes `.gitkeep` when placing the real binary:
```yaml
      - name: Prepare binary
        run: |
          # ...
          # Remove .gitkeep if present (binary replaces placeholder)
          rm -f "$DEST_DIR/.gitkeep"
```

New `Bin/darwin/.gitkeep` is an empty file that serves as a placeholder so git tracks the directory.

---

## Shared Patterns

### Platform Detection Constants
**Source:** LMS framework (`main::ISWINDOWS`, `main::ISMAC`)
**Apply to:** Helper.pm, Settings.pm
```perl
# LMS provides these as compile-time constants:
main::ISWINDOWS  # true on Windows
main::ISMAC      # true on macOS (darwin)
```
Already used in Helper.pm line 21 (`!main::ISMAC` guard) and line 28 (`main::ISWINDOWS`). Settings.pm should use `main::ISMAC` for the `isMac` template variable.

### Binary Directory Convention
**Source:** `Plugins/SpotOn/Bin/` directory structure + `build-librespot.yml` matrix `bin_dir` field
**Apply to:** Workflow, Helper.pm, .gitkeep
```
Bin/x86_64-linux/spoton       # Linux x86_64
Bin/aarch64-linux/spoton      # Linux ARM64
Bin/armhf-linux/spoton        # Linux ARMv7
Bin/arm-linux/spoton           # Linux ARM
Bin/i386-linux/spoton          # Linux i386
Bin/x86_64-win64/spoton.exe   # Windows
Bin/darwin/spoton              # macOS Universal Binary (NEW)
```
Note: `darwin/` breaks the `{arch}-{os}` naming convention intentionally because a Universal Binary covers both x86_64 and aarch64.

### Conditional Warning UI Pattern
**Source:** `basic.html` lines 29-33, 99-105
**Apply to:** basic.html (Gatekeeper hint)
```html
[% IF condition %]
<div style="color: orange; margin-bottom:8px">
    [% 'STRING_KEY' | string %]
</div>
[% END %]
```
Orange for warnings (vs. red for errors like binary-missing). No custom CSS classes -- inline styles are the established pattern in this template.

### i18n String Format
**Source:** `strings.txt` throughout
**Apply to:** strings.txt (Gatekeeper hint)
- Key on its own line (no indentation)
- Each translation: tab + 2-letter language code + tab + translation text
- All 11 languages: CS, DA, DE, EN, ES, FR, IT, NL, NO, PL, SV
- Blank line between string blocks

### GitHub Actions Artifact Naming
**Source:** `build-librespot.yml` line 105
**Apply to:** New macOS build and lipo jobs
```yaml
name: spoton-${{ matrix.bin_dir }}
# Examples: spoton-x86_64-linux, spoton-aarch64-linux, spoton-x86_64-win64
```
macOS intermediate artifacts: `spoton-aarch64-darwin`, `spoton-x86_64-darwin`
macOS final artifact: `spoton-darwin`

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| (none) | -- | -- | All files have exact analogs in the existing codebase |

Every file in this phase is either a modification of an existing file (with clear self-analog patterns) or a trivial new file (.gitkeep). No novel patterns are needed.

## Metadata

**Analog search scope:** `.github/workflows/`, `Plugins/SpotOn/`, `t/`, `README.md`
**Files scanned:** 8 (build-librespot.yml, Helper.pm, Settings.pm, basic.html, strings.txt, README.md, 06_binary_check.t, Bin/ directory)
**Pattern extraction date:** 2026-06-11
