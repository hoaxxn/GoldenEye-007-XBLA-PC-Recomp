# macOS-Port: Arbeitsnotizen

Stand 2026-10-19, nach PR [CaiusCodes/GoldenEye-007-XBLA-PC-Recomp#7](https://github.com/CaiusCodes/GoldenEye-007-XBLA-PC-Recomp/pull/7).

## Bauen und testen

- Build: `cmake --preset mac-arm64-release` und `cmake --build --preset mac-arm64-release`.
  Das Spiel liegt dann in `out/build/mac-arm64-release/`, `assets` ist ein Symlink auf `../../../game`.
- **Falle:** Wenn sich nur Code in `rexglue-sdk/` ändert, werden die neu gebauten Bibliotheken
  (`librexgpu-xenos.dylib`, `librexruntime.dylib` in `rexglue-sdk/out/mac-arm64/`) **nicht** neben das Spiel
  kopiert. Das Kopieren ist ein POST_BUILD-Schritt des Targets `ge` und läuft nur, wenn `ge` neu gelinkt wird.
  Sonst läuft das Spiel stillschweigend mit der alten Bibliothek. Vor jedem Test mit `cmp` prüfen oder von Hand kopieren:
  ```bash
  cp rexglue-sdk/out/mac-arm64/librexgpu-xenos.dylib rexglue-sdk/out/mac-arm64/librexruntime.dylib out/build/mac-arm64-release/
  ```
- SDK-Änderungen gehören in `patches/rexglue.patch`. Neu erzeugen aus dem Submodul (neue Dateien mit `add -N` einbeziehen):
  ```bash
  cd rexglue-sdk && U=$(git ls-files --others --exclude-standard) && git add -N $U && git diff --no-color > ../patches/rexglue.patch; git reset -q $U
  ```
  Probe: Submodul mit `git checkout -- . && git clean -fd` zurücksetzen, dann `cmake --preset ...`. Das wendet den Patch neu an.
- Testflags auf der Kommandozeile landen in `ge.toml`, weil SaveConfig alle vom Standard abweichenden Werte schreibt.
  Vor Testläufen die Datei sichern und danach zurückspielen.
- Mit `--ge_debug_input_file=<datei>` lässt sich das Spiel steuern. Die Datei nimmt Wörter wie `START A UP LEFT WAIT`, `CLICK=x,y` und `AIM=n` an.
  Für Bildvergleiche diente eine temporäre Aufnahme über `Presenter::CaptureGuestOutput` (`src/ge_debug_capture.cpp`, wieder entfernt).
- Schneller Test für den Linienfehler: Im Hauptmenü die Maus über das weiße Bond-Foto bewegen.
  Ist der Fehler da, zeigt sich um das Zielkreuz ein Rechteck mit Diagonale.

## Linienfehler (dunkle, gestrichelte Linien an Dreieckskanten, nur Mac)

**Ursache:** Die Pixel auf den Linien sampeln die kleinste Mip-Stufe (1×1, die Durchschnittsfarbe der Textur).
Die Textur-Ableitungen in den 2×2-Quads an Dreieckskanten sind riesig. Das gilt für Shader-`dfdx`/`dfdy` und
für das Hardware-LOD. Grund: Apple-GPUs liefern über MoltenVK in nicht abgedeckten Quad-Spuren (Helper)
unbrauchbare Interpolationswerte. Das Menü rendert mit 4× MSAA.

**Fix (im Patch):** `Features::quad_robust_texture_derivatives`, nur `__APPLE__`.
Voraussetzungen: SPIR-V ≥ 1.3, Demote-to-Helper und Quad-Subgroup-Operationen im Fragment-Shader.
`TextureCoordinateDerivative(s)` in `spirv_translator_fetch.cpp` arbeitet so:
- Die Ableitung kommt nur aus Quad-Paaren ohne Helper.
- Ist das eigene Paar unbrauchbar, wird die andere Zeile bzw. Spalte genommen.
- Sind beide Paare brauchbar, gewinnt die kleinere Ableitung. Ist keines brauchbar, ist sie 0.
- Zusätzlich wird eine Achse verworfen, die mehr als 16× länger ist als die andere.

Damit ist das Rechteck um das Zielkreuz weg, und die Linien sind stark reduziert.

**Rest:** Feine Punkte an manchen Kanten. Ein Restpixel hat genau den Mittelwert aus korrekter Farbe und
1×1-Mip-Farbe. Unter MSAA wird ein Kantenpixel von beiden Dreiecken schattiert, und bei einer der beiden
Schattierungen ist schon der eigene interpolierte Wert unbrauchbar, obwohl die Spur nicht als Helper gilt.

**Ausgeschlossen (bringt nichts oder nicht gegen den Rest):**
- Polygon-Modus LINE/POINT: Es entstehen gar keine solchen Pipelines.
- Fast Math (`MVK_CONFIG_FAST_MATH_ENABLED=0`), Positions-Invarianz.
- `--no-native_2x_msaa`, `early_fragment_tests` aus, Centroid-Interpolation aus.
- NaN/Inf: Die Werte sind endlich, nur viel zu groß.
- Implizites Hardware-LOD statt expliziter Gradienten: Damit ist es gleich kaputt.
- PrimitiveID-Vergleich im Quad: Fremde Dreiecke im Quad sind nicht die Ursache.
- Sample-Rate-Shading (`minSampleShading = 1`): Die Restlinien bleiben.
- Gradienten = 0 beseitigt alles, wäre aber ohne Mip-Mapping, also kein Fix.

**Nützliche Werkzeuge:**
- `MVK_CONFIG_SHADER_DUMP_DIR=<ordner>` schreibt den erzeugten Metal-Code (`.metal`) und SPIR-V aller Shader.
- Shader-Diagnose per Umgebungsvariable im Übersetzer einbauen, etwa so: Eine Kennfarbe statt des Texturergebnisses
  ausgeben, wenn die Ableitung groß ist. Diese Methode hat Helper- und Nicht-Helper-Fälle getrennt.

**Nächster sinnvoller Schritt:** Metal-GPU-Capture in Xcode mit dem Shader-Debugger an einem Restpixel.
So lassen sich die Interpolationswerte pro Quad-Spur ansehen. Möglicherweise ist es ein Fehler in MoltenVK oder
im Apple-Treiber, der sich isoliert nachbauen und melden lässt.

## Weitere bekannte Punkte

- Vollbild kann schwarz bleiben. Deshalb startet das Spiel auf dem Mac standardmäßig im Fenster (Patch in `window.cpp`).
- Performance: Bei `resolution_scale = 2` (Standard) liefen etwa 15 FPS, bei `1` etwa 60 FPS.
  Mit ganzzahligen Faktoren ist 1080p nicht möglich.
- Der Windows-Build ist mit den Mac-Änderungen ungetestet.
