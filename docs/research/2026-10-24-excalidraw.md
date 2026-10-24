# Excalidraw as a video render substrate — primary-source research

Researched 2026-10-24. Nothing written to the excalillm repo.

**Sources of truth used**

| Thing | Version / revision |
|---|---|
| `@excalidraw/excalidraw` published | **0.18.1**, published 2026-04-20 ([npm](https://www.npmjs.com/package/@excalidraw/excalidraw)) |
| Excalidraw `master` | commit [`b7de45e`](https://github.com/excalidraw/excalidraw/commit/b7de45e88111f5680ae4bb3c3f3541c7d3b279fe), 2026-10-23 — **ahead of npm**, export API rewritten |
| Remotion | **4.0.538** ([npm](https://www.npmjs.com/package/remotion)) |
| `excalidraw-animate` | 0.7.2, published 2026-04-24, last commit 2026-05-29 ([repo](https://github.com/dai-shi/excalidraw-animate)) |
| `excalimate` | 0.5.1, pushed 2026-10-24 ([repo](https://github.com/excalimate/excalimate)) |
| resvg | issues checked 2026-10-24; `direction` still unsupported |

Confidence tags: **[verified]** = read in source/docs/issues, **[inferred]** = follows from source but not explicitly documented, **[unknown]** = not established.

---

## TL;DR — flags that would invalidate "Excalidraw is the render substrate"

1. **Arabic has no bundled font and no custom-font API.** All 7 bundled/registered Excalifont faces declare no Arabic Unicode ranges; Arabic always falls through to the OS `sans-serif` stack. `FONT_FAMILY` is a numeric enum, `Fonts.register()` is `private`, and maintainers say custom font injection is "not yet" available (issue open since 2025-02-13). **[verified]** → Arabic layout is machine-dependent unless you fork.
2. **RTL in SVG export is currently broken for right-aligned text**, and resvg **ignores `direction`/`unicode-bidi`** entirely. The two most attractive "headless SVG → frames" paths are therefore the two that mis-render Arabic. **[verified]**
3. **Canvas RTL rendering depends on a real DOM**: Excalidraw sets the canvas `dir` attribute and appends the canvas to `document.body` to get mixed RTL/LTR right. node-canvas has neither behaviour. **[verified in code]**
4. **Text wrapping has zero bidi awareness.** The tokenizer only knows whitespace, hyphen, CJK, emoji. Mixed Arabic/Latin line breaking and caret placement are approximate; cursor bugs are open (#12352, Oct 2026). **[verified]**
5. **The export API is churning.** `master` removed `exportToSvg` / `exportToCanvas` / `exportToBlob` / `getFreeDrawSvgPath` from the package root and replaced them with `exportScene({type, data, config})`. Pin 0.18.1 or your renderer breaks on upgrade. **[verified]**
6. **No stable per-element handles in exported SVG.** `data-id` is only set under `isTestEnv()`. You cannot select "element X" in production export output. **[verified]**
7. Every `exportToSvg` call re-fetches and re-subsets fonts (no subset cache) — per-frame SVG export is both slow and bloated. **[verified]**

Net: Excalidraw is a good **authoring format and editor**, and a mediocre-to-bad **frame renderer** for Arabic/RTL video. Treat the JSON as the source of truth and render it yourself, or render once and animate the DOM in Chromium.

---

## 1. Arabic / RTL text in the current text element

**Verdict: partial, implicit, and not a first-class feature. [verified]**

What exists:

- **Direction is inferred, not stored.** `isRTL(text)` tests whether the first strong directional character is RTL (`^[^LTR]*[RTL]`, ranges `\u0591-\u07FF`, `\uFB1D-\uFDFD`, `\uFE70-\uFEFC`). There is no `direction` field on `ExcalidrawTextElement`, and no RTL toggle. → [`packages/common/src/utils.ts#L387-L402`](https://github.com/excalidraw/excalidraw/blob/master/packages/common/src/utils.ts#L387-L402), [`packages/element/src/types.ts#L253-L291`](https://github.com/excalidraw/excalidraw/blob/master/packages/element/src/types.ts#L253-L291)
- **Alignment is physical only**: `TEXT_ALIGN = { LEFT, CENTER, RIGHT }`, default `left`. No `start`/`end`. Arabic text defaults to left-aligned. → [`packages/common/src/constants.ts#L277`, `#L434`](https://github.com/excalidraw/excalidraw/blob/master/packages/common/src/constants.ts#L277)
- **Canvas renderer** sets `canvas.setAttribute("dir", rtl ? "rtl" : "ltr")` and, for RTL, temporarily appends the canvas to `document.body` — comment: *"to correctly render RTL text mixed with LTR, we have to append it to the DOM"*. → [`packages/element/src/renderElement.ts#L661-L680`](https://github.com/excalidraw/excalidraw/blob/master/packages/element/src/renderElement.ts#L661)
- **Editing (WYSIWYG)** uses per-line direction (`dir=auto`, `unicode-bidi: plaintext`) and, for lines containing RTL chars, asks the browser's native layout where the caret is. → [`packages/excalidraw/wysiwyg/textWysiwyg.tsx#L108-L118`, `#L655-L680`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/wysiwyg/textWysiwyg.tsx#L108)
- **SVG export** computes direction **once per element**, sets `direction` and `text-anchor`, and emits one `<text>` per visual line. → [`packages/excalidraw/renderer/staticSvgScene.ts#L808-L830`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/renderer/staticSvgScene.ts#L808)

**Bidi and shaping.** Excalidraw does no shaping and no bidi reordering itself. Canvas 2D and SVG `<text>` delegate both to the browser (HarfBuzz + ICU in Chromium). That works *if* an Arabic-capable font is in the stack — and it isn't (see §2). **[verified]**

**Wrapping.** `wrapText()` / `getWrappedTextLines()` tokenize on whitespace, hyphen, CJK break rules and emoji, then measure candidate lines with `canvas.measureText`. No UBA/bidi pass, no `\u200F`/`\u2066` isolating marks, no per-run direction. Mixed-direction lines are measured in logical order; prefix widths are only correct for LTR. → [`packages/element/src/textWrapping.ts`](https://github.com/excalidraw/excalidraw/blob/master/packages/element/src/textWrapping.ts), measurement in [`textMeasurements.ts`](https://github.com/excalidraw/excalidraw/blob/master/packages/element/src/textMeasurements.ts)

**Open issues / known limitations (as of 2026-10-24):**

- [#12443](https://github.com/excalidraw/excalidraw/pull/12443) *(open PR, 2026-10-23)* — right-aligned RTL text in SVG exports falls outside its box; canvas is fine, SVG is not. **This is the export path you'd use for video.**
- [#12352](https://github.com/excalidraw/excalidraw/issues/12352) *(open, 2026-10-19)* — cursor moves wrong direction / letters shift when typing Hebrew/English.
- [#8645](https://github.com/excalidraw/excalidraw/pull/8645) *(open since 2024-10-15)* — highlight rectangle for RTL text.
- [#11399](https://github.com/excalidraw/excalidraw/pull/11399) *(open PR, 2026-05-25)* — adds 5 Arabic fonts + auto script switching. Review comment (2026-08-24) rejects the approach: *"Hardcoding language-specific logic … into the core appState doesn't scale"*. Effectively not landing.
- Font requests still open: [#8394](https://github.com/excalidraw/excalidraw/issues/8394) (2024-08), [#9984](https://github.com/excalidraw/excalidraw/issues/9984) / [#9986](https://github.com/excalidraw/excalidraw/issues/9986) (2025-09), [#11725](https://github.com/excalidraw/excalidraw/pull/11725) (Playpen Arabic, 2026-07).
- UI-level RTL (menus, dialogs) is a separate, actively-churning area (44 results for "RTL"; e.g. #12421, #12404, #12386). UI language `ar-SA` exists with `rtl: true` → [`packages/excalidraw/i18n.ts#L24`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/i18n.ts#L24).

**Confidence:** RTL mechanics **[verified]**; quality of mixed-bidi wrapping **[inferred from absent code]**; shaping correctness on canvas in Chromium **[verified by platform behaviour, not tested here]**.

---

## 2. Custom fonts

**Verdict: not supported in the public API. Fork, or fake it with CSS (and lose export embedding). [verified]**

Facts:

- Fonts are a **numeric enum**, not a font-family string: `Virgil:1, Helvetica:2, Cascadia:3, (4 reserved), Excalifont:5, Nunito:6, "Lilita One":7, "Comic Shanns":8, "Liberation Sans":9, Assistant:10`. Comment on 4: *"historically used for Assistant … or custom font (Obsidian)"*. → [`packages/common/src/constants.ts#L143-L154`](https://github.com/excalidraw/excalidraw/blob/master/packages/common/src/constants.ts#L143)
- `Fonts.register(family, metadata, ...faces)` is **`private static`**, with `// TODO: likely we will need to abandon number value in order to support custom fonts`. → [`packages/excalidraw/fonts/Fonts.ts#L386-L403`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/fonts/Fonts.ts#L386)
- `getFontFamilyString()` reverse-maps the numeric id to the enum key, then appends generic fallbacks (`sans-serif`, `Segoe UI Emoji`). A family not in the enum renders as **only** `Segoe UI Emoji`. → [`packages/common/src/utils.ts#L137-L156`](https://github.com/excalidraw/excalidraw/blob/master/packages/common/src/utils.ts#L137)
- Issue [#9132](https://github.com/excalidraw/excalidraw/issues/9132) "Enable custom fonts when embedded as an npm package" (open since 2025-02-13). Maintainer reply 2025-02-16: *"Not yet, but should be there in the future!"* [#10184](https://github.com/excalidraw/excalidraw/issues/10184) (2025-10) is still unanswered.
- Fonts shipped: Virgil, Helvetica (system), Cascadia, Excalifont, Nunito, Lilita One, Comic Shanns, Liberation Sans, Xiaolai (CJK), Emoji. **No Arabic ranges are declared in any of them** — grep for `U+06xx`/`U+0750` in `packages/excalidraw/fonts/*/index.ts` returns nothing; Excalifont's 7 faces cover Latin-1/Latin-Ext-A, punctuation, spacing modifiers, and one Cyrillic range. → [`packages/excalidraw/fonts/`](https://github.com/excalidraw/excalidraw/tree/master/packages/excalidraw/fonts)

### What a custom Arabic font actually requires

| Goal | Requirement |
|---|---|
| Show Arabic in the editor | Nothing — OS fallback already happens. Just don't expect the font you want. |
| Pick a specific font in the UI | Fork: add enum entry (e.g. `"Noto Naskh Arabic": 11`), `FONT_METADATA` metrics (unitsPerEm/ascender/descender/lineHeight), and `ExcalidrawFontFace` descriptors; register in `Fonts.init()`. **[verified-by-code]** |
| Embedded in exported SVG/PNG | The face must be registered in `Fonts.registered` **and** served as **woff2**, because export subsets via `subsetWoff2GlyphsByCodepoints` and inlines base64 `@font-face` (`toCSS`). TTF/OTF input is not supported by that path. **[verified]** → [`ExcalidrawFontFace.ts#L37-L70`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/fonts/ExcalidrawFontFace.ts#L37), [`export.ts#L1101-L1112`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/scene/export.ts#L1101) |
| Host-page-only hack (review UI) | Inject `@font-face { font-family: Assistant; src: url(NotoNaskh.woff2) }` and set elements to `fontFamily: 10`. Canvas + measurement will use it; **SVG export will not embed it**, so rasterizers need the font installed or you must inject the `@font-face` into the exported SVG yourself. **[inferred, high confidence]** |

Note the export pipeline is otherwise good: it subsets per glyph set, so exported SVGs are smaller than full-font embeds — but that's per export, see §5.

**Confidence:** API absence and woff2/subset requirement **[verified]**; CSS-override hack **[inferred]**.

---

## 3. Element model and export surface

### `.excalidraw` (schema v2) — [docs](https://docs.excalidraw.com/docs/codebase/json-schema) **[verified]**

```json
{ "type": "excalidraw", "version": 2, "source": "https://excalidraw.com",
  "elements": [...], "appState": {...}, "files": { "<fileId>": {...} } }
```

- `appState` is a sparse scene config (`gridSize`, `viewBackgroundColor`, `exportBackground`, `exportEmbedScene`, `exportScale`, `exportWithDarkMode`, …).
- `files` maps id → `{mimeType, id, dataURL, created, lastRetrieved}` for image elements.
- Element base (`_ExcalidrawElementBase`): `id, x, y, strokeColor, backgroundColor, fillStyle, strokeWidth, strokeStyle, roundness, roughness, opacity, width, height, angle (radians), seed, version, versionNonce, index (fractional), isDeleted, groupIds, frameId, boundElements, updated, created, link, locked, customData`. → [`packages/element/src/types.ts#L40-L88`](https://github.com/excalidraw/excalidraw/blob/master/packages/element/src/types.ts#L40)
- Text adds: `fontSize, fontFamily (number), baseFontSize, text (wrapped), textAlign, verticalAlign, containerId, originalText, autoResize, lineHeight (unitless), labelPosition`. → [`#L253-L291`](https://github.com/excalidraw/excalidraw/blob/master/packages/element/src/types.ts#L253)
- `seed` makes roughjs output deterministic for a given element — good news for frame-to-frame consistency. **[verified]**
- Frame elements exist (`frame`, `magicframe`), usable as scene/camera regions. Unknown whether a frame's clip is applied in your own renderer; export does create clipPaths (`frame-<id>`).

### Export functions — pinned 0.18.1 vs `master`

**0.18.1** (what `npm i` gives you) re-exports from `@excalidraw/utils/export`, verified in the published `.d.ts`:

```ts
exportToSvg({ elements, appState, files, exportPadding, renderEmbeddables, exportingFrame, skipInliningFonts, reuseImages }): Promise<SVGSVGElement>
exportToCanvas({ elements, appState, files, maxWidthOrHeight?, getDimensions?, exportPadding?, exportingFrame? }): Promise<HTMLCanvasElement>
exportToBlob({ ...ExportOpts, mimeType?, quality?, exportPadding? }): Promise<Blob>
```
Docs: <https://docs.excalidraw.com/docs/@excalidraw/excalidraw/api/utils/export>. Also exported in 0.18.1: `getFreeDrawSvgPath`, `getCommonBounds`, `getVisibleSceneBounds`, `convertToExcalidrawElements`, `serializeAsJSON`, `restore*`.

**`master` (unreleased, 2026-10-23)** replaced all of that with:

```ts
exportScene({ type: "svg" | "png" | "jpeg" | "canvas", data: { elements, appState?, files? }, config? })
exportToClipboard(...)
```
`exportToSvg`/`exportToCanvas`/`exportToBlob`/`getFreeDrawSvgPath` are **no longer exported from the package root** → [`packages/excalidraw/index.tsx#L426-L432`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/index.tsx#L426), [`scene/export.ts#L962`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/scene/export.ts#L962). Master also adds a `RenderEnvironment` (`document`, `createCanvas`, `createImage`, `createPath`, `textMetrics`, `loadFonts`) so classes can run under Node/`linkedom`. → [`renderEnvironment.ts#L1-L60`](https://github.com/excalidraw/excalidraw/blob/master/packages/element/src/renderEnvironment.ts#L1)

**Plan implication:** pin `@excalidraw/excalidraw@0.18.1` and write an adapter, or target `exportScene` and accept unreleased APIs. Do not straddle both. **[verified]**

---

## 4. Animation feasibility

**Verdict: feasible, but you will be building the animator. There is no animation model in Excalidraw — no keyframes, no timing, no per-element SDK hooks. [verified]**

### Building blocks that exist

| Block | Availability |
|---|---|
| Per-element JSON with geometry (`x,y,w,h,angle`, `points`, `seed`) | **public**, schema v2 |
| Deterministic rough geometry | `ShapeCache.generateElementShape(el, theme)` → roughjs `Drawable` / `Drawable[]` / SVG path string; `getElementShape`, `getFreeDrawSvgPath` → [`shape.ts#L87-L170`, `#L1148`](https://github.com/excalidraw/excalidraw/blob/master/packages/element/src/shape.ts#L87) |
| `getFreeDrawSvgPath` | **exported in 0.18.1**, removed from root in `master` |
| Scene bbox (`getCommonBounds`), element bbox (compute from x/y/w/h + angle; `getElementAbsoluteCoords` is internal) | partially public |
| Exported SVG has one `<g transform="translate(x y) rotate(deg cx cy)">` per element, in z-order | **verified** in `renderElementToSvg` → [`staticSvgScene.ts#L140-L160`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/renderer/staticSvgScene.ts#L140) |
| Stable per-element selector in that SVG | **no** — `data-id` only when `isTestEnv()`; only clipPath/mask ids reference element ids |
| Partial stroke ("draw-on") | **no API**; must be done with `stroke-dasharray`/`stroke-dashoffset` or path-`d` interpolation |

### Existing tools

- **[dai-shi/excalidraw-animate](https://github.com/dai-shi/excalidraw-animate)** — 2.1k★, v0.7.2, last commit 2026-05-29, depends on `@excalidraw/excalidraw@0.18.1`. Loads `.excalidraw`/`.excalidrawlib`, assigns per-element "Order"/"Duration", animates the exported SVG with **SMIL `<animate>`** on `opacity`, `d`, and `stroke-dashoffset`. Implementation walks `childNodes` by index and rewrites `d` to degenerate curves (`animatePolygon`, `animatePath`, `animateFillPath`). WebM export is `getDisplayMedia` screen capture — README lists it as a known imperfection. **Existence proof and technique reference, not a library to depend on.** → [`src/animate.ts`](https://github.com/dai-shi/excalidraw-animate/blob/main/src/animate.ts)
- **[excalimate](https://github.com/excalimate/excalimate)** — 96★, pushed 2026-10-24, `@excalidraw/excalidraw ^0.18.0`, React 19. Keyframes for opacity/position/scale/rotation, plus **arrow draw-on**; exports MP4/WebM via WebCodecs (`mp4-muxer`, `webm-muxer`), GIF via `gif.js`, animated SVG (CSS keyframes or SMIL), Lottie/dotLottie. All client-side. Draw-on is `pathLength`-normalized `stroke-dasharray: 1` + `stroke-dashoffset` → [`player-runtime/src/svgAdapter.ts#L120-L127`](https://github.com/excalimate/excalimate/blob/main/packages/player-runtime/src/svgAdapter.ts#L120). Their `docs/export-runtime.md` documents the same pitfalls this report does (DOM raster not worker-safe, per-format fallbacks). **Closest prior art.**
- **[dai-shi/excalidraw-claymate](https://github.com/dai-shi/excalidraw-claymate)** — 572★, stop-motion/slides, not programmatic.
- `inscribed`, `excalidraw-smart-presentation`, `trazo`, `Archscribe` — slide/stop-motion variants, smaller or unverified.

### What animating actually costs

- To get per-element control you must **either** (a) render each element separately with `exportToSvg([el])` and compose your own SVG (N calls per frame — expensive), **or** (b) write your own SVG emitter over `ShapeCache`/`getFreeDrawSvgPath` (full control, but you own text layout, frames, images, arrowheads), **or** (c) export the whole scene once and drive the DOM in Chromium, using **element order** to map nodes to elements (fragile, but what excalidraw-animate does).
- Text draw-on is not a stroke; it can only fade or wipe per line (`element.text` is already wrapped into visual lines).
- Frames, images, and bound labels are special-cased in export and are the parts most likely to break in a custom renderer.

**Confidence:** building blocks **[verified]**; the "no animation model" and cost analysis **[verified/inferred]**.

---

## 5. Headless rendering — practical routes and pitfalls

| Route | Fidelity | Arabic/RTL | Notes |
|---|---|---|---|
| **Playwright/Puppeteer + Chromium**, page calls `exportToSvg` (or mounts `<Excalidraw viewModeEnabled>`) | highest | correct **if** Arabic font is installed/`@font-face`d in that browser | Needs Chrome + Linux shared libs; ~150–400 ms/frame cold, better warm. This is what Remotion does internally. **[verified docs + inferred perf]** |
| **Remotion** (`renderMedia` / `renderStill`, each frame = React component over the JSON) | highest | correct if font loaded via `@remotion/fonts` / `FontFace` + `delayRender` | Chromium-based, `chromiumOptions`, `imageFormat: png`, `concurrency`. Free only for individuals/orgs ≤3 people; otherwise Company License (automation: $0.01/render, $100 min). **[verified]** → [renderMedia](https://www.remotion.dev/docs/renderer/render-media), [license FAQ](https://www.remotion.dev/docs/license/faq) |
| **Node + `exportToSvg` + `linkedom`** | medium | depends on final rasterizer | Only clean on `master` via `RenderEnvironment`; 0.18.1 needs global DOM/canvas shims. Fonts must be supplied through `environment.loadFonts`; without `document.fonts` it warns and skips loading. **[verified]** → [export.test.ts#L552-L600](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/tests/scene/export.test.ts#L552) |
| **Node + `exportToCanvas` + node-canvas/@napi-rs/canvas** | low-medium | **broken for mixed bidi** | Excalidraw sets canvas `dir` and DOM-appends the canvas; node canvases don't implement that. Also needs `createPath`/`Path2D` or every freedraw throws. **[verified code / inferred result]** |
| **`exportToSvg` → resvg / resvg-js** | fast, deterministic | **broken** | resvg has no native text layout: *"No native text rendering"*, and [#1184 "Support `direction` and `unicode-bidi` on text"](https://github.com/linebender/resvg/issues/1184) is **open** (2026-10-21). RTL multiline `<tspan dy>` bugs #1093 (fixed 2026-07), #583 dx/dy RTL (open since 2023), #475 bidi ordering (open since 2021). **[verified]** |
| **`exportToSvg` → headless Chromium screenshot of the SVG alone** | high | correct | Cheap variant of route 1; still a browser, but no Excalidraw mount. Good default. **[inferred]** |

### Specific pitfalls

1. **Fonts.** Bundled fonts have no Arabic. In headless Linux, `sans-serif` usually means DejaVu/Liberation → **tofu or wrong Arabic**. Install Noto Naskh/Amiri/ Cairo in the image *and* load the same file in the browser; otherwise editor preview and render disagree. **[verified ranges / inferred environment]**
2. **Font inlining cost.** `exportToSvg` calls `Fonts.generateFontFaceDeclarations` → for every family/charset it fetches woff2 and runs the woff2 subsetter, then base64-inlines. No subset cache; only HTTP `cache: "force-cache"`. Doing this per frame is pure waste — export once and animate, or hoist font declarations out of the frame loop. **[verified]** → [`Fonts.ts#L229-L264`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/fonts/Fonts.ts#L229), [`ExcalidrawFontFace.ts#L85-L100`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/fonts/ExcalidrawFontFace.ts#L85)
3. **SVG bloat.** `exportEmbedScene: true` embeds the whole scene JSON in `<metadata>` (documented to increase file size); font `@font-face` base64 adds more per file. **[verified docs]**
4. **Browser-only.** Docs: *"Since Excalidraw doesn't support server side rendering"*; Next.js needs `dynamic(..., { ssr: false })`. **[verified]** → [Integration](https://docs.excalidraw.com/docs/@excalidraw/excalidraw/integration)
5. **Assets / offline.** Browser build loads fonts/emoji from `EXCALIDRAW_ASSET_PATH` (default esm.sh). In an offline render container you must self-host `dist/prod/` assets. **[verified]** → [`ExcalidrawFontFace.ts#L12-L16`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/fonts/ExcalidrawFontFace.ts#L12), integration docs.
6. **Chrome on Linux** needs ~15 shared libs (`libnss3`, `libgbm-dev`, `libpango`, …); Alpine unsupported, NixOS unsupported (Remotion). **[verified]** → [Linux dependencies](https://www.remotion.dev/docs/miscellaneous/linux-dependencies)
7. **FFmpeg.** Remotion bundles its own FFmpeg; if you roll your own, `image2` demuxer gives `-framerate N -i 'frame-%05d.png'` and requires identical size/pixfmt across frames. **[verified]** → [ffmpeg-formats image2](https://ffmpeg.org/ffmpeg-formats.html#image2-1)

---

## 6. Excalidraw libraries (`.excalidrawlib`)

**Format [verified in source + real files]:**

```jsonc
// v2 (current)
{ "type": "excalidrawlib", "version": 2, "source": "https://excalidraw.com",
  "libraryItems": [
    { "id": "...", "status": "published" | "unpublished",
      "elements": [ /* full Excalidraw elements, coordinates relative to item origin */ ],
      "created": 1690000000000, "name": "Business object" }
  ] }
```

- Type union and constants: `VERSIONS.excalidrawLibrary = 2`, `EXPORT_DATA_TYPES.excalidrawLibrary = "excalidrawlib"` → [`packages/common/src/constants.ts#L350`, `#L421`](https://github.com/excalidraw/excalidraw/blob/master/packages/common/src/constants.ts#L350).
- `LibraryItem = { id, status, elements, created, name?, error? }` → [`packages/excalidraw/types.ts#L656-L665`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/types.ts#L656).
- v1 files exist in the wild: `{type, version:1, library: [[element,...], ...]}` — a bare array of element arrays, no ids/names. `restoreLibraryItems()` migrates array-shaped items by generating id/created/status. → [`packages/excalidraw/data/restore.ts#L1391-L1425`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/data/restore.ts#L1391). I confirmed `lipis/stars.excalidrawlib` is v1 and `timothygalvin/excalidraw-archimate-template.excalidrawlib` is v2/`libraryItems`.
- Serialization/loading: `serializeLibraryAsJSON`, `loadLibraryFromBlob`, `loadSceneOrLibraryFromBlob`, `mergeLibraryItems`, `parseLibraryTokensFromUrl` → [`data/json.ts#L137`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/data/json.ts#L137), [`packages/excalidraw/index.tsx#L434-L465`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/index.tsx#L434).
- Programmatic install: `excalidrawAPI.updateLibrary({ libraryItems, merge?, prompt?, openLibraryMenu? })` → [api docs](https://docs.excalidraw.com/docs/@excalidraw/excalidraw/api/props/excalidraw-api#updatelibrary).

**Where they live.** Official catalog: <https://libraries.excalidraw.com>, backed by [`excalidraw/excalidraw-libraries`](https://github.com/excalidraw/excalidraw-libraries) — 232 libraries, each `libraries/<author>/<name>.excalidrawlib`, indexed by `libraries.json` (`name, description, authors, source, preview, created, updated, version`).

**Arabic relevance — bad news. [verified]** Submission guideline #4: *"We only accept libraries in English. This includes any texts/labels, and library description and titles."* A scan of the 232 catalog entries for `arab|rtl|islam|quran|persian|hebrew` returns **0**. So:
- Arabic *labelled* icon sets are not in the official catalog; they live as self-hosted `.excalidrawlib` files (your own repo, or via `libraryReturnUrl` / a library URL token).
- Language-neutral icon sets (arrows, architecture, IT icons, shapes, stick figures) are still fine and are the bulk of the catalog; Arabic text would be added as separate elements in your own scene, not inside library items.

---

## 7. Embedding the React component for a local review UI

**Verdict: straightforward, and yes — edit + export programmatically both work. [verified]**

- **Peer deps:** `react ^17.0.2 || ^18.2.0 || ^19.0.0`, same for `react-dom` → [`packages/excalidraw/package.json`](https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/package.json).
- **CSS:** `import "@excalidraw/excalidraw/index.css";` (this is the 0.18 path; older tutorials show `dist/excalidraw.css`). Container must have explicit height.
- **Browser-only:** no SSR; Next.js needs `ssr: false`. Set `window.EXCALIDRAW_ASSET_PATH` if you want to self-host assets.
- **Props that matter:** `initialData` (elements/appState/files/libraryItems, can be a Promise), `excalidrawAPI`, `onChange(elements, appState, files)`, `viewModeEnabled` (read-only review), `UIOptions` (hide menus/export), `theme`, `langCode: "ar-SA"` for RTL UI, `name`. Full list → [Props](https://docs.excalidraw.com/docs/@excalidraw/excalidraw/api/props).
- **Programmatic editing:** `api.updateScene({ elements, appState, captureUpdate })`, `getSceneElements()`, `getAppState()`, `getFiles()`, `updateLibrary()`, `scrollToContent()`, `setActiveTool()`, `history.clear()`, `onChange(cb)`. → [excalidrawAPI](https://docs.excalidraw.com/docs/@excalidraw/excalidraw/api/props/excalidraw-api)
- **Programmatic export:** call the package-level `exportToBlob`/`exportToSvg` with `api.getSceneElements()` + `api.getFiles()` + appState (the docs' own example does exactly this). Alternatively hide the default menu with `UIOptions.canvasActions.export = false` and own the export button. Note `captureUpdate: CaptureUpdateAction.NEVER` when hydrating a scene so you don't pollute undo history.
- **Custom Arabic font in the review UI:** the CSS-override trick in §2 works here (host page `@font-face` + `fontFamily: 10`), but export fidelity still depends on the rasterizer having that font.

---

## Suggested architecture (if you proceed)

1. **Own the JSON.** `.excalidraw` v2 is a clean IR; store it, diff it, use `seed` for deterministic rough shapes.
2. **Renderer = headless Chromium via Remotion**, component draws either (a) inline SVG produced once by `exportToSvg` at 0.18.1, or (b) your own SVG from element JSON. Animate with CSS/JS per element; Remotion gives frame-accurate timing and PNG frames.
3. **Do not** use `exportToSvg` per frame; do not use resvg for RTL; do not rely on `data-id`.
4. **Font:** ship Noto Naskh Arabic (or Cairo/Amiri) as woff2, install it in the container, load via `@remotion/fonts` for preview and inject an `@font-face` with base64 into any exported SVG you keep. Prefer a fork that registers a real `FONT_FAMILY` id if you need Excalidraw itself to export Arabic correctly.
5. **Pin 0.18.1**, wrap export in your own module, and watch #12443 (RTL SVG alignment) and the `exportScene` migration before upgrading.
6. **Review UI** = embedded `<Excalidraw viewModeEnabled={false} langCode="ar-SA">` + `onChange` autosave; render pipeline reads the same JSON.

## Unresolved / would need a spike

- Actual visual quality of Arabic in Chromium canvas vs SVG export for a real mixed Arabic/English diagram (no test executed here).
- Whether `@excalidraw/element` (published at `0.18.0-f0063e113`, i.e. preview-tagged) is stable enough to import `ShapeCache`/`generateElementShape` in production. **[unknown]**
- Remotion + `exportToSvg` inside a component: font subsetting per frame must be memoized; not measured.
- Whether `exportScene` in `master` will keep `RenderEnvironment` as a public option or move it before release. **[unknown]**
