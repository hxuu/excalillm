# Fork Excalidraw for Arabic text

We use `@excalidraw/excalidraw` (pinned at 0.18.1) as the board format, the embedded review editor, and the base of the render path. Upstream has no first-class RTL, no public custom-font API, no bidi-aware wrapping, and no bundled Arabic font — the Arabic-font PR was rejected. Rather than post-process its output, we maintain an in-repo fork that registers Arabic fonts, adds an explicit direction flag, and fixes RTL line breaking.

**Consequences.** Every upstream upgrade becomes a merge. Rendering must go through a real Chromium build, because resvg and node-canvas both mis-render Arabic bidi — which rules out the cheap headless SVG path. See #3.
