# Glossary

Terms this project uses in a specific way. If code, issues, or docs use a different word for one of these, the code is wrong.

## Animatic

The deliverable: a rendered video assembled from Excalidraw boards, whose elements appear in sync with a narration track. Not "animation" (nothing is authored frame-by-frame) and not "video editing" (no compositing, keyframes, or timeline editing of footage).

## Scene

One page of an animatic. Shown as a unit, then replaced — the page flips. A scene owns the elements on its canvas and the timing of when each element appears. Scene boundaries are the only place the canvas can entirely change.

## Board

The Excalidraw document behind a scene (`scene_001.excalidraw`). The editable artifact. A board is not the deliverable; the animatic is.

## Element

One item on a board: a shape, a text run, an arrow, a freedraw stroke, or a placement of a library icon. Elements animate independently within their scene.

## Frame

One rendered instant of a scene. Frames are produced by the renderer and never authored. Not a screenwriting beat.

## Transition

What happens between one scene and the next: `cut` (instant replacement) or `draw` (the incoming board's strokes write themselves on).

## Cue

The binding between one element's appearance and a moment in the narration. The unit of voice sync.
