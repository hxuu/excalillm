# Arabic STT for a local pipeline — research report

Compiled 2026-10-24. All WER figures are **after** the benchmark's text normalisation (punctuation removed, diacritics removed, ٠-٩→0-9, hamza forms unified — see `eval.py` in the leaderboard repo). Confidence tags: **verified** (read in a primary source), **inferred** (reasoned from primary sources, not directly stated), **unknown**.

---

## 1. Does "Vocalinux" exist?

**Yes — verified.** https://github.com/VocaHQ/vocalinux · https://vocalinux.com · PyPI `vocalinux` · current release v0.18.4 (README, fetched 2026-10-24).

What it is:
- Linux **desktop dictation app** (X11 + Wayland), Python 3.11+/GTK3, **AGPL-3.0**, text injected into the focused window via xdotool/IBus/wtype/ydotool/clipboard.
- Engines it wraps (README table): **whisper.cpp (default, Vulkan GPU)**, **OpenAI Whisper** (PyTorch/CUDA), **Faster Whisper** (CTranslate2 INT8), **VOSK**, **Parakeet via sherpa-onnx** (README: "25 European languages" — i.e. **not** an Arabic engine), plus an optional remote HTTP API / self-hosted VocaGateway.
- Original author jatinkrmalik; Codeberg mirror at `codeberg.org/jatinkrmalik/vocalinux`.

Can it be used as a library/CLI for Arabic?

| Path | Verdict |
|---|---|
| Mic dictation in Arabic | **Yes (verified)** — `ar` is in its language catalogue (`web/src/app/languages/page.tsx`) and `--language` takes catalogue codes. Arabic is not in the site's highlighted list but is present in the catalogue array. |
| Batch/CLI transcription of an Arabic file | **No (verified)** — `--transcribe-file PATH` exists (`src/vocalinux/main.py:168`), but it is hard-wired to **TinyDiarize `small.en-tdrz`** (`diarization.py`, `whispercpp_model_info.py:44`), which the catalogue itself labels *"TinyDiarize speaker-turn model, English-only"*. Output = `HH:MM:SS` speaker blocks, segment-level only, no words. |
| Library import | **Not really (verified)** — engines are written for a streaming mic loop (recognition_manager + VAD + injection). No documented public API, no batch entry points, no word timestamps. AGPL-3.0 also makes embedding in a non-AGPL product awkward. |

Bottom line: a dictation front-end, not an Arabic ASR toolkit. Useful only as a reference implementation of "whisper.cpp + Silero VAD + model download/checksum plumbing".

---

## 2. Arabic ASR state of the art, open/local, Oct 2026

Primary source: **Open Universal Arabic ASR Leaderboard** (ELM Research Center), data as displayed 2026-10-19 (last update line: "Oct 19th 2026: Sygma-ASR-Arabic"). https://huggingface.co/spaces/elmresearchcenter/open_universal_arabic_asr_leaderboard · paper https://arxiv.org/abs/2412.13788 (Interspeech 2025 version: https://www.isca-archive.org/interspeech_2025/wang25_interspeech.pdf)

Datasets and what dialect they proxy (from the Munsit paper's Table 1, https://arxiv.org/html/2504.12254v2):
- **SADA** — 10+ dialects, *mostly Saudi* → Gulf proxy. Broadcast/news, dialectal.
- **Common Voice 18** — 25 dialects but read speech, overwhelmingly MSA → MSA proxy.
- **MASC clean/noisy** — 7–8 dialects, MSA lectures (clean/noisy).
- **MGB-2** — ~70% MSA broadcast news (Egypt-origin Aljazeera), rest dialectal + English/French.
- **Casablanca** — 8 dialects, conversational Moroccan → Maghrebi proxy (hardest set for everyone).

### Leaderboard, WER % (lower better)

| Model | Avg | SADA (Gulf) | CV18 (MSA) | MASC clean | MASC noisy | MGB-2 (Eg/bcast) | Casablanca (Maghrebi) |
|---|---|---|---|---|---|---|---|
| **IFM/Jais-ASR** (Apache-2.0) | **22.82** | 30.52 | 8.59 | 16.14 | 25.47 | 9.60 | 46.61 |
| **audarai/Audar-ASR-V1-Turbo** (community lic.) | 23.17 | **28.92** | 8.09 | 16.73 | 27.19 | 11.08 | 47.02 |
| abedbanna/Sygma-ASR-Arabic (Cohere-derived + beam) | 23.01 | 33.63 | 5.24 | 15.92 | 25.10 | 13.48 | 44.67 |
| **CohereLabs/cohere-transcribe-arabic-07-2026** (Apache-2.0) | 24.19 | 36.83 | **5.27** | 16.35 | **25.65** | 15.33 | 45.68 |
| omnilingual-asr/omniASR_LLM_7B (Meta, open) | 28.32 | 41.61 | 8.75 | 19.69 | 29.29 | 14.13 | 56.46 |
| omniASR_LLM_3B / 1B | 29.96 / 29.96 | | | | | | |
| Qwen3-Omni-30B-A3B | 30.71 | 44.82 | 11.46 | 21.47 | 30.85 | 13.09 | 62.55 |
| audarai/Audar-ASR-V1-Flash (0.78B) | 32.04 | 44.36 | 15.38 | 22.56 | 34.19 | 17.05 | 58.71 |
| google/gemma-4-E4B-it | 32.98 | 43.40 | 19.65 | 24.86 | 33.59 | 17.72 | 58.63 |
| Qwen/Qwen3-ASR-1.7B | 33.36 | 45.53 | 16.90 | 24.37 | 34.29 | 16.57 | 64.47 |
| Voxtral-Small-24B / Mini-3B | 34.47 / 42.58 | | | | | | |
| **openai/whisper-large-v3** | **36.86** | 55.96 | 17.83 | 24.66 | 34.63 | 16.26 | 71.81 |
| facebook/seamless-m4t-v2-large | 38.16 | 62.52 | 21.70 | 25.04 | 33.24 | 20.23 | 66.25 |
| **openai/whisper-large-v3-turbo** | 40.05 | 60.36 | 25.73 | 25.51 | 37.16 | 17.75 | 73.79 |
| Qwen/Qwen3-ASR-0.6B | 42.19 | 53.75 | 28.28 | 31.34 | 42.63 | 25.45 | 71.68 |
| openai/whisper-medium | 45.57 | 67.71 | 28.07 | 29.99 | 42.91 | 29.32 | 75.44 |
| nvidia Parakeet CTC 1.1B (universal / concat) | 51.96 / 46.54 | 73.58 / 70.70 | 40.01 / 26.34 | 36.16 / 30.49 | 50.03 / 45.95 | 30.68 / 24.94 | 81.30 / 80.80 |
| facebook/mms-1b-all | 54.54 | 77.48 | 26.52 | 38.82 | 57.33 | 39.16 | 87.95 |

`SeamlessM4T-v2` is *not* competitive for Arabic ASR (38.16). `Parakeet`/`Canary` are European-language models; Parakeet is only on the board because it was force-evaluated — do not consider it for Arabic.

### Dialect-specific reading (verified numbers, inferred labels)
- **MSA (read)**: Whisper-v3 17.8, Jais 8.6, Cohere 5.3.
- **Gulf/Saudi broadcast**: Whisper-v3 56.0 → Jais 30.5 / Audar 28.9. Huge gap; this is where Whisper fails.
- **Egyptian / MSA broadcast news**: Whisper 16.3, Jais 9.6.
- **Maghrebi (Moroccan darija)**: 45–72% for everyone; unusable without fine-tuning.
- **Levantine**: **unknown as a standalone number** — the leaderboard does not publish a per-dialect split, and Levantine is only inside MASC/Casablanca aggregates (verified absence). The only Arabic-WER *per-dialect* public baselines I could find are fine-tunes, e.g. `HebArabNlpProject/whisperLevantine` (whisper-medium, Israeli-Levantine), no comparable zero-shot table. Expect Levantine between Gulf and Egyptian for Whisper, i.e. **~35–55% WER, inferred**.

### Closed reference points (for sanity, cannot run locally)
- **Munsit** (CNTXT/Inception, 2025): Conformer + weak supervision on ~30k h, claims #1 across 18 dialects; **API-only, no weights** (paper https://arxiv.org/abs/2504.12254 · NADI 2025 follow-up https://arxiv.org/abs/2508.08912). The `ksa.ai` model page states plainly that no weights/repo exist on HF (https://ksa.ai/models/munsit).
- **Speechmatics Melia 1** avg 23.51 and **ElevenLabs Scribe v2** 24.54 on the same six sets, measured by IFM (Jais-ASR card table, 2026).

### Notable open weights, with conditions
- **IFM/Jais-ASR** — https://huggingface.co/IFM/Jais-ASR — Apache-2.0, "6B params", **11.4 GB bf16, ~12 GB GPU**, based on Jais-3-4B. Supports `ar`/`msa`/19 dialect codes, `diacritics=True`, `code_switch=True`, dialect label, `[IFM-NoTranscript]` refusal. vLLM plugin with OpenAI-compatible `/v1/audio/transcriptions`. **No timestamps** (inferred: card's API returns `text/language/refused` only).
- **Audar-ASR-V1-Turbo** — https://huggingface.co/audarai/Audar-ASR-V1-Turbo — 2.35B, Qwen3-ASR architecture, **GGUF for llama.cpp** (Q4_K_M 1.28 GB, Q8_0 2.16 GB, +0.64 GB BF16 mmproj), **30 s encoder window**, non-permissive "AudarAI Community License v1.0" (research + limited commercial). Notably CPU-runnable via `llama-mtmd-cli`.
- **Audar-ASR-V1-Flash** — 0.78B, avg WER 32.0, same GGUF story, license "AudarAI Open v1.0".
- **Cohere Transcribe Arabic** — 2.07B (2,065,804,048 params per HF API), **Apache-2.0**, **gated** on HF, vLLM-optimised, RTFx 525 vs Whisper-v3 146 (https://cohere.com/blog/transcribe-arabic, 2026-07-07). Card table uses slightly different numbers than the Oct-2026 leaderboard (25.87 avg vs 24.19) — different decoding/normalisation; treat leaderboard as current.
- **Meta omniASR** — 300M→7B, open, 1600 languages, avg 28–39 for Arabic; the 1B/300M are the only sub-GB *multilingual* options but Arabic quality is mid.

---

## 3. Word / phrase timestamps

| Engine | Words? | Mechanism / format | Confidence |
|---|---|---|---|
| **whisper.cpp** | **Yes, token-level** | C API `token_timestamps`; CLI `-ml 1` ("Word-level timestamp (experimental)", README §Word-level timestamp); server `verbose_json` emits `segments[].words[] = {word,start,end,t_dtw,probability}`; `--dtw {tiny..large.v3}` enables cross-attention DTW for better token times; `-sow` split-on-word; `-wt` word prob threshold. | **verified** (README + `examples/server/server.cpp:1100-1150`) |
| **faster-whisper** | **Yes** | `transcribe(word_timestamps=True)` → `Word(word,start,end,probability)`; same cross-attention+DTW as OpenAI Whisper (not a separate aligner); `vad_filter=True` uses Silero. | verified (README §Word-level timestamps) |
| **WhisperX** | **Yes, best accuracy** | VAD cut → batched faster-whisper → **wav2vec2 phoneme forced alignment**; Arabic align model `jonatasgrosman/wav2vec2-large-xlsr-53-arabic` (`whisperx/alignment.py:77`). 70× realtime large-v2 claim. | verified |
| **stable-ts** | **Yes** | Stabilised cross-attention DTW + silence-based adjustment; `word_timestamps=True` default; also `model.align(audio, text)`. | verified |
| **sherpa-onnx** | Yes (Whisper path) | `SherpaOnnxOfflineWhisperModelConfig.enable_token_timestamps` / `enable_segment_timestamps`; result carries parallel `tokens_arr` + `timestamps` (seconds). | verified (`c-api.h:885-889, 1483-1492`) |
| **VOSK** | **Yes, native word boundaries** | `KaldiRecognizer.SetWords(True)` → JSON `result[] {word,start,end,conf}`; streaming, monotonic times. | verified (`vosk/__init__.py:174`, `SrtResult`) |
| **Jais / Audar / Cohere / Qwen3-ASR / Voxtral / omniASR** | **No** (inferred) | Generative decoders return text (+language). Audar's *hosted* realtime API does LocalAgreement-2 word commits, but the open checkpoint exposes no timing. | inferred from cards/APIs |

Accuracy reality (inferred, consistent across sources): Whisper-family word times come from cross-attention DTW, typically good to a few hundred ms on clean speech, degrading on fast speech, music, and code-switched spans. WhisperX's own abstract calls out that Whisper's utterance timestamps are "prone to inaccuracies" and that word times are "not available out-of-the-box" (https://arxiv.org/abs/2303.00747). For Arabic specifically, DTW operates on BPE tokens, so a token can be a prefix/clitic (`و`, `ال`) — merged into words by whitespace heuristics, which is where ±100–300 ms jitter shows up. **Vosk** word times are acoustic-model-aligned (no DTW guesswork) and monotonic, but its Arabic WER is worse.

---

## 4. Script + audio → forced alignment

| Tool | Arabic? | Notes |
|---|---|---|
| **stable-ts `align()`** | **Yes, language-agnostic** | `model.align('a.wav', script_text, language='ar')` then `align_words`. Uses Whisper itself as the acoustic model, so it works for any Whisper language. Requires the *text to match what Whisper would emit*: undiacritised, hamza-normalised, no tatweel. Diacritised or heavily MSA-ised scripts vs dialectal audio will drop words. **verified for API/CLI, Arabic quality inferred** (https://github.com/jianfch/stable-ts#alignment) |
| **WhisperX** | Arabic align model present | But its `align()` consumes ASR *segments* first, so it is a transcribe-then-align pipeline, not pure script→audio. Arabic wav2vec2 model is a char-CTC trained on Common Voice — decent for MSA, shaky on dialect. |
| **MFA (Montreal Forced Aligner)** | **Dictionary only — no acoustic model** | mfa-models docs list `Arabic/Arabic MFA dictionary v2.0.0` and a G2P, but the acoustic-model index has **no Arabic entry** (languages go Abkhaz…Vietnamese, no Arabic). You would have to train an acoustic model yourself. **verified** (https://mfa-models.readthedocs.io/en/latest/acoustic/index.html, https://mfa-models.readthedocs.io/en/latest/dictionary/index.html) |
| **aeneas** | ARA confirmed | eSpeak/eSpeak-ng TTS + MFCC/DTW. Listed in "38 languages confirmed working: … ARA …" (https://github.com/readbeyond/aeneas). Quality ceiling: unvocalised Arabic read by eSpeak is approximate → good for **sentence/paragraph** sync, weak at word level. **verified support, inferred quality** |
| **torchaudio `forced_align`** | via MMS/wav2vec2 | CTC forced alignment with any character/phoneme CTC model; Arabic wav2vec2 works, but needs romanisation or an Arabic vocab — DIY. |

Practical recommendation: if the script is user-supplied and trustworthy, **stable-ts `align(language="ar")`** is the only low-friction word-level option; if it's approximate, align at phrase level with aeneas and snap to Whisper segment boundaries.

---

## 5. Diacritics, punctuation, numbers, code-switching

Verified facts:
- The Arabic leaderboard **normaliser** (https://github.com/Natural-Language-Processing-Elm/open_universal_arabic_asr_leaderboard/blob/main/eval.py) strips punctuation `!"#$%…،؛؟`, strips harakat `\u064B-\u0652`, maps **Eastern Arabic numerals → Western**, unifies `أإآ→ا`, `ؤ→و`, `ئ→ي`, drops bare hamza, and merges proclitic `و`. That normaliser exists because models disagree on all of those.
- **Whisper large-v3** emits **undiacritised** Arabic, **inconsistent punctuation** (often none, sometimes `.`/`،`), and **normalises dialect toward MSA** rather than transcribing literally. Concrete side-by-side from Cohere's launch post: Gulf "انا ابغي اقدم طلب annual leave من نظام hris…" → Whisper "انا اريد ان اقدم طلب … ولكن بصراحة لا املك خبرة …" (WER 51.2 vs 5.9), and Whisper keeps English terms in Latin script (https://cohere.com/blog/transcribe-arabic, 2026-07-07). Independent practitioner write-up says the same: "Punctuation and diacritics are unreliable… Numbers and proper nouns are weak" (https://omarbadran.dev/blog/whisper-speech-to-text/).
- **Jais-ASR** exposes `diacritics=True` (adds harakat) and `code_switch=True` (keeps English/French/Spanish/Italian spans in Latin script), plus a dialect label per utterance — the only open model with an explicit diacritisation switch (**verified**, model card).
- **Audar** sells "dialect-faithful orthography" and lists diacritic correctness / code-switch handling as KTO alignment axes; Emirati card notes residual WER is largely orthographic convention (انتو↔انتوا, Latin vs Arabic loanwords), not misrecognition.
- **Cohere Transcribe Arabic** preserves Latin-script technical terms verbatim (verified in blog examples).

Inferred: expect **Western digits** from Whisper/Jais for MSA, Arabic-Indic digits from some dialectal models; mixed-script output is common and should be preserved rather than transliterated. For subtitles you will need your own punctuation/diacritics post-pass (e.g. an Arabic text restoration model) — no ASR in this list gives reliable comma placement.

---

## 6. Install footprint and throughput

### Model sizes (on disk)
| Model | Params | Quantised | Full | RAM/VRAM working set |
|---|---|---|---|---|
| whisper tiny/base/small/medium | 39M–769M | — | 75 / 142 / 466 MiB / 1.5 GiB (ggml f16) | 0.5–2.5 GB |
| whisper large-v3 | 1.55B | **1.1 GiB (q5_0)** | 2.9 GiB (f16) | ~2–4 GB CPU, ~3–4 GB VRAM int8 |
| whisper large-v3-turbo | 809M | 547 MiB (q5_0) | 1.5 GiB | ~1.5–2.5 GB |
| Audar Turbo | 2.35B | 1.28 GB (Q4_K_M) / 2.16 GB (Q8) + 0.64 GB mmproj | 4.7 GB | ~2.5–5 GB |
| Audar Flash | 0.78B | Q4/Q8 GGUF | ~1.6 GB | ~1–2 GB |
| Cohere Transcribe Arabic | 2.07B | — | ~4.2 GB bf16 | ≥8 GB VRAM comfortable |
| Jais-ASR | ~6B | — | **11.4 GB bf16** | **~12 GB VRAM** |
| Vosk ar-mgb2-0.4 | — | — | **318 MB** | ~0.5–1 GB |
| Vosk ar-0.22-linto | — | — | 1.3 GB | ~2 GB |

### Throughput (all measured on **13 min** of audio)
From faster-whisper's own benchmark table (https://github.com/SYSTRAN/faster-whisper#benchmark), 8 threads, i7-12700K / RTX 3070 Ti 8 GB:
| Setup | Time | ×real-time |
|---|---|---|
| whisper.cpp small, CPU fp32 | 2m05 | **6.2×** |
| faster-whisper small, CPU int8 | 1m42 | 7.6× |
| faster-whisper small, CPU int8 batch 8 | 51 s | **15×** |
| whisper.cpp large-v2, GPU fp16 + FA | 1m05 | **12×** |
| faster-whisper large-v2, GPU fp16 batch 8 | 17 s | **46×** |
| WhisperX large-v2, GPU (project claim) | — | **70×** |

Extrapolations (**inferred**): large-v3 on 8 CPU threads ≈ **0.7–1.3× real-time** (a 25-min file takes ~20–35 min); on a mid GPU ≈ 10–15× (~2 min); batched GPU + VAD ≈ 40–70× (~25 s). Audar-Turbo GGUF on CPU is roughly 1–2× real-time (2.35B decoder + audio tower; no published number). Cohere's RTFx 525 is server-GPU with vLLM; not comparable to single-file desktop.

### Node-only (no Python) paths — yes, but with a ceiling
- `whisper-cpp-node` (v0.2.12, prebuilt native addons, darwin-arm64/win32; GPU acceleration) — https://www.npmjs.com/package/whisper-cpp-node
- `@spader/node-whisper-cpp` (+ per-platform prebuilds incl. `x64-linux-cuda-gnu`, `x64-linux-cpu-gnu`) — https://www.npmjs.com/package/@spader/node-whisper-cpp
- `nodejs-whisper`, `whisper-cpp-static`, `@remotion/install-whisper-cpp` (spawn the binary + model downloader)
- `sherpa-onnx-node` v2.0.0 (native addon; Whisper ONNX, Silero VAD, token timestamps) — https://www.npmjs.com/package/sherpa-onnx-node
- `vosk` v0.3.39 (official Node binding; word-level `result[]`) — https://www.npmjs.com/package/vosk
- **No** Node binding for Jais/Cohere; the only Node-reachable path for **Audar** is `llama-server` (llama.cpp mtmd, OpenAI-style endpoint) called over HTTP — verified that Audar ships GGUF + mmproj and documents `llama-mtmd-cli`, so a `llama-server` HTTP call is a straightforward Node integration (inferred).

Nothing Node-native does forced alignment. Alignment (stable-ts/WhisperX/MFA/aeneas) is Python-only.

---

## 7. Long audio (e.g. 25 minutes)

Whisper-family models are **30 s window** models; everything else is chunking policy.

- **Failure mode is documented, not hypothetical**: WhisperX paper — "their application to long audio transcription via buffered or sliding window approaches is **prone to drifting, hallucination & repetition**" (https://arxiv.org/abs/2303.00747).
- **whisper.cpp**: internal seek over 30 s windows with `--max-context`; **Silero VAD built in** (`--vad -vm silero_vad.onnx`, plus `-vspd/-vsd/-vmsd/-vp` tuning) — verified in `examples/cli/cli.cpp:107-114,216-222`; `-ot/-on` for offsets/resume. Timestamps are absolute and generally stable, but silence/music triggers hallucinated segments unless VAD is on.
- **faster-whisper**: `vad_filter=True` (default only for the batched pipeline); `condition_on_previous_text` defaults to True → repetition loops on noisy long files; batched pipeline + VAD is the recommended 25-min path.
- **WhisperX**: VAD cut → merge with `chunk_size=30` → batched decode → phoneme alignment. This is the design that fixes both drift and hallucination. Best long-form accuracy/timestamp combo.
- **Audar/Jais**: 30 s encoder window; Audar card says "chunk longer inputs client-side" and admits "Long-form audio can drift on very long recordings" (verified). Jais needs `max_new_tokens` raised per chunk and has a `truncated=True` flag → you must chunk around 30 s yourself and stitch text (no timing to stitch, so you lose absolute time unless you keep chunk offsets).
- **Vosk**: streaming decoder, no window and no drift; timestamps monotonic. Accuracy is the weak point.
- **Cohere/vLLM**: server-side batching, no public long-form guidance beyond "optimised for varying input lengths" (unknown).

Practical recipe for 25 min: Silero VAD → cut at silence with 0.3–0.5 s pad → decode each cut with absolute offset → concatenate; never rely on a single 25-min pass without VAD.

---

## 8. Most viable setups for an Arabic-first local pipeline

### A. whisper.cpp + Silero VAD + DTW word timestamps (Node-friendly, single binary)
`whisper-cli --vad -vm silero_vad.onnx -ml 1 --dtw large.v3` or `whisper-server` `response_format=verbose_json` → `segments[].words[]`.
- **Pros**: no Python, 1.1 GiB q5 model, real word timestamps, runs anywhere, easy to bundle (`whisper-cpp-node`/`@spader/node-whisper-cpp`).
- **Cons**: Arabic is its weakest area — 55.96 WER on Gulf, 71.81 on Maghrebi, only ~17.8 on MSA read. Normalises dialect toward MSA. OCR-grade diacritics: none.
- **Use when**: MSA/lecture/broadcast material, or when shipping a zero-dependency desktop binary matters more than dialect accuracy.

### B. Python: faster-whisper large-v3 + WhisperX (VAD + wav2vec2-ar alignment)
`whisperx --model large-v3 --language ar --align_model jonatasgrosman/wav2vec2-large-xlsr-53-arabic`.
- **Pros**: best timestamp accuracy available (phoneme forced alignment), 40–70× realtime on GPU, handles 25-min files cleanly, speaker diarization available.
- **Cons**: Python + torch + pyannote, ~6–8 GB VRAM, and the accuracy ceiling is still Whisper's Arabic (same 56% Gulf problem). The Arabic align model is CommonVoice-trained → MSA-leaning.

### C. Arabic-first two-pass: Audar-Turbo (or Jais-ASR) transcript → stable-ts align for timing
Chunk at 30 s with Audar Q8/llama.cpp (CPU-viable, 2.35B, 28.9 Gulf WER) or Jais-ASR (Apache, 12 GB GPU, 30.5 Gulf WER, optional diacritics), then feed the **resulting text + original audio** into stable-ts `align(language="ar")` using whisper large-v3-turbo, then re-project word times onto the Audar text.
- **Pros**: ~25 WER points better on Gulf than any Whisper path; proper Arabic orthography, code-switching, and (Jais) optional diacritics; still fully local.
- **Cons**: two models, ~2× cost, alignment accuracy limited to what Whisper can hear (on dialectal audio the aligner is the weak link); Jais needs 12 GB VRAM and has no per-word output of its own; Audar's licence is not OSI-permissive.

**Ranking for an Arabic-first local pipeline**: C for transcript quality, B for timing quality, A for deployability. If you must pick one: **B** if a GPU and Python are acceptable; **A** only if the corpus is MSA-heavy; **C** if dialect accuracy is the product.

---

## Open questions / gaps
1. No public **Levantine-only** zero-shot WER for the current model set (leaderboard aggregates only). Needs an in-house eval set.
2. No published CPU throughput for Audar-Turbo/Jais — must measure locally.
3. Whether Cohere Transcribe Arabic emits timestamps at all (gated card, no API mention) — **unknown**.
4. Diacritisation quality of Jais `diacritics=True` is unbenchmarked; the leaderboard strips diacritics so it is invisible to every published number.
