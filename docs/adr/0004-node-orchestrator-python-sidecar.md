# Node orchestrator, Python sidecar behind a files-and-JSON seam

The CLI, project state, review UI and renderer are TypeScript on Node 22, because Remotion, the Excalidraw fork and the review UI are all Node already. Speech recognition and forced alignment are a Python 3.12 sidecar, because the Arabic alignment tooling has no Node equivalent. The sidecar is a one-shot leaf process — never a daemon, never a socket — invoked with files in and files out, versioned JSON Schemas in `contracts/`, JSONL progress on stderr, and killed by process group.

**Consequences.** Two runtimes must be installed, so `setup` provisions uv and the sidecar env and `doctor` checks both. The contract between them is a versioned schema, which makes the boundary testable without running either side. A pure-Node promise is impossible while Arabic alignment is Python; a pure-Python promise would mean re-implementing the renderer. Adding a socket or daemon later would be a reversal, not an extension. See #6.
