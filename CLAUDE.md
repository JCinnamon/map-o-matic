# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Map-O-Matic is a browser-only cartography studio (Map Maker, Map Composer, Field Mapper, Atlas, plus a Home/project screen). It has no server, accounts or backend; everything is stored client-side. It is deployed at jonathancinnamon.com/map-o-matic. `README.md` is the user-facing manual and describes every feature in detail, so check it to see how a feature is meant to behave.

## Development

There is no build step, package manager, linter or test suite. **The whole application is `index.html`** (~23k lines, ~1.5 MB): inline CSS, inline JS and pinned CDN libraries.

- Run locally: serve the folder over HTTP (e.g. `python3 -m http.server`) and open `index.html`. Some features (IndexedDB persistence, File System Access folder sync, map capture) behave differently or fail under `file://`.
- Folder sync uses the File System Access API, so it works only in Chrome or Edge.
- Only one tab may run the app at a time (a second tab shows a takeover notice). Share links, shared atlases and phone forms are exceptions.
- Commits typically land directly on `main`, with messages like "Update index.html".

## Navigating index.html

The file is too large to read whole. Use grep to find the section banners, then read the range you need:

```
grep -n -E '^/\* =+' index.html        # major modules
grep -n -E '^/\* -{3,}' index.html      # subsections
```

Rough layout:
- Lines ~1–2460: `<head>`, the theme bootstrap (`mom-theme` in localStorage, `data-theme` on `<html>`), all CSS (`:root` tokens such as `--navy`, `--teal`; there is a dark-mode variant) and the HTML markup for every module and modal.
- ~2438–2454: CDN `<script>` tags. Each library is pinned to an exact version with an SRI `integrity` hash. A following script checks that each global (`L`, `fabric`, `turf`, `d3`, `chroma`, `Papa`, `JSZip`, `ss`) loaded and shows an error otherwise. If you add or upgrade a library, update the hash and that check list. Rarely used libraries are lazy-loaded through helpers under "loading helpers for libraries used only occasionally".
- `window.MAPOMATIC_PUBLISH`: config for GitHub publishing (OAuth client ID and gatekeeper URL, empty by default).
- ~2487 to end: one big IIFE (`"use strict"`) holding the whole app. In order, its banners are: shared utils → MAP MAKER (basemaps, globe, swipe, CRS, symbology/classification, hexbin/heatmap/cartograms, layer list, attribute table, spatial tools, editing, georeferencing) → FIELD MAPPER → ATLAS VIEWER / ATLAS → PROJECTED MAP VIEWER → BATCH GEOCODER → legend/send-to-layout → PROJECTED PREVIEW (d3-geo) → MAP LAYOUT (Fabric.js composer) → SKETCH tools → ONLINE DATA SOURCES → FOLDER SYNC → METHODS LOG → PROJECT MANAGER (IndexedDB) → export/embed/share builders.
- Comments are tagged with feature/version labels (`v67`, `F10`, `U2`, …). The current release string is `MOM_VERSION` (search for `const MOM_VERSION`) and appears in the ⚙ menu and Help. Bump it when you ship a release.

## Key architectural patterns

**Exported HTML is generated from app code via `Function.prototype.toString()`.** Interactive web map exports, share-link viewers, projected/globe viewers and atlas exports are standalone HTML documents built as strings inside `index.html` (search `buildEmbedHtml`, `buildAtlasHtml`, `projViewer`, `atlasViewer`). They embed app functions by serializing their source:
- The block between `/* ===== self-contained runtime ... */` and `/* ===== end of the self-contained runtime ===== */` (terrain, basemap stack, OHM year, etc.) is joined into `MOM_STACK_RT` / `MOM_GLOBE_RT`.
- `atlasViewer`, `projViewer`, `glbAddLayer`, `glbColor`, `momCanvasStack`, `momTiers`, `momPattern`, and others are also `.toString()`-ed.

These functions **must stay self-contained**. They may not reference app globals, module-scope helpers or closures, and they must not use syntax that breaks once embedded. Inside these template strings, closing script tags must be written as `<\/script>`, and JSON data is escaped (`</script`, `<!--`, U+2028/2029) before it is inlined.

**Duplicated `<script src>` tags deeper in the file** (e.g. Leaflet, MapLibre or D3 again around lines 13.9k and 22.1k–22.4k) belong to those generated documents, not the app. Keep their versions and integrity hashes in sync with the head tags.

**Persistence.** IndexedDB `cartoStudioDB` (version 2) has stores `projects`, `datasets` (the shared Data Library) and `settings` (the sync-folder handle). Projects embed their datasets. Since v94, `projectsCache` holds summaries only (`projStub`: name, dates, thumbnail, methodsLog, layer count, dataset ids); read the whole project with `projectFull(id)` and store with `projCachePut(rec)`. Datasets keep `geojson` in memory only while the open project uses them (`datasetsLoad`, `datasetsTrim`, `datasetFull`). Never `idbPut` a summary or a dataset without `geojson`; use `idbUpdate` to change a stored record in place. In a saved record `canvasJSON` is the same object as the open layout in `layouts` (stored once), and files written with `projFileRec` leave it empty. The legacy "cartoStudio" name still appears in DB and storage keys and in `.cartostudio.json` sync files, which are still read and renamed to `.mapomatic.json`. Don't rename these keys without a migration. The Field Mapper phone side uses its own `momCollectDB`. UI preferences go in localStorage under `mom-*` keys, always wrapped in try/catch because storage throws inside sandboxed frames.

**Hash-routed modes.** The same page also runs as a read-only viewer or phone app, depending on `location.hash`: `#c=` opens a Field Mapper phone form, `#a=` opens a shared atlas, and share links carry compressed payloads made by `encodeShare`/`decodeShare`. Treat decoded payloads and imported project files as untrusted. They are sanitized (e.g. `sanitizeCollectForm`) and HTML-escaped with `esc()` before rendering, so keep that invariant.

**UI conventions.** Dialogs are `.modal-overlay` elements opened with `openModal(id)` and closed with `closeModal(id)`. `enhanceModal` makes them draggable and handles focus. A MutationObserver auto-links `.field` labels to their controls for accessibility. User feedback goes through `toast(msg)`. User-facing text uses plain British/Canadian spelling ("colour").

## How Jonathan works

- This is a teaching tool: students run the live version in their own browsers, so a regression reaches a whole class at once.
- **Propose before implementing.** Before editing `index.html`, say what you plan to change and where, and wait for approval. Requests come as a numbered list and often end with "askquestionsfirst". In that case, also ask about the genuinely open choices (with recommended options). Within an approved plan, make sensible small calls and list them at the end.
- **Make targeted edits, not rewrites.** Change the fewest lines that do the job, and don't restructure or reformat code you weren't asked to touch.
- Keep replies short. Deliver the change, then say what changed in a few bullets, with one next step if there is one.
- Never delete files without asking.
- He tests on his own hardware and real class data. Say clearly which fixes were checked only in a headless browser or with synthetic data, and what he should check by hand.
- Every release:
  - bump `MOM_VERSION`
  - update `README.md` (and the user guide, if asked)
  - update the Help text (`#modal-help`) when a module's behaviour changes.

## Product rules

- App text stays generic: no course-, school- or place-specific examples, warnings or wording (local datasets only as presets). The credit line is "Jonathan Cinnamon" only. The old name "Carto Studio" must not appear in the UI, though the old storage keys stay as they are.
- Students use their own laptops (Windows/Mac, mixed browsers, often 8 GB RAM or less), and classes run 50–110 students on the live site. **Memory use is a release criterion**: never copy layer data per step, page or map when one shared copy will do. See "Memory" below.
- Keep the interface simple. Longer explanations go behind small ⓘ buttons (`atInfo(text)` in the Atlas), not into on-screen paragraphs. Show only the controls that apply to the current mode.

## Atlas (v89+)

Design decisions (do not reintroduce the v87/v88 approaches):
- **One set of pages, two reading styles.** `atlas.mode` is `'book'` or `'scroll'` ("Reads as"), set by the author; readers can't switch it. Every page type works in both styles. Never add a page type or setting that exists in only one style. Style-specific settings (spread and fill in Book; guided tour in Scrolling) are kept in the data and simply ignored by the other style.
- Page kinds are `map`, `layout`, `image`, `text`, `embed` and `section`. There are no "unlisted" pages, and no page refers to another page's map.
- **Panels.** A map page holds up to 4 panels: the page's own map, plus `p.more[]` items with `t:'map'|'image'|'text'`. Each panel owns its data.
  - Arrangement is `p.arr`: `side`/`compare` for 2, `big`/`row` for 3, `grid` for 4. Use `atArrOpts`/`atArrFix`. Compare needs two maps.
  - Linked maps: `p.sync`. Compare divider position: `p.cmpX`.
- **Guided tour** (the UI name; the field is `section.pin`): in Scrolling style, the pages after a section become cards scrolling past a map held on screen. It has a step bar (◀ n / N ▶ plus dots) and ←/→ keys. Do not call it "pin" in the UI.
- Readers' **map of all the maps** (`atlas.overview`) replaced the old locator page. `atlas.sources` adds an automatic Sources page. Captions are controlled by `atlas.figNums` and `atlas.apparatus` (`none`/`compact`/`full`). Each page has `source`, `alt` and `pageText` (`below`/`above`/`left`/`right`/`box`).
- `atSanitize` must load any older atlas unchanged in look: new settings default to off for old atlases.

Data and memory:
- **Layer data is stored once per atlas** in `atlas.geo[key]` (features without styling). A map layer stores `{gk, st, si}`: the key, a table of distinct styles, and the style index for each feature.
  - Pack with `atPackSnap`.
  - Rebuild for in-app drawing with `atUnpackLayer`.
  - `atPruneGeo` drops unused data on load.
  - Merge (`runMerge`) must copy the `geo` entries the copied pages use.
- **Exports, previews and share links all go through `atViewData(atlas, {live, deep, only})`.**
  - Live maps carry HTML with `"@@L<n>@@"` tokens instead of data.
  - The viewer's `fcStr`/`mapSrc` put the data back only when a map is mounted.
  - Don't inline full feature collections into page HTML.
- The viewer mounts map iframes only near the screen and unmounts them when they are far away (Scrolling). A guided tour keeps only the current and next map live. Keep it that way; reference figures are ≤5 live maps and ~180 MB heap for a 10-page atlas built on an 8,000-polygon layer.
- Pictures: `thumb` is 900 px. `still` (for the PDF) is 2800 px for a single map, or 1600 px when the map shares its page. In the reader's copy they appear once, inside `maps[]`.
- `atCaptureMap` must show Map Maker *before* `embedSnapshot()`. A hidden Leaflet map gives a zero-size view.

Viewer modes (`atlasViewer(root, D)`, self-contained, `.toString()`-ed):
- `D.__pdfOnly`: PDF only.
- `D.__cover`: cover thumbnails.
- `D.__single` (+ `__still`): one page for the editor's live preview, drawn in the atlas's style.
- `D.__onView` / `D.__onCmp`: callbacks so that panning a map or dragging a divider in the preview saves to the page. Panning sets `stale=true`; "↻ Retake" (`atRetakeStill`) takes a new picture.
- The preview updates text in place without reloading iframes; it compares a picture-only render (`atPvSig`). If you add a new text element to the viewer, add its selector to `AT_PV_TXT`.
- In `__single` mode, never attach document-level listeners. Message listeners are replaced per root through `root._momL`.
- YouTube/Vimeo won't play under `file://` or in sandboxed `srcdoc` frames, so the play button opens the video in a new tab there. That's expected, not a bug.

Map Maker → Atlas: Send / Export ▾ → "Add to or update an atlas page" lists map pages by their number in the (numbered) Atlas page list.
- ＋ Add calls `atSendFromMaker(id, {add:true})`.
- ↻ k calls `atSendFromMaker(id, {k})`.

## Testing

There is no test suite. Before reporting done:
1. **Syntax check:** extract the large inline `<script>` and run `node --check` on it. Also check that `atlasViewer.toString()` contains no literal `</script>` and that `ATLAS_CSS` contains no backticks or `${`.
2. **Browser test:** use headless Chromium with Playwright, served over HTTP; the CDN libraries need network access. The app runs inside one IIFE, so for scripted tests make a *test copy* with `window.__ev=c=>eval(c);` inserted just before the IIFE's closing `})();`, then call app functions through it. Never ship that hook.
3. **Atlas checks:**
   - both reading styles
   - a guided tour
   - a 4-panel page, compare and linked maps
   - the readers' map and search (`/`)
   - PDF export
   - an exported atlas opened on its own
   - a share-link round trip (`encodeShare` → `decodeShare` → `atSanitize`)
   - an atlas saved by an older version
   - zero page errors in the console
4. **Memory check** for data-heavy changes: add an ~8,000-polygon layer, build ~10 map pages, then compare JSON size and heap (CDP `Runtime.getHeapUsage` after GC) with the previous release. Also compare whole-browser memory (renderer and GPU process sizes, e.g. from `ps` or Chrome's task manager) during captures and Composer sends: the JS heap misses image and graphics memory, which is most of the cost there.

When editing, prefer small, anchored find-and-replace edits on unique strings over rewriting large blocks. Re-run the syntax check after every batch of edits.
