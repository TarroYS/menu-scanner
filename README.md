# menu-scanner
A tool to crop, perspective correct, adjust and join images together
Turns phone photos of restaurant menus into a clean PDF and OCR-ready images. Everything runs in the browser — no photo ever leaves the machine it was opened on.

Built for the Wonders Menu Change workflow, but there is nothing restaurant-specific in the code.

Live: https://<username>.github.io/<repo>/ ← replace once Pages is on

What it does

Five steps, left to right along the top:

		
1 Reorder	Drop photos in and drag them into page order	Number 1 becomes page 1 of the PDF
2 Straighten	Drag four corner handles onto the menu, hit Flatten	Or Use as-is if the photo is already square-on
3 Adjust	Hit Auto-enhance all images	Measures every page separately and sets it
4 Join	Only if the menu arrived in halves	Drag the second half into place; you stay here to do more pairs
5 Export	Three buttons, one download each	See below
The three exports
Export all images (ZIP) — images only: flattened, straightened, cropped and joined. Flat at the archive root, nothing else in the file.
Export PDF — the finished pages in order, for upload in CMA.
Export OCR set (ZIP) — the same pages un-joined, one file per sheet, plus a README.txt and manifest.json so a parser knows what it has. This button only appears once you have actually joined something — without a join it would be byte-identical to the first archive.

Each button downloads exactly one file, which is what stops Chrome blocking the second one.

Keyboard and mouse
Scroll                zoom in / out, centred on the cursor
Hold SPACE + drag     pan            (or middle-click drag)
+ / -                 zoom in / out
0                     fit to window
1-5                   jump to a step
Arrow keys            nudge a corner 1 px      (Shift = 10 px)
Hold B                peek at the original, before adjustments
Double-click slider   reset it to its default
Hover a corner        magnifier appears, so you can find the paper edge
Running it

Hosted (recommended). It is one static HTML file. Push it as index.html, turn on GitHub Pages (Settings → Pages → Deploy from a branch → main → /), and share the link. Works on tablets too.

Offline. Double-click START-WINDOWS.bat or START-MAC.command. They serve the folder locally and open a browser. Windows uses the PowerShell HTTP listener first (Python is not reliably installed, and python is often a Microsoft Store stub), falling back to Python, then to opening the file directly.

Opening index.html straight from disk usually works too — every dependency is CDN-hosted over HTTPS, so there is no local fetch for the browser to block.

Files
index.html            the entire tool — no build step, no bundler
START-WINDOWS.bat     offline launcher
START-MAC.command     offline launcher (needs one chmod +x, see HOW-TO-RUN)
HOW-TO-RUN.txt        plain-language guide, written for non-technical users
README.md             this file
How it works
Dependencies

All from CDN with mirrors; the app degrades rather than dies if one is blocked.

Library	Used for	If it fails
OpenCV.js 4.10	perspective warp, adaptive threshold, CLAHE	hand-rolled homography + bilinear warp takes over
jsPDF 2.5.1	PDF writing	built-in minimal PDF writer takes over
JSZip 3.10.1	ZIP writing	built-in store-only ZIP writer takes over
IBM Plex (Google Fonts)	type	falls back to the system stack

The three fallbacks are not decoration — a blocked CDN used to mean every upstream step worked and both export buttons were dead, which is the worst failure shape a tool like this can have. The ZIP writer is validated against Python's zipfile (CRC verification) and unzip -t; the PDF writer against a structural parser that walks every xref offset.

The order of operations matters

Geometry is corrected per photo, before joining. A single 4-point warp cannot rectify two photos taken from two camera poses, so stitching first and warping the composite is geometrically unsound. Correct each sheet to a rectangle, then join rectangles.

Joining costs one resample

Because each page keeps its source pixels (p.raw) and its resolved warp geometry (p.geom), the warp can be re-run at any output size. At join time both halves are re-warped from the originals to a matched scale, so the composite is a 1:1 drawImage with integer offsets — no interpolation at all.

stitch then warp        2 resamples, and geometrically wrong
warp then rescale       2 resamples
re-warp at matched      1 resample per photo   ← this

There is deliberately no free-form canvas board. Once a page is rectified it is an axis-aligned rectangle: rotation is already gone and scale is forced, so the only meaningful freedom is a 2-D translation. Rotate/scale handles would add an arbitrary extra resample and manual error, and nothing else.

Illumination flattening

The single biggest win for readability and OCR. Max-pool the image into cells to drop the ink and keep the paper, blur that into a smooth illumination field, then divide it out. Unlike a brightness/contrast curve it is spatial, so it fixes a shadowed fold that no global adjustment can touch; unlike adaptive thresholding it stays continuous-tone, so thin CJK strokes survive.

Measured on a synthetic page with a hard diagonal gradient: paper uniformity SD 40.2 → 3.1, ink/paper separation 117 → 199, and zero crushing of half-covered (anti-aliased) stroke pixels.

Auto-enhance measures rather than guesses

Four measurements, each mapped to one control:

measured	→	control
spread of the blurred illumination field	→	flatten
p2–p98 tonal span, taken after a notional flatten	→	contrast
median luma, after flatten	→	brightness (lift only)
Laplacian variance normalised by tonal span	→	sharpen

That last normalisation matters: without it a merely faded page (in focus, low contrast) is mistaken for a blurry one and gets sharpened when what it actually needs is contrast.

It runs on every straightened page, each measured on its own — two halves of one menu can be lit completely differently.

Quality and size

Two separate resolution caps, because the two outputs want opposite things.

	default	why
Image longest edge	4000 px	a phone photo is about 4000 px on its long side, so exports are essentially untouched — resolution is the one thing worth spending file size on for a parser
PDF longest edge	2600 px	the PDF is for reading; it does not need the full sensor
JPEG quality	92 %	visually identical to the original on menu text, roughly a tenth the size of PNG

Lossless PNG is available under Quality and size if a downstream step ever needs it. The export panel shows a live size estimate for each button before you commit — measured by encoding the preview proxy and scaling by pixel count, rather than by rendering a full 12 MP page every time the panel opens.

Configuration

Four flags at the top of the script. Each switches a whole feature, and the code behind every one is still present and still exercised by the checks.

js
const CONFIG = {
  enableSlicing:             false,  // adds a Slice step: draw crop regions per page
  requireSliceBeforeEnhance: false,  // gate the Adjust controls until a page is sliced
  autoDetect:                false,  // "find the edges for me" button
  autoSeam:                  false,  // "find the seam automatically" button
};

autoDetect and autoSeam are off because on real menu photos they were wrong often enough that correcting a bad guess cost more than doing it by hand. Both algorithms still work — the seam search finds a synthetic 64 px overlap at 65 px despite a 22 % exposure difference between halves.

Also easy to change: APP (name, version, tagline), ZIP_FOLDERS, PRESETS.scan (the default look every page starts on).

Known limits
Export blocks the UI. Full-resolution tone processing is synchronous, so the spinner freezes mid-page on large jobs. Moving processCanvas into a Web Worker with OffscreenCanvas is the clean fix; the pipeline is already shaped for it (pure function of canvas + adjustment object).
HEIC and TIFF are not decoded. The app names the file and tells you how to convert. On iPhone: Settings → Camera → Formats → Most Compatible.
The window is not resized on the Reorder step to avoid disturbing work in progress.
Nothing persists. Refresh the tab and the job is gone. There is no storage and no server by design.
Version history

Nothing before 1.0 was tagged; those labels are applied retrospectively. Descends from the earlier "Menu Copy Stand" (A1 / B1) line.

	
1.0	Reorder step with large sharp previews · OCR-specific tone pass dropped (it did not help) · split resolution caps · equal-weight button groups · auto-enhance covers every image · Join stays put for multiple pairs · images-ZIP is images-only · OCR export appears only when a join makes it distinct · first build carrying a version string
0.9	Export rebuilt as three single-download buttons
0.8	Reading profile as default view · Join shortcut from Adjust · auto seam search disabled
0.7	Renamed Menu Scan · light theme · advanced controls collapsed · dedicated Pages step · Use as-is · two-folder ZIP · JPEG defaults
0.6	Hand cursor on Space · real-time rotation · flatten feedback · page-strip grouping · export selection · drag reorder
0.5	Major reflow — geometry corrected per photo before joining · Fabric.js dropped, board replaced with offset-only join · slicing retired behind a flag
0.4	Board layout axis and free rearrangement · dual OCR/reading tone profiles
0.3	Illumination flattening · lossless quarter turns · JS edge-detection fallback · dependency-free ZIP/PDF writers · resolution-relative kernels · slice rubber-band coordinate fix
0.2	Launcher scripts · slicer decoupled from the tone pipeline
0.1	First build: 5 steps, OpenCV perspective warp, Fabric stitching board, PDF + ZIP export
