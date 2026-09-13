# Speak — Offline Conversation Partner for Any Language (Android)

Personal, offline, no accounts, no cloud. The app listens to you speak, thinks
with a local LLM, and answers out loud — target latency: you finish talking →
first spoken word back ≤ 2s (≤ 1s on a good day).

Core principles:
- **Tiny APK (~20 MB):** ships zero model files, only a small language
  catalog (JSON asset). Models are downloaded at runtime, per language.
- **Languages screen = switches for every language sherpa-onnx supports.**
  Toggling one ON downloads that language's ASR + TTS files (size + progress
  shown); toggling OFF deletes them.
- **Active-language selector:** picked from installed languages; the choice
  switches ASR + TTS config + tutor prompt, including mid-session.
- EN and JA are curated (best models, tested); other languages ship as
  catalog data.

Target device: Xiaomi Poco X7 Pro (Dimensity 8400 Ultimate, 12 GB RAM, Mali-G720).
Min SDK: 26. Everything runs on-device.

## 1. Architecture

```
 Mic (AudioRecord 16kHz mono, VOICE_COMMUNICATION + AEC)
   └─► Silero VAD ──► end-of-speech detector ──► ASR (sherpa-onnx, streaming)
            │                                         └─► transcript text
            ▼                                              ▼
        UI state machine ──► LLM (Gemma 3 1B, LiteRT-LM)  ──► token stream
                                                        └─► sentence splitter
                                                              └─► TTS (sherpa-onnx Kokoro)
                                                                    └─► AudioTrack
 Barge-in: mic stays hot during TTS (with AEC) so you can interrupt.
```

One-way pipeline, all components replaceable behind 3 interfaces:
`SttEngine`, `LlmEngine`, `TtsEngine`.

## 2. Component choices

| Piece | Choice | Why / Alternatives |
|---|---|---|
| VAD | Silero VAD via sherpa-onnx (`silero_vad.onnx`) | tiny, proven; enables hands-free turn detection |
| STT (M1) | sherpa-onnx streaming Zipformer English (`sherpa-onnx-streaming-zipformer-en-20M-2023-02-17`) | true streaming partial results = lowest perceived latency |
| STT (M3 upgrade) | Whisper `base.en`/`small.en` int8 (two-pass) or NVIDIA Parakeet-TDT-0.6B int8 (`sherpa-onnx-nemo-parakeet-tdt-0.6b-v2-int8`) | accuracy bump once UX works |
| STT (Japanese, M5) | `sherpa-onnx-zipformer-ja-reazonspeech-2024-08-01` (non-streaming) or multilingual Whisper `small` | ReazonSpeech = best JA in sherpa-onnx; Whisper covers JA in two-pass |
| LLM | Google **LiteRT-LM** (MediaPipe LLM Inference API, `com.google.mediapipe:tasks-genai` / `com.google.ai.edge.litertlm`) running **Gemma 3 1B IT, Q4** (~0.9–1.2 GB) GPU delegate | only first-party option that cleanly streams tokens on Dimensity/Mali. Fallback: llama.cpp JNI (learn from ChatterUI's `cui-llama.rn`, but stay Kotlin). |
| LLM (Japanese, M5) | same Gemma 3 1B (multilingual, incl. JA); if JA quality is weak → **Qwen2.5-1.5B/3B IT Q4** (excellent JA, runs on same stack) | swap-in via `LlmEngine` |
| TTS | sherpa-onnx **Kokoro-82M** English voice (best quality at real-time on this SoC). Fallback Piper/`amy` if disk-constrained | near-native accent matters for a learning app |
| TTS (Japanese, M5) | Kokoro JA voices (`jf-*`/`jm-*`, multi-lang Kokoro build in sherpa-onnx); fallback Piper `jp-*` (needs unidic via MeCab — heavier packaging) | Kokoro JA is already integrated in sherpa-onnx |
| App | Kotlin + Jetpack Compose, Coroutines/Flow, single `:app` module, `dev.tienvx.speak` | |

Libs integrated as: sherpa-onnx prebuilt **AAR from GitHub releases** (contains
Kotlin API `com.k2fsa.sherpa.onnx` + `.so` for arm64-v8a; verify exact Gradle
coordinate — may be on Maven/JitPack — in M0) + MediaPipe/LiteRT-LM artifact.
We do NOT fork/modify sherpa-onnx source.

## 3. Models — tiny APK + data-driven language packs

### APK size strategy

- No models, no phoneme dictionaries in assets — only `assets/languages.json`
  (<10 KB). sherpa-onnx AAR restricted to **arm64-v8a only**, R8 minified,
  AAB build. Target APK ≤ ~25 MB (its native `.so` dominates).
- All model files under `<filesDir>/models/<lang>/…` — uninstall a language =
  delete its folder. Shared files (Silero VAD, espeak-ng data, a multilingual
  Whisper) stored once under `models/shared/` and referenced by ID.
- Disk usage is the user's choice: each language reports its size before
  download; total-usage banner on the Languages screen.

### Language catalog

`assets/languages.json` entry:

```json
{
  "id": "de", "name": "Deutsch", "flag": "🇩🇪",
  "tts":  { "engine": "piper", "package": "piper-de", "files": ["…"],
            "url": "…", "sha256": "…", "sizeMB": 60, "quality": "good" },
  "asr":  [ { "engine": "whisper", "package": "whisper-small", "kind": "multilingual",
              "url": "…", "sha256": "…", "sizeMB": 110, "streaming": false, "quality": "good" },
            { "engine": "zipformer-streaming", "package": "…fr…", "url": "…",
              "sizeMB": 100, "streaming": true, "quality": "best" } ],
  "shared": ["silero-vad"], "g2p": ["espeak-ng-data"],
  "levelScale": "cefr"
}
```

Downloaded via plain OkHttp + sha256 verify + atomic rename; multiple `asr`
options = pick-best-in-list at engine build time (streaming preferred when
hands-free, quality-preferred for corrections screen).

### Coverage (built by a script in M0)

- **ASR broad:** multilingual **Whisper** (tiny→large, ~99 languages — every
  language gets at least this), **SenseVoice** (zh/en/ja/ko/yue + zh dialects,
  fast), **Paraformer/Dolphin** (zh-centric), per-language Zipformers incl.
  streaming (en, fr, ko, ru, th, ja via ReazonSpeech…), **Parakeet** (en),
  KWS for wake word later.
- **TTS broad:** **Piper** (~50 languages, quality "good/mediocre", G2P
  dictionaries per voice), **Kokoro-82M** (en/ja/zh — quality "very good"),
  **Matcha** (zh, zh+en), **ZipVoice** (zh/en voice cloning later).
- **LLM:** Gemma 3 1B is multilingual for all catalogued languages; per
  override to Qwen2.5 IT if a language is weak (config field `llm: qwen`).
- Languages with ASR but no TTS (e.g. some minority languages) appear in the
  catalog with `"tts": null` → UI shows "listen-only (text replies)" — the
  switch still works (ASR downloads), replies shown as text + optional
  transliteration.
- **Fill procedure:** `tools/build_catalog.py` scrapes the sherpa-onnx release
  index + Piper voice list, emits `languages.json` with real URLs/sizes/
  checksums (`sha256sum`), plus a `catalog_check` script that HEAD-checks every
  URL. Curated entries (en, ja) are hand-picked and hand-tested.

### Curated sizes (reference, downloaded on toggle, not in APK)

| Language | ASR | TTS | ~Total |
|---|---|---|---|
| en | streaming Zipformer (~100 MB) | Kokoro-82M (~330 MB) | ~430 MB |
| ja | ReazonSpeech (~200 MB) | Kokoro JA voice (~40 MB) | ~240 MB |
| any Whisper-only lang | Whisper `small` multilingual (~110 MB, shared) | Piper voice (~30–60 MB) | ~140–170 MB |

`models/shared/` counts once toward the total no matter how many languages are
enabled. App needs ≥1 complete language pack (its ASR + (TTS or text-fallback))
+ LLM to function; LLM (Gemma Q4 ~1 GB) is one shared download, separate
switch.

## 4. Conversation design (language-practice specific)

- **System prompt = tutor persona**, two modes selectable in UI:
  1. *Chat mode*: natural conversation, stays in topic, level-adjusts vocabulary.
  2. *Correction mode*: reply + short `Fixes:` block of your grammar/word errors
     (spoken only if it is short; shown as text cards).
- Roles/scene presets (ordering coffee, job interview, small talk…) as
  starter prompts.
- Conversation state kept in memory + persisted to app DB (no cloud).
- Difficulty setting "CEFR B1/B2/C1" injected into prompt.

### 4a. Japanese mode (M5)

- Same architecture; the language switch changes 3 things only: ASR model
  config, TTS voice, and tutor prompt — the `SttEngine`/`TtsEngine` interfaces
  are language-agnostic.
- JA ASR: ReazonSpeech Zipformer (non-streaming) — turn latency comes from
  VAD end-of-speech + one decode pass (~RTF 1 on arm64). Streaming partials
  for JA are not available in sherpa-onnx; accept the extra ~0.5s or use
  multilingual Whisper two-pass.
- JA TTS: Kokoro Japanese voices — note Kokoro's JA quality is good but not
  NHK-level; no accent feedback in v1 (text correction only).
- JA text pipeline: TTS side needs Japanese text → kana/kanji handling is done
  by Kokoro's built-in G2P (Jieba/Mecab bundled by sherpa-onnx config).
- Sentence splitter must switch delimiters: `.!? ` (EN) vs. `。！？、` (JA) —
  no space-based chunking in JA.
- LLM: Gemma 3 handles JA; benchmark M5 first — if weak, swap to Qwen2.5 IT
  for JA sessions (better at keigo, kanji).
- Prompt modes mirror EN: chat / correction. Correction mode adds
  furigana/romaji annotation under the corrected sentence (opt-in).
- Level setting for JA: N5–N1 injected into prompt instead of CEFR.

## 5. Latency budget (you stop talking → you hear reply)

| Step | Budget |
|---|---|
| end-of-speech detection | ~300 ms trailing silence |
| final ASR (streaming, mostly already done) | <100 ms |
| final ASR, JA (non-streaming decode after VAD) | +300–600 ms |
| LLM first token | 100–400 ms (prompt cache; keep model loaded in RAM) |
| first sentence decoded → Kokoro first audio | ~150–300 ms |
| **target total** | **~1–1.5 s** |

Rules: LLM streams token-by-token; sentence splitter flushes TTS at
`.!?`; TTS starts on sentence #1 while LLM still generating; keep LLM warm
(no preload per turn), keep ASR+TTS engine singletons; `PARTIAL_WAKE_LOCK` not
needed but keep screen-on during active call (FLAG_KEEP_SCREEN_ON) to dodge
thermal/idle stalls. Watch thermal throttling: 8-core budget on Dimensity.
Sentence-splitter delimiters come from the catalog entry per language
(`.!? ` latin, `。！？` ja/zh, none for Thai/Lao → flush on pause/word
count). Non-spaced scripts (CJK) never chunk on spaces.

## 6. App shape

```
speak/
  PLAN.md            (this file)
  app/               Android module (Compose)
  ...standard Gradle wrapper (create-android-project scaffold in M0)
```

Key classes:
- `MainActivity` → `SpeakApp` (Compose nav: Home / Conversation /
  **Languages** / Settings)
- `SpeechOrchestrator` — state machine: `IDLE → LISTENING → THINKING → SPEAKING
  (→ LISTENING)`. Barge-in: in SPEAKING, if VAD detects user speech for >X ms,
  stop TTS immediately and re-enter LISTENING.
- `AudioService` — single AudioRecord loop feeding VAD and ASR; AEC via
  `AudioEffect.ACOUSTIC_ECHO_CANCEL` on `VOICE_COMMUNICATION` (must bind
  mic/speaker into same audio session; else AI hears itself — worst bug here).
- `LlmService` — owns LiteRT-LM native handle; chat history; streaming tokens
  → `Flow<String>` chunks.
- `TtsService` — builds engines from the active language's catalog entry
  (Kokoro or Piper) → `Flow<AudioFrame>` per sentence.
- `LanguageManager` — parses `assets/languages.json`; installs/uninstalls
  language packs (download, sha256, atomic move, shared-store dedup), reports
  sizes/progress, exposes `installed: List<LanguageEntry>` state for UI.
- `SessionStore` — chat persistence (per language tag, stored with each turn).

UI modes:
- M1: push-to-talk hold button (walkie-talkie) — simplest, no AEC needed.
- M2+: hands-free with VAD + barge-in.
- **Languages screen**: one row per catalogued language — switch (on =
  download with size + progress; off = delete), `tts: null` languages show
  "text replies only", shared/total disk usage banner.
- **Active-language selector** (dropdown of installed languages, default `en`)
  on Home + in the conversation screen; switching mid-session is allowed
  (next reply uses the new language).

Permissions: `RECORD_AUDIO` only (plus `POST_NOTIFICATION` if foreground service
later; not for v1). No INTERNET permission except model download uses
`OkHttp` — fine to include; make a "download models" screen.

## 7. Milestones

- **M0 — Scaffold (½–1 day):** Gradle/AGP/Compose project, arm64-v8a AAR deps
  wired, model download screen. `tools/build_catalog.py` + `catalog_check`
  scrape the sherpa-onnx release index into `assets/languages.json` (all
  languages, real URLs/sizes/sha256). Accept: APK ≤ ~25 MB, builds and runs
  on the phone; `catalog_check` passes on the machine.
- **M1 — Voice loop, push-to-talk (1–2 days):** curated `en` pack as data in
  the catalog; hold mic → streaming STT (show partial text) → release → LLM
  answer → Kokoro reads it. Accept: full round-trip works offline (airplane
  mode); en toggle on/off downloads/deletes the pack correctly.
- **M2 — LLM real + prompt (1–2 days):** LiteRT-LM/Gemma 3 1B on GPU verified
  on Dimensity (fallback CPU/OpenCL path + token/s bench recorded). Tutor and
  correction prompts (per-language template, level scale field from catalog).
  Accept: first-token ≤ ~500 ms typical, streamed speech starts before LLM
  finishes.
- **M3 — Hands-free + barge-in (2–3 days):** VAD turn detection, AEC verified
  no self-hearing, interrupt handling. Accept: two full conversations without
  touching screen; interrupt works.
- **M4 — Learning polish (ongoing):** scene/role presets, CEFR level, correction
  cards, session history, accuracy upgrade of ASR if needed.
- **M5 — Languages screen + Japanese (2–3 days):** Languages screen with
  switches (download/delete/progress/disk banner, `tts:null` handling),
  shared-store dedup, JA pack from catalog (ReazonSpeech + Kokoro JA, N-level
  scale, JA sentence splitter, Gemma-vs-Qwen benchmark). Accept: install
  `de` (Piper+Whisper) and `ja` by toggling; JA round-trip OK (+0.5 s latency); 50-sentence kanji/kana TTS test.
- **M6 — Broad expansion (opportunistic):** any language whose entry passes
  `catalog_check` gets a switch; test one representative per TTS engine
  (Piper/MATCHA) before marking quality "good" in catalog.

## 8. Key risks

| Risk | Mitigation |
|---|---|
| LiteRT-LM GPU perf on Mali/OpenCL unknown on this SoC | benchmark in M2; fallback = llama.cpp CPU 8+4 cores (Qwen2.5-1.5B/Q3 as lighter model) |
| Echo cancellation (AI hears own TTS) | VOICE_COMMUNICATION session + platform AEC; worst case: mute mic while TTS, no barge-in |
| sherpa-onnx Kotlin API integration details (AAR coordinate, model file lists) | M0 spike: compile against 3 of its own Android demo apps as reference (Apache-2.0, copy allowed) |
| Kokoro quality vs. disk | Piper ~60 MB fallback |
| Thermal throttling in long sessions | keep 2 of 8 cores idle for TTS+ASR, cap session length, watch CPU-freq |
| 12 GB RAM but Gemma 1B + Kokoro + ASR together resident | all fit comfortably (~2 GB RSS); still allow killing LLM under memory pressure |
| JA kanji G2P in Kokoro (needs Jieba/Mecab dict files shipped with the kokoro_multi release) | test in M5 spike; dict files come with the same model tarball |
| Gemma 1B JA quality (weak keigo/dialect handling) | M5 benchmark → Qwen2.5 IT swap for JA sessions |
| ReazonSpeech only non-streaming → JA turn latency | VAD-driven decode; fine for practice, not for interrupting |
| Language coverage is uneven (Whisper ≈99 langs but mediocre on distant ones; Piper quality "OK" at best; streaming ASR exists for only ~a dozen langs) | `quality` flag per catalog entry drives UI badges; only curated en/ja get "best"; others labeled good/mediocre/listen-only — never silently promised equal quality |
| Disk bloat when many languages enabled | size shown before download, total-usage banner, one-tap delete; shared Whisper/VAD files deduped |
| Catalog URLs/models go stale upstream | `catalog_check` script re-verifies URLs + checksums; catalog regenerated by script |

## 9. Reference projects

- `k2-fsa/sherpa-onnx` Android demos (ASR/TTS/VAD APK sources) — Apache-2.0
- `Vali-98/ChatterUI` — llama.cpp-on-Android patterns (AGPL; read, don't fork)
- `Open-LLM-VTuber/Open-LLM-VTuber` — hands-free/barge-in interaction design
- sherpa-onnx docs: k2-fsa.github.io/sherpa/onnx (Kotlin/Java API, model lists)

## 10. Non-goals (v1)

No publish-store compliance, no iOS, no wake word (later maybe keyword spotting
with KWS model), no cloud fallback, no user accounts. Personal use only.
