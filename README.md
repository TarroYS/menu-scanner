# Menu Scan

A browser-based image processing tool to crop, perspective-correct, adjust, and join photos into clean PDFs and OCR-ready image archives. Built specifically for phone photos of restaurant menus, but fully applicable to any document scanning workflow. Everything runs entirely client-side in the browser—no photos are ever uploaded to a server.

**Live Demo:** [https://<username>.github.io/<repository>/](https://<username>.github.io/<repository>/)

---

## What It Does

The tool processes photos through five linear steps:

1. **Reorder:** Drop photos in and drag them into page order. Photo #1 becomes Page 1 of the PDF.
2. **Straighten:** Drag the four corner handles onto the corners of the menu sheet and click **Flatten** (or **Use as-is** if already aligned).
3. **Adjust:** Click **Auto-enhance all images**. Measures every page independently and adjusts tone, illumination, contrast, and sharpness.
4. **Join:** *(Optional)* Combine split menu pages. Drag the second half into place to form a composite page.
5. **Export:** Generate final production deliverables.

### Export Options

Each export button downloads exactly one file to avoid browser popup blocks:

* **Export all images (ZIP):** Contains all processed images (flattened, straightened, cropped, joined) flat at the root level.
* **Export PDF:** Outputs the finished pages in sequential order for standard document viewing.
* **Export OCR set (ZIP):** Contains un-joined single pages, a `README.txt`, and a `manifest.json` metadata file for automated parsers. *(Only visible after joining image pairs).*

---

## Keyboard & Mouse Shortcuts

| Action | Shortcut / Control |
| --- | --- |
| **Zoom in / out** | Mouse Scroll (centered on cursor) or `/` / `-` |
| **Pan view** | Hold `Space` + Drag (or Middle-Click + Drag) |
| **Fit to window** | `0` |
| **Jump to step (1–5)** | `1` – `5` |
| **Nudge corner handle** | Arrow keys (1 px) or `Shift` + Arrow keys (10 px) |
| **Peek original photo** | Hold `B` |
| **Reset slider** | Double-click slider handle |
| **Corner magnifier** | Hover cursor over any corner handle |

---

## Running It

### Hosted (Recommended)
This is a single static HTML file. Push `index.html` to GitHub and enable Pages:
> **Settings** → **Pages** → **Deploy from a branch** → `main` / `/ (root)`

### Offline / Local Execution
* **Windows:** Double-click `START-WINDOWS.bat`. Uses PowerShell's HTTP listener first (falling back to Python, then directly opening the file).
* **Mac:** Run `START-MAC.command` (ensure executable permissions via `chmod +x START-MAC.command`).
* **Direct File Open:** You can open `index.html` directly in modern browsers. All dependencies are fetched over HTTPS via CDN.

---

## File Structure

```text
├── index.html          # Entire web application (no build step or bundler needed)
├── START-WINDOWS.bat   # Windows local HTTP server launcher script
├── START-MAC.command   # macOS local HTTP server launcher script
├── HOW-TO-RUN.txt      # Plain-language running instructions for non-technical users
└── README.md           # Project documentation
