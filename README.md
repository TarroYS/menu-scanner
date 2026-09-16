# Menu Scan

A browser-based image processing tool to crop, perspective-correct, adjust, and join photos into clean PDFs and OCR-ready image archives. Built specifically for phone photos of restaurant menus (originally for the Wonders Menu Change workflow), but fully applicable to any document scanning workflow—there is nothing restaurant-specific in the code.

Everything runs entirely client-side in the browser—no photos are ever uploaded to a server, and no data ever leaves the machine it was opened on.

**Live Demo:** [https://tarroys.github.io/menu-scanner/](https://tarroys.github.io/menu-scanner/)

**Current version:** v2.2 — long-press the app name in the tool (or press `?`) to see the exact version and build date of whatever copy you're looking at.

---

## What It Does

The tool processes photos through six linear steps (left to right along the top navigation):

1. **Reorder:** Drop photos in and drag them into page order. Photo #1 becomes Page 1 of the PDF.
2. **Straighten:** Drag the four corner handles onto the corners of the menu sheet and click **Flatten** (or **Use as-is** if the photo is already square-on). Corners are detected automatically where that can be done reliably—see *Gated Auto-Detection* below.
3. **Adjust:** **Auto-enhance runs by itself** on arrival, measuring every page independently and setting tone, illumination, contrast, and sharpness. A **strength slider** (Extra gentle → Strong) is picked per page and can be overridden.
4. **Join:** *(Optional — only if the menu arrived in halves)* Pick two or more pages and drag them into place to form a composite. You stay on this step, so a four-photo job can be two joins in a row.
5. **Resize:** *(Optional — most jobs need nothing here)* Four preset buttons control output size and compression, with a live table showing the exact dimensions each page will come out at.
6. **Export:** Generate final production deliverables using three one-click buttons.

### Export Options

Each button downloads exactly one file, which prevents Chrome from blocking subsequent downloads:

* **Export all images (ZIP):** Contains all processed images (flattened, straightened, cropped, and joined) flat at the archive root. *(If there is only one page, it downloads as a single image—no point zipping one file.)*
* **Export PDF:** Outputs the finished pages in sequential order for standard document viewing (e.g., CMA uploads).
* **Export OCR set (ZIP):** Contains un-joined single pages, a `README.txt`, and a `manifest.json` metadata file so automated parsers know what they have. *(Note: This button only appears once you have actually joined something—without a join, it would be byte-identical to the first archive).*

**Why the OCR set is deliberately un-joined:** a parser downscales whatever you give it to a fixed size. A single sheet keeps its text roughly **45% taller** than the same sheet joined to another—the difference between small print being read and being guessed at.

---

## Keyboard & Mouse Shortcuts

Press `?` inside the tool for this list, plus the version and build date.

| Action | Shortcut / Control |
| --- | --- |
| **Main action for this step** | `Enter` — on Straighten it flattens, then jumps to the next photo |
| **Undo / redo** | `Ctrl` + `Z` (30 steps) / `Ctrl` + `Shift` + `Z` |
| **Previous / next photo** | `[` / `]` |
| **Use photo as-is** | `U` |
| **Rotate left / right** | `,` / `.` |
| **Auto-enhance all images** | `A` *(on Adjust)* |
| **Reset this image** | `R` *(on Adjust)* |
| **Difference view** | `D` *(on Join)* |
| **Zoom in / out** | Mouse Scroll (centered on cursor) or `+` / `-` |
| **Pan view** | Hold `Space` + Drag (or Middle-Click + Drag) |
| **Fit to window** | `0` |
| **Jump to step (1–6)** | `1` – `6` |
| **Nudge corner handle** | Arrow keys (1 px) or `Shift` + Arrow keys (10 px) |
| **Peek original photo** | Hold `B` (before adjustments) |
| **Reset slider** | Double-click slider handle |
| **Corner magnifier** | Hover cursor over any corner handle to find the paper edge |
| **Version / shortcut card** | `?` or long-press the app name |

Nothing fires while a text field has focus, and nothing is bound on the Export step—no download can happen by accident.

---

## How It Works

### Dependencies
All libraries load from CDNs with mirrors. The app is built to degrade gracefully rather than die if a dependency is blocked:

| Library | Used for | If it fails |
| --- | --- | --- |
| **OpenCV.js 4.10** (~10 MB) | Perspective warp, adaptive threshold, CLAHE *(Off by default)* | Span-interpolated bilinear warp takes over (~0.9s for a 10 MP page). |
| **jsPDF 2.5.1** | PDF writing | Built-in minimal PDF writer takes over. |
| **JSZip 3.10.1** | ZIP writing | Built-in store-only ZIP writer takes over. |
| **IBM Plex** (Google Fonts) | Typography | Falls back to the system font stack. |

**OpenCV Note:** OpenCV is not loaded at boot (`CONFIG.useOpenCV = false`). It is a ~10 MB WASM bundle that, in the shipped configuration, does exactly one job (perspective warp) which the pure-JS warp already handles in under a second. Turning it on buys a faster warp and specific enhance modes, but giving a dependency that size a fallback list can cause catastrophic renderer crashes if it retries. It gets **one** mirror and a short timeout—fall-through is right for a 90 KB library and disastrous for a 10 MB one.

### Order of Operations Matters
Geometry is corrected *per photo*, before joining. A single 4-point warp cannot rectify two photos taken from two camera poses, so stitching first and warping the composite is geometrically unsound. Correct each sheet to a rectangle, then join rectangles.

### Joining Costs One Resample
Because each page keeps its source pixels (`p.raw`) and its resolved warp geometry (`p.geom`), the warp can be re-run at any output size. At join time, every part is re-warped from the originals to a matched scale, so the composite is a 1:1 `drawImage` with integer offsets—no interpolation at all.

* Stitch then warp = 2 resamples, geometrically wrong
* Warp then rescale = 2 resamples
* **Re-warp at matched = 1 resample per photo** *(Used here)*

This is also why **N pages join in one step** rather than as repeated pairs: the first composite has `geom = null`, so a second pairwise join could only *rescale* it, costing an extra resample on the pages already merged.

The shared edge is forced to match exactly (`fitShared`). `renderFlatAt` scales the warp by a ratio, but `rotateCrop` rounds, leaving the joined edge two or three pixels out—a white sliver down the seam.

There is deliberately no free-form canvas board. Once a page is rectified, it is an axis-aligned rectangle. Adding rotation/scale handles would just add arbitrary extra resamples and manual error.

### Illumination Flattening
This is the single biggest win for readability and OCR: estimate the paper-white field, then divide it out. Unlike a global brightness/contrast curve it fixes shadowed folds; unlike adaptive thresholding it stays continuous-tone so thin CJK strokes survive.

The estimate is the subtle part. Taking the brightest pixel in each cell finds paper on a text page—but on a photograph the brightest pixel belongs to the *picture*, and dividing by it strips the picture's own light and shade. That was the "photos look bleached" problem, and it was a wrong model rather than a wrong constant.

Brightness alone can't separate the two: a photo highlight and a patch of shadowed paper sit at the same level. What separates them is **structure at a coarse scale**—paper stays flat across many cells even under a gradient, while a picture's cell maxima jump about. So the discriminator is the spread of cell maxima over a 5×5 window of cells, never anything measured inside a single cell, because at cell scale every smooth gradient looks flat. Untrusted cells are inpainted from trusted neighbours, and a smoothed picture map scales the correction down where it applies—so **one page can hold a photograph and a price column and get both right**.

### Auto-Enhance Measures Rather Than Guesses
It runs on every straightened page independently (two halves of one menu can be lit completely differently). A local-mean mask splits each page into **ink pixels and paper pixels**, and each control is driven by a measurement that can only come from the correct population:

| Measured | Maps to Control |
| --- | --- |
| Spread of the blurred illumination field | **Flatten** |
| Paper mean − ink mean (the real contrast gap) | **Contrast** |
| **Paper** mean, post-flatten | **Brightness** (lift only) |
| **Paper** mean − **paper** 5th percentile | **Shadows** |
| Fraction of **paper** at 250+ | **Highlights** (glare recovery) |
| Edge-transition width across ink/paper boundaries, normalized by the contrast gap | **Sharpen** |

The population split is the whole point. An earlier version used whole-page percentiles, which cannot tell text from shadow: it read the darkest 2% as "shadow" when on a menu the darkest 2% is the **text**, so it lightened the ink and fired hardest when the ink was darkest. Two other terms were dead or constant for related reasons.

A **strength** value (0–140%) is also chosen per page from how pictorial it is—a cover full of photographs gets a lighter touch than a page of prices, because contrast and sharpening both damage continuous tone.

### Gated Auto-Detection
Corner detection separates page from background by brightness, so it is accurate when they differ and hopeless when they do not—and that is measurable rather than a matter of hope. Sampling inside the detected quad and in a ring just outside it:

| Page-to-surface contrast | Corner error |
| --- | --- |
| 91–120 (dark table, wood, tilted, noisy) | 0.19 – 1.25% |
| 20–43 (mid-grey, light, white table) | 10 – 24% |

`quadConfidence` refuses below 60 rather than hand over a wrong quad, giving clean separation across ten test scenarios: seven accurate detections offered, three hopeless cases declined, no wrong quads. It runs once per page, silently, on arrival.

---

## Quality & Size

Size and compression have their own step (**5 — Resize**) with four preset buttons. Most jobs need nothing here; **Standard** is pre-selected.

| Preset | Setting | Result on a phone photo |
| --- | --- | --- |
| **Full size** | No resize, quality 96% | Unchanged — nothing resampled |
| **Standard** | 4000 px, quality 92% *(default)* | 3000×4000, ~98% of the pixels |
| **Half size** | 2400 px, quality 86% | 1800×2400, ~35% |
| **Small** | 1600 px, quality 82%, ≤400 KB each | 1200×1600 |

A **Fine tune** section holds the individual controls: resize by longest edge, percent, or exact width & height (fit inside or stretch to); quality; a per-image size limit; and lossless PNG. The sliders remain the single source of truth—a preset simply sets them, and which button is highlighted is worked out by reading them back, so the two can never disagree. Hand-tuning anything reads as *Custom*.

The **size limit** lowers quality only as far as it must. Past a floor of q0.55 it sheds resolution instead, because a 12 MP page will not fit in 200 KB at any quality worth having, and JPEG mush looks worse than a smaller sharp picture.

The PDF keeps its own separate cap:

| Output | Default Limit | Why |
| --- | --- | --- |
| **Image** (longest edge) | `4000 px` | A phone photo is ~4000 px. Exports are mostly untouched. Resolution is worth the file size for OCR parsers. |
| **PDF** (longest edge) | `2600 px` | The PDF is for reading and does not need full sensor resolution. |
| **JPEG Quality** | `92%` | Visually identical to original text, roughly 1/10th the size of PNG. |

---

## Configuration

Flags sit at the top of the script. Each switches a whole feature on/off. Code for all features remains present and checked:

```javascript
const CONFIG = {
  enableSlicing:             false,  // Adds a Slice step to draw crop regions per page
  requireSliceBeforeEnhance: false,  // Gates Adjust controls until a page is sliced
  autoDetect:                true,   // Automatic corner detection, confidence-gated
  detectMinContrast:         60,     // Refuse below this page-to-surface contrast
  autoSeam:                  false,  // "Find seam automatically" button
  useOpenCV:                 false,  // Loads the 10 MB WASM bundle
  playful:                   true,   // Kitchen-verb progress, fortune cookie, easter eggs
};
```

---

## Version History

Only 1.0 onward carry a version string in the file; the 0.x labels are applied retrospectively to build milestones. The line descends from an earlier "Menu Copy Stand" (A1 / B1) which is not in this repo.

### 2.x

| Version | Updates |
| --- | --- |
| **2.2** | Resize step reduced to **four preset buttons** (Full size / Standard / Half size / Small) with every slider moved into a collapsed **Fine tune** section. The sliders stay the single source of truth, so buttons and fine-tuning can never disagree; hand-tuning shows as *Custom*. |
| **2.1** | Resize and compress moved out of a collapsed disclosure into **their own step (5 of 6)**, with a live table showing exact per-page output dimensions. Defaults verified byte-identical to before: same pixels, same resample count, compress pass never runs when the limit is off. |
| **2.0** | **Auto edge detection back on, confidence-gated.** **Auto-enhance runs on arriving at Adjust** (undoable, resettable). **Strength became a 0–140% slider** with four named stops. **A single page exports as an image, not a zip.** **Resize** by longest edge / percent / exact dimensions, and **compress to a size limit** that prefers shedding resolution over destroying quality. |

### 1.x

| Version | Updates |
| --- | --- |
| **1.9** | **Fixed: joined pages exported with no adjustment at all** — `Join.commit` still read `a.adj.ocr` / `a.adj.read` from before the two tone profiles were collapsed, so every field came back undefined and the pipeline was skipped. **Join now takes N pages in one step.** Click a thumbnail in the strip to assign it to a slot. **Blank strip at the seam fixed** via `fitShared`. |
| **1.8** | **Extra gentle** strength added, and the real cause of remaining washout fixed: brightness had been deliberately left *out* of the strength multiplier and was the single largest cause, lifting a photo region 38 levels regardless of setting. Brightness now scales with everything else, and paper-derived terms are damped on pictorial pages. Cover brightness shift **+87 → +13** levels; blacks **+58 → +4**. |
| **1.7** | **Photographs stop being bleached.** `flattenLight` now estimates a **paper-only** field using coarse-scale structure, inpaints untrusted cells, and applies a smoothed **picture map**. Plus a **Gentle / Normal / Strong** strength control chosen automatically per page. |
| **1.6** | **Undo / redo**, 30 steps, `Ctrl`+`Z`. Snapshots hold metadata plus canvas *references*, never pixel copies. Join no longer pre-selects pages. Rotate buttons renamed **Rotate left / Rotate right**. "Where next" buttons moved into a **pinned footer** on every step. Join preview labels each half with the original photos inside it. |
| **1.5** | **Use as-is** goes straight to the next photo. Shortcut set expanded: context-sensitive **Enter**, `[` / `]`, `U`, `,` / `.`, `A`, `R`, `D`, and `?` for a card carrying the version, build date and the whole list. |
| **1.4** | **Auto-enhance rewritten to measure ink and paper as separate populations.** Whole-page percentiles had left three of six terms broken: `sha` lifted the ink, `bri` never fired at all, `shp` was a constant in disguise. Sharpness measured as edge-transition width, calibrated against actual blur. Adjust exits numbered **4 ·** and **5 ·**; Join seam section collapsed. |
| **1.3** | **OpenCV off by default.** Big optional dependencies now get **one** mirror, never a fallback list — a readiness bug had been loading three OpenCV builds and three WASM runtimes into one tab, crashing the renderer. Build date added to the header tooltip and manifest. |
| **1.2** | **No adjustment until you ask for one** — pages had been silently seeded with a preset altering every pixel by ~55 levels on arrival. Added **Before / after** toggle, **Reset this image**, **Reset all images**. |
| **1.1** | **OpenCV loader fixed** — `window.cv` is a Promise in MODULARIZE builds and the readiness test refused to await it, so it timed out on every load. Fallback warp rewritten with perspective-correct 16-pixel spans: **4.1× faster** (3598 ms → 885 ms on 10 MP), max geometric drift 0.0011 px. |
| **1.0** | First build carrying a version string. Pages → **Reorder** with large sharp previews. **OCR tone profile dropped** — greyscale and sharpening did not help; keeping the OCR set un-joined did. Split resolution caps. Equal-weight button groups. Auto-enhance covers every image. Join stays put for multiple pairs. Images ZIP is images-only. |

### 0.x — build milestones

| Version | Updates |
| --- | --- |
| **0.9** | Export rebuilt as three single-download buttons, each captioned with its purpose. |
| **0.8** | Reading profile as the default view · Join shortcut from Adjust · automatic seam search disabled. |
| **0.7** | Renamed **Menu Scan** · light minimal theme · advanced controls collapsed behind disclosures · dedicated Pages step · **Use as-is** · two-folder ZIP · JPEG defaults over PNG. |
| **0.6** | Hand cursor on Space · real-time rotation via source rotate + corner mapping · flatten feedback and progress · page-strip grouping · export selection · drag reorder. |
| **0.5** | **Major reflow.** Geometry corrected per photo *before* joining. Fabric.js dropped; the free-form board replaced with an offset-only join at **one resample per photo**. Slicing retired behind a flag. |
| **0.4** | Board layout axis and free rearrangement · dual OCR/reading tone profiles. |
| **0.3** | **Illumination flattening** (paper SD 40→3) · lossless quarter turns · JS edge-detection fallback · dependency-free ZIP and PDF writers · resolution-relative kernels · slice rubber-band coordinate fix. |
| **0.2** | Launcher scripts and hosting instructions · slicer decoupled from the tone pipeline. |
| **0.1** | First build: five steps, OpenCV perspective warp, Fabric stitching board, PDF and ZIP export. |
