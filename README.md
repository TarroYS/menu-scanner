# Menu Scan

A browser-based image processing tool to crop, perspective-correct, adjust, and join photos into clean PDFs and OCR-ready image archives. Built specifically for phone photos of restaurant menus (originally for the Wonders Menu Change workflow), but fully applicable to any document scanning workflow—there is nothing restaurant-specific in the code.

Everything runs entirely client-side in the browser—no photos are ever uploaded to a server, and no data ever leaves the machine it was opened on.

**Live Demo:** [https://tarroys.github.io/menu-scanner/](https://tarroys.github.io/menu-scanner/) *(Replace once Pages is on)*

---

## What It Does

The tool processes photos through five linear steps (left to right along the top navigation):

1. **Reorder:** Drop photos in and drag them into page order. Photo #1 becomes Page 1 of the PDF.
2. **Straighten:** Drag the four corner handles onto the corners of the menu sheet and click **Flatten** (or **Use as-is** if the photo is already square-on).
3. **Adjust:** Click **Auto-enhance all images**. This measures every page independently and sets the optimal tone, illumination, contrast, and sharpness.
4. **Join:** *(Optional - only if the menu arrived in halves)* Drag the second half into place to form a composite page. You can stay on this step to join multiple pairs.
5. **Export:** Generate final production deliverables using three one-click buttons. 

### Export Options

Each button downloads exactly one file, which prevents Chrome from blocking subsequent downloads:

* **Export all images (ZIP):** Contains all processed images (flattened, straightened, cropped, and joined) flat at the archive root.
* **Export PDF:** Outputs the finished pages in sequential order for standard document viewing (e.g., CMA uploads).
* **Export OCR set (ZIP):** Contains un-joined single pages, a `README.txt`, and a `manifest.json` metadata file so automated parsers know what they have. *(Note: This button only appears once you have actually joined something—without a join, it would be byte-identical to the first archive).*

---

## Keyboard & Mouse Shortcuts

| Action | Shortcut / Control |
| --- | --- |
| **Zoom in / out** | Mouse Scroll (centered on cursor) or `+` / `-` |
| **Pan view** | Hold `Space` + Drag (or Middle-Click + Drag) |
| **Fit to window** | `0` |
| **Jump to step (1–5)** | `1` – `5` |
| **Nudge corner handle** | Arrow keys (1 px) or `Shift` + Arrow keys (10 px) |
| **Peek original photo** | Hold `B` (before adjustments) |
| **Reset slider** | Double-click slider handle |
| **Corner magnifier** | Hover cursor over any corner handle to find the paper edge |

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

**OpenCV Note:** OpenCV is not loaded at boot (`CONFIG.useOpenCV = false`). It is a ~10 MB WASM bundle that, in the shipped configuration, does exactly one job (perspective warp) which the pure-JS warp already handles in under a second. Turning it on buys a faster warp and specific enhance modes, but giving a dependency that size a fallback list can cause catastrophic renderer crashes if it retries. 

### Order of Operations Matters
Geometry is corrected *per photo*, before joining. A single 4-point warp cannot rectify two photos taken from two camera poses, so stitching first and warping the composite is geometrically unsound. Correct each sheet to a rectangle, then join rectangles.

### Joining Costs One Resample
Because each page keeps its source pixels (`p.raw`) and its resolved warp geometry (`p.geom`), the warp can be re-run at any output size. At join time, both halves are re-warped from the originals to a matched scale, so the composite is a 1:1 `drawImage` with integer offsets—no interpolation at all.

* Stitch then warp = 2 resamples, geometrically wrong
* Warp then rescale = 2 resamples
* **Re-warp at matched = 1 resample per photo** *(Used here)*

There is deliberately no free-form canvas board. Once a page is rectified, it is an axis-aligned rectangle. Add rotation/scale handles would just add arbitrary extra resamples and manual error.

### Illumination Flattening
This is the single biggest win for readability and OCR. The app max-pools the image into cells to drop the ink and keep the paper, blurs that into a smooth illumination field, and divides it out. Unlike a global brightness/contrast curve, it fixes shadowed folds; unlike adaptive thresholding, it stays continuous-tone so thin CJK strokes survive.

### Auto-Enhance Measures Rather Than Guesses
It runs on every straightened page independently (two halves of one menu can be lit completely differently). Four measurements map to four controls:

| Measured | Maps to Control |
| --- | --- |
| Spread of the blurred illumination field | **Flatten** |
| p2–p98 tonal span (taken after a notional flatten) | **Contrast** |
| Median luma (after flatten) | **Brightness** (lift only) |
| Laplacian variance normalized by tonal span | **Sharpen** |

*(Normalizing sharpen by tonal span ensures a faded page isn't mistaken for a blurry one).*

---

## Quality & Size

The tool uses two separate resolution caps because the outputs require different optimizations. The export panel shows a live file size estimate before committing.

| Output | Default Limit | Why |
| --- | --- | --- |
| **Image** (longest edge) | `4000 px` | A phone photo is ~4000 px. Exports are mostly untouched. Resolution is worth the file size for OCR parsers. |
| **PDF** (longest edge) | `2600 px` | The PDF is for reading and does not need full sensor resolution. |
| **JPEG Quality** | `92%` | Visually identical to original text, roughly 1/10th the size of PNG. |

*Lossless PNG is available under the advanced options if a downstream step requires it.*

---

## Configuration

Four flags sit at the top of the script. Each switches a whole feature on/off. Code for all features remains present and checked:

```javascript
const CONFIG = {
  enableSlicing:             false,  // Adds a Slice step to draw crop regions per page
  requireSliceBeforeEnhance: false,  // Gates Adjust controls until a page is sliced
  autoDetect:                false,  // "Find edges for me" button (often less accurate than manual)
  autoSeam:                  false,  // "Find seam automatically" button
  useOpenCV:                 false,  // Loads the 10 MB WASM bundle
};
