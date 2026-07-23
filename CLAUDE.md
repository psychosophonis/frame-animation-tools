# frame-animation-tools

Two self-contained, client-side HTML tools for frame-based animation, split out of the BCM116 course site (`psychosophonis/bcm116Inter`) for reuse and independent development. Live at https://psychosophonis.github.io/frame-animation-tools/.

- `flipbook-generator.html` — folder of frames → A4-to-A6 imposed, print-ready flipbook PDF.
- `phenakistoscope-generator.html` — folder of frames, or a blank ring config → print-and-cut phenakistoscope disc/template.
- `index.html` — landing page linking the two.

No build step, no bundler, no `node_modules`. Each tool is one `.html` file: inline `<style>`, a giant bundled minified jsPDF (v2.5.1) in one `<script>`, and the actual app logic in a second `<script>` at the bottom. Edit files directly; there's nothing to compile.

## Finding the real code

Each generator's file is 90%+ bundled jsPDF noise. **Don't grep the whole file blind** — matches land inside the minified blob and are useless. Isolate the app-specific script first:

```bash
python3 - <<'EOF'
import re
html = open('phenakistoscope-generator.html', encoding='utf-8').read()
scripts = re.findall(r'<script(?:\s[^>]*)?>(.*?)</script>', html, re.S)
open('/tmp/app.js', 'w', encoding='utf-8').write(scripts[-1])  # last <script> = app code
EOF
grep -n "function foo" /tmp/app.js
```

The app script is the *last* `<script>` block (~60-65KB), always after the jsPDF bundle. Line numbers in the full `.html` file shift constantly as edits land — re-grep for anchor strings before editing, don't trust remembered line numbers across turns.

## No browser in this environment

Changes can't be visually verified here — no way to open the file and click through it. Compensate with:
1. **Syntax check** the extracted app script: `node --check /tmp/app.js`.
2. **Standalone math verification**: copy the relevant pure functions (`polar`, `wedgeBBox`, `chooseBlankGrouping`, `ringLayout`, etc.) into a throwaway `node -e '...'` snippet and assert against hand-computed expected values before trusting a geometry change. This caught real issues during development (confirming division-line page-ownership at piece seams, confirming crop-to-content math against the 16/14/13 default example, etc.) — worth doing for any change to the tiling/geometry code, not just trusting it compiles.
3. Ask the user to test-print or test-spin (e.g. via phenaspin.com) before treating a geometry change as done — several fixes here originated from real-world testing surfacing what math alone didn't catch (the original crop bug, the ring-band label overlap).

## phenakistoscope-generator.html architecture

**Two modes**, toggled by `#mode` select, sharing fieldsets 03 (full disc preview) and 04 (Disc: diameter/paper/margin):
- **Photo frames** (default): folder of images → one photo per wedge. Fieldsets 01 (Frames) + 02 (crop/placement calibration).
- **Blank ring template**: no photos — prints concentric guide rings for hand-drawing, each with its own division count and radial width. Fieldset 01 replaced by a dynamic ring-row list (`rings` array, `renderRingRows()`).

Both modes generate PDFs via the same underlying **wedge-piece tiling system**: a disc gets cut into `pieceCount` equal angular sectors sized to fit the chosen paper (`chooseGrouping` for photo mode / `chooseBlankGrouping` for blank mode — same idea, blank mode isn't constrained to divide evenly by any one ring's frame count since there's no per-image raster to keep intact). Each piece is printed on its own page with a registration crosshair at the vertex (disc centre) and seam ticks on its two straight edges, so pieces reassemble by aligning vertices, not by edge-matching artwork. `polar()`, `wedgeBBox()`, `toPage()`/`pagePt()` are the shared geometry primitives — same convention in both code paths.

**Key design decisions, in case they look reversible but aren't:**
- **Zoom can go below 100%** (`cropBoxFor`, min 20%). Below 100 the crop grows *past* the source image and `getPaddedSquare()` pads the excess transparent rather than cropping — this is what fixed full-framed (e.g. 16:9) source frames losing their sides. Don't reintroduce a `min="100"` on the zoom input.
- **Blank-mode output crops to `outerContentR`** (the outermost configured ring's own outer edge), not the nominal "Diameter" input (`ringLayout()`'s return value). This applies to the preview canvas, the PDF cut boundary, and the PNG export — all three must stay in sync if this logic changes. Rationale: printing the cut line out at the full nominal diameter when rings don't reach that far leaves dead paper and a meaningless outer circle. When rings are configured to fill the whole disc (`truncated: true`), `outerContentR === R` and this is a no-op.
- **No text is drawn inside a ring's radial band** in the PDF — only the arcs and division lines that define its own edges. Ring identification text lives in the page-margin piece label instead. This was a deliberate fix, not an oversight — a ring label placed mid-band risked sitting on top of a cut line for narrow rings.
- **8 divisions is the enforced minimum** per ring (`MIN_RING_DIVISIONS`) — below that a phenakistoscope doesn't read correctly when spun. No enforced maximum, by design (user request, not an oversight).
- **A flat circle needs both paper dimensions ≥ its own diameter** to print as one uncut piece — no margin or orientation choice changes that. This is why an 11" disc can't fit on one A4 sheet (A4's short edge is 210mm). The paper-size indicator (`maxSinglePieceDiameter`) and the "fits on: A3, A2..." suggestion in the post-generation plan text both exist because this fact is non-obvious until you hit it.

## Style conventions

- Units are mm throughout (jsPDF `unit:'mm'`), except font sizes (pt) and on-screen canvas pixels.
- IBM Plex Mono/Sans, CSS custom properties per tool (`--accent` differs: teal for flipbook, rust/brown for phenakistoscope) — kept deliberately distinct, not shared into a common stylesheet, since each tool is meant to stay a single portable file.
- No comments explaining *what* code does — only *why*, for non-obvious geometry/physics constraints (see examples throughout `phenakistoscope-generator.html`, e.g. the `wedgePath()`/`edgeSpan()` gap-math comments). Match that density; don't over-comment straightforward code.

## Deploying

`git push` to `main` — GitHub Pages serves directly from the repo root, no build/publish step. Changes are live within a minute or two of pushing.
