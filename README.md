# Menu Scan

A browser-based image processing tool to crop, perspective-correct, adjust, and join photos into clean PDFs and OCR-ready image archives. Built specifically for phone photos of restaurant menus (originally for the Wonders Menu Change workflow), but fully applicable to any document scanning workflow—there is nothing restaurant-specific in the code.

Everything runs entirely client-side in the browser—no photos are ever uploaded to a server, and no data ever leaves the machine it was opened on.

**Live Demo:** [https://tarroys.github.io/menu-scanner/](https://tarroys.github.io/menu-scanner/)

**Current version:** v2.6 — long-press the app name in the tool (or press `?`) to see the exact version and build date of whatever copy you're looking at.

---

## What It Does

The tool processes photos through six linear steps (left to right along the top navigation):

1. **Reorder:** Drop photos in and drag them into page order. Photo #1 becomes Page 1 of the PDF.
2. **Straighten:** Drag the four corner handles onto the corners of the menu sheet and click **Flatten** (or **Use as-is** if the photo is already square-on). Corners are detected automatically where that can be done reliably—see *Gated Auto-Detection* below.
3. **Adjust:** **Auto-enhance runs by itself** on arrival, measuring every page independently — and applies the result at the **lightest strength** (Extra gentle, 25%, or less where even that would bleach the whites or hurt a picture). **Enhancement strength** is the main control here: Off / Extra gentle / Gentle / Normal / Strong, with a slider for in between. **Hold `B`** (or press and hold the B button) to see the original. Set a page to **Off** — or `CONFIG.autoEnhance = false` for every job — when exports must be the photo itself.
4. **Join:** *(Optional — only if the menu arrived in halves)* **Click the pages in the strip at the bottom**: the first click is page A, the second is page B (click a chosen page again to take it out). Then drag them into place to form a composite. You stay on this step, so a four-photo job can be two joins in a row.
5. **Resize:** *(Optional — most jobs need nothing here)* Four preset buttons control output size and compression, with a live table showing the exact dimensions each page will come out at. The default is **Original quality**.
6. **Export:** Generate final production deliverables using three one-click buttons.

### Export Options

Each button downloads exactly one file, which prevents Chrome from blocking subsequent downloads:

* **Export all images (ZIP):** Contains all processed images (flattened, straightened, cropped, and joined) flat at the archive root. *(If there is only one page, it downloads as a single image—no point zipping one file.)*
* **Export PDF:** Outputs the finished pages in sequential order for standard document viewing (e.g., CMA uploads). Each page embeds **exactly the same image data** as the image export — full resolution by default, no re-encoding.
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
| **Peek original photo** | Hold `B`, or press and hold the **B** button / the chip on the page *(on Adjust)* |
| **Reset slider** | Double-click slider handle (the strength slider returns to what auto-enhance chose) |
| **Corner magnifier** | Hover cursor over any corner handle to find the paper edge |
| **Version / shortcut card** | `?` or long-press the app name |

Nothing fires while a text field or a dropdown has focus, and nothing is bound on the Export step—no download can happen by accident. Shortcuts do keep working after you have touched a slider.

---

## How It Works

### Dependencies
None are required. The PDF and ZIP writers are built in (since v2.5 — jsPDF and JSZip are gone), so no third-party script runs in the page unless OpenCV is switched on:

| Library | Used for | If it is off or fails |
| --- | --- | --- |
| **OpenCV.js 4.10** (~10 MB) | Edge detection, adaptive threshold, CLAHE *(Off by default)* | Built-in versions do the work. The **warp is always the built-in one** (Lanczos-3, ~1.3 s for a 9 MP page): OpenCV's was bicubic and half a pixel out, so it is no longer used for that. |
| **IBM Plex** (Google Fonts) | Typography | Falls back to the system font stack. |

### Security
* A **Content-Security-Policy** (meta tag — GitHub Pages cannot send headers) forbids the page from making any network connection (`connect-src 'none'`), loading remote images, posting forms or running `eval`. Client photos cannot leave the machine even if something malicious got into the page.
* Every file name is HTML-escaped before it is shown. (Before v2.5 a photo named `<img src=x onerror=…>.jpg` ran its script when dropped in.)
* Exported originals have their EXIF/XMP metadata (GPS position, device, owner) removed; the image data and ICC colour profile are untouched.

**OpenCV Note:** OpenCV is not loaded at boot (`CONFIG.useOpenCV = false`). It is a ~10 MB WASM bundle that the shipped configuration does not need. Turning it on buys OpenCV's edge detection and specific enhance modes, but giving a dependency that size a fallback list can cause catastrophic renderer crashes if it retries. It gets **one** mirror and a short timeout—fall-through is right for a 90 KB library and disastrous for a 10 MB one.

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

A **strength** value is also chosen per page, and since v2.6 it is deliberately the lightest one that does anything: **25% ("Extra gentle")** for a page of text, **15%** for a page with a picture or two, **5%** for one that is mostly picture — contrast and sharpening both damage continuous tone. Then one more check: if that strength would push more than 0.5% of the page into pure white or black that was not there before, it is stepped down until it does not. Staying close to the photograph means destroying nothing.

Why so little: measured on a straightened test page of text, 100% moved the page by **69 levels** on average and pushed **96%** of its pixels to pure white — a scanner's look, with the paper texture gone. At 25% the same page moves **21 levels** and **0.01%** clips. The full range is still on the slider; it is just no longer where a page starts. `CONFIG.autoStrength = 1` restores the old behaviour.

### Gated Auto-Detection
Corner detection separates page from background by brightness, so it is accurate when they differ and hopeless when they do not—and that is measurable rather than a matter of hope. Sampling inside the detected quad and in a ring just outside it:

| Page-to-surface contrast | Corner error |
| --- | --- |
| 91–120 (dark table, wood, tilted, noisy) | 0.19 – 1.25% |
| 20–43 (mid-grey, light, white table) | 10 – 24% |

`quadConfidence` refuses below 60 rather than hand over a wrong quad, giving clean separation across ten test scenarios: seven accurate detections offered, three hopeless cases declined, no wrong quads. It runs once per page, silently, on arrival.

---

## Quality & Size

Size and compression have their own step (**5 — Resize**) with four preset buttons. Most jobs need nothing here; **Original quality** is pre-selected.

| Preset | Setting | Result on a phone photo |
| --- | --- | --- |
| **Original quality** *(default)* | No resize, JPEG 100% (4:4:4), PNG stays PNG | **Resolution never touched.** An unchanged photo goes out as the **original file**. Changed pages are near-lossless (~59 dB) but about 2.5–3× the size of a 95% file. |
| **Original size, smaller files** | No resize, quality 95% | Same pixels, roughly a third of the size; colour detail halved (4:2:0) |
| **Half size** | 2400 px, quality 88% | 1800×2400, ~35% |
| **Small** | 1600 px, quality 82%, ≤400 KB each | 1200×1600 |

### What "original quality" means

1. **Untouched pages are the original file.** A page used as-is, with no tone change and no resize, is exported as the camera's own JPEG (or the original PNG): byte-identical image data, with only the EXIF/XMP metadata removed. The PDF embeds those same bytes. This is offered only when a strict reader will show the file exactly as the tool did — EXIF orientation 1, 3-channel — because many OCR pipelines ignore the orientation tag.
2. **Changed pages are written at JPEG 100.** Chrome subsamples colour 4:2:0 at **every** quality below 1.0 (measured: 92%, 95% and 99% all write 4:2:0), so 100 is the only JPEG setting that keeps full colour resolution — it matters for red prices and thin coloured text. Pages from lossless sources (screenshots) stay PNG.
3. **One resample per page, with the best practical kernel.** Straightening is the one unavoidable resample, so it uses **Lanczos-3** (six taps each way), samples pixel centres, and folds any deskew rotation — and, in a join, the exact edge fit — into the same single mapping. Measured against ground truth with only the kernel changed: bilinear 33.8 dB (v2.4), Catmull-Rom 35.4 dB (v2.5), Lanczos-3 **36.1 dB** (v2.6); Lanczos-4 adds nothing further. An identity map still copies pixels exactly.
4. **The export matches the preview.** Sharpening and illumination-flattening are computed in page terms, so what you approve on screen is what lands in the file.

**Nothing outside this step ever changes an image's resolution.** Changing how many pixels a file has is a decision, so it is made here or not at all — a job that never opens step 5 exports at the size it came in.

A **Fine tune** section holds the individual controls: resize by longest edge, percent, or exact width & height (fit inside or stretch to); quality; a per-image size limit; and lossless PNG. The sliders remain the single source of truth—a preset simply sets them, and which button is highlighted is worked out by reading them back, so the two can never disagree. Hand-tuning anything reads as *Custom*.

### The Warp Samples the Quad, Not the Frame

`warpJS` pre-scales the source before sampling, so `getImageData` never allocates a full 140 MB frame for a small output. That ratio has to be measured against the **quad** — the warp only ever samples inside it — not against the whole picture.

Measuring against the frame was badly wrong whenever the quad was a small part of it. A menu letterboxed inside a phone screenshot occupies about a quarter of the frame, so the old formula discarded 42% of the linear resolution *before* sampling, then upscaled the remainder to fill the output:

| | Detail retained | PSNR vs a straight crop |
| --- | --- | --- |
| Measured against the frame | 8% | 21.6 dB |
| **Measured against the quad** | **100%** | **99 dB (pixel-identical)** |

### The Resample Dead-Band

Any resample runs a reconstruction filter over every pixel, and that filter attenuates high frequencies whether the ratio is 0.3 or 0.99. At 0.99 you pay the entire softening cost for almost none of the size saving.

Measured on a 4032 px phone photo capped at 4000 px — a 1.6% pixel reduction:

| Filter | Fine detail retained |
| --- | --- |
| Lanczos | 82% |
| Bicubic | 70% |
| Bilinear | 49% |

Browsers sit somewhere between those. `scaleCanvas` therefore skips any trim smaller than 3% (`RESAMPLE_DEADBAND = 0.97`), and `Size.outFor` mirrors the same rule so the preview table can't promise a resize that doesn't happen.

The **size limit** lowers quality only as far as it must. Past a floor of q0.55 it sheds resolution instead, because a 12 MP page will not fit in 200 KB at any quality worth having, and JPEG mush looks worse than a smaller sharp picture.

The PDF keeps its own separate cap:

| Output | Default Limit | Why |
| --- | --- | --- |
| **Image** (longest edge) | *no cap by default* | Resolution is only changed from the Resize step. The 4200 px cap is available there, set above typical phone resolution. |
| **PDF** (longest edge) | *full resolution by default* | A cap (4200 – 1600 px) is available on the Export step for when the PDF is too heavy to upload. |
| **JPEG Quality** | `100%` | The only setting at which the browser keeps full colour resolution. |

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
  autoEnhance:               true,   // Auto-enhance on arriving at Adjust (changes pixels)
  autoStrength:              0.25,   // How hard it pushes a page of text: 0.25 = Extra gentle, 1 = as v2.5
  playful:                   true,   // Kitchen-verb progress, fortune cookie, easter eggs
};
```

---

## Version History

Only 1.0 onward carry a version string in the file; the 0.x labels are applied retrospectively to build milestones. The line descends from an earlier "Menu Copy Stand" (A1 / B1) which is not in this repo.

### 2.x

| Version | Updates |
| --- | --- |
| **2.6** | **Auto-enhance at the lightest strength, and a sharper straighten.** Auto-enhance still measures every page on arrival but applies it at **Extra gentle (25%)** instead of up to 100% — 15% or 5% for pages that are partly or mostly picture, and stepped down further on any page whose whites would start to clip. On a straightened test page of text that is 69 → 21 levels of average change, and 96% → 0.01% of pixels pushed to pure white. **Strength is now the main control on Adjust:** named buttons, a large slider, a plain-words line on what each level costs, "use on every page" — and it works from an untouched page (it did nothing after a Reset). **Hold B** is shown as a key you can also press and hold with the mouse, with a chip on the page that turns amber while the original is showing. **Join:** the strip is highlighted as the page picker; pane, strip and empty stage all say what the next click does, and chosen pages carry a large A / B. **Straighten uses Lanczos-3:** 35.4 → 36.1 dB against ground truth, deskewed pages 35.6 → 36.5 dB. **A deskewed page in a join no longer gets a second resample** when its edge lands a pixel off (34.1 → 36.4 dB). **Fixed:** B, A and R did nothing while a slider had focus; 5% strength displayed as "off"; the compare button lit up with nothing to compare at strength 0; choosing join pages from the dropdowns did not update the strip; OpenCV, when enabled, replaced the warp with a lower-quality one. The Resize step now says in plain words that it compresses and/or shrinks, and never crops. |
| **2.5** | **Original-quality export, measured end to end.** Default is now JPEG 100 (Chrome writes 4:2:0 colour at every quality below 1.0), untouched photos go out as the **original file** (metadata stripped, image data byte-identical), PNG sources stay PNG, and the **PDF embeds the same bytes at full resolution** (was re-encoded and capped at 2600 px: 31.4 → 35.4 dB against ground truth). **Bicubic warp** with pixel-centre sampling (+1.5 dB), **deskew folded into the warp** — one resample instead of two (+3.2 dB). **Export now matches the preview:** sharpening radius and the illumination field are computed in page terms (the export had been softer than the preview and 1.5–2 levels darker). **Security:** file names are escaped (a crafted name executed script), jsPDF/JSZip (loaded from jsDelivr/unpkg with no integrity hashes) removed, Content-Security-Policy blocks all network connections. **Fixed:** the Export step's "Lossless PNG" box did nothing (duplicate element id); the built-in PDF ignored the page-format setting; transparent PNGs exported with a black background; an encoder refusal crashed the export silently; deleting from the strip could not be undone; pages straightened after the first visit to Adjust were never auto-enhanced; a quarter turn re-warped an as-is page; Ctrl+Z in a text box undid the app; ZIP names are UTF-8-flagged and keep CJK characters; undo history has a memory budget. |
| **2.4** | **Fixed: cropping a letterboxed screenshot destroyed its detail.** `warpJS` sized its pre-scale against the source *frame* rather than the *quad* it actually samples, so a menu occupying a quarter of a screenshot lost 42% of its linear resolution before the warp, then got upscaled back — 8% of the fine detail and 21.6 dB. Now pixel-identical for a 1:1 crop. **Resolution is no longer changed by default**: the Resize step is the only place it happens, and its default is *Original size*. |
| **2.3** | **Fixed: the default resolution cap was softening every export.** Capping a 4032 px photo at 4000 px is a 0.992× resample — 1.6% fewer pixels for 18–51% of the fine detail, depending on the browser's filter. Added a **resample dead-band**: any trim under 3% is skipped entirely. Cap raised to **4200 px** (above phone resolution) and quality to **95%**, which matches what cameras write and makes the re-encode near-idempotent. Measured end-to-end on a 12 MP photo: **61 dB PSNR, no resample, 100% of the original file size** — up from a resampled 2.43 MB at 82% detail. |
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
