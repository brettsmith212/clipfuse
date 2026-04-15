# Implementation Plan — clipfuse Phase 0

This plan covers Phase 0 (signal discovery) from the project plan. Each milestone builds on the previous one and ends with passing tests. Work bottom-up: models → source → transcribe → signals → fusion → select → clip → assemble.

---

## Milestone 0: Test infrastructure & audio fixtures

**Goal:** Be able to run `uv run pytest` and get green from day one. Establish the fixtures every later test depends on.

### Tasks
- [ ] `tests/conftest.py` — shared pytest fixtures
- [ ] `scripts/download_test_audio.py` — downloads a short podcast clip (~5–10 min) via `yt-dlp` or direct URL to `tests/fixtures/`. Gitignored but documented.
- [ ] `tests/fixtures/` directory with:
  - `short_clip.mp3` — real audio, ~5 min, gitignored (downloaded by script)
  - Synthetic fixtures generated in conftest: sine wave, silence, noise burst, speech-like audio (for unit tests that must run without real audio or network)
- [ ] `tests/fixtures/.gitkeep` + `.gitignore` entry for `*.mp3`, `*.wav` in fixtures
- [ ] Add `yt-dlp` download helper function in `scripts/download_test_audio.py`:
  ```python
  # Usage: uv run python scripts/download_test_audio.py [youtube_url]
  # Default: downloads a short, freely available podcast clip
  # Output: tests/fixtures/short_clip.mp3
  ```

### Tests
- `test_fixtures.py` — assert synthetic fixtures are generated correctly (duration, sample rate, non-silent)

---

## Milestone 1: Core data models (`models.py`)

**Goal:** All dataclasses from the project plan implemented, importable, and tested.

### Tasks
- [ ] Implement `Word`, `Segment`, `Transcript`, `AudioSource`, `SignalSet`, `CandidateRegion`, `Highlight`, `Clip`, `Compilation`
- [ ] `Transcript.text()`, `Transcript.words()`, `Transcript.slice(start, end)` methods
- [ ] `SignalSet.coverage` property

### Tests (`tests/test_models.py`)
- Construct each dataclass, verify frozen immutability
- `Transcript.text()` concatenates segment text
- `Transcript.words()` flattens segment words
- `Transcript.slice()` returns correct sub-transcript
- `SignalSet.coverage` math is correct (0/0 → 0.0, 3/5 → 0.6)

---

## Milestone 2: Audio source fetching (`source.py`)

**Goal:** `fetch_audio(url) -> Path` downloads audio to a temp/cache dir. Also support local file paths (passthrough).

### Tasks
- [ ] `fetch_audio(url_or_path, cache_dir=None) -> Path`
  - HTTPS URLs: stream to disk with `urllib.request` (stdlib only, no `requests`)
  - Local paths: return as-is after validation
  - Cache: hash URL → filename, skip if exists
- [ ] `make_source_id(url) -> str` — stable hash for `AudioSource.id`

### Tests (`tests/test_source.py`)
- Fetch a local file (fixture) → returns same path
- Fetch from URL → file lands on disk, is valid audio (use a small public domain mp3 URL or mock)
- Cache hit → no re-download
- `make_source_id` is deterministic

---

## Milestone 3: Transcription (`transcribe.py`)

**Goal:** `transcribe(audio_path, backend="whisper-api") -> Transcript`. One real backend (Whisper API) + one fake backend for testing.

### Tasks
- [ ] `transcribe(audio_path: Path, backend: str = "whisper-api") -> Transcript`
- [ ] `_transcribe_whisper_api(audio_path) -> Transcript` — calls OpenAI Whisper API with word-level timestamps
- [ ] `_transcribe_fake(audio_path) -> Transcript` — returns a hardcoded/deterministic transcript for testing (no API calls)
- [ ] Parse Whisper API response into `Segment` / `Word` objects

### Tests (`tests/test_transcribe.py`)
- Fake backend returns valid `Transcript` with correct structure
- Whisper API backend (integration test, marked `@pytest.mark.integration`):
  - Transcribes `short_clip.mp3` fixture
  - Returns segments with timestamps
  - Word-level timestamps are present
  - Segments are non-overlapping and monotonically increasing

---

## Milestone 4: Signal collector base + registry (`signals/base.py`, `signals/__init__.py`)

**Goal:** The `SignalCollector` protocol and `gather_signals()` orchestrator work. Registering a custom collector and having it called automatically.

### Tasks
- [ ] `SignalCollector` protocol/ABC in `signals/base.py`:
  ```python
  class SignalCollector(Protocol):
      name: str
      def collect(self, transcript, audio_path=None, metadata=None) -> list[SignalResult]: ...
  ```
- [ ] `SignalResult` dataclass: `(start, end, score, details)`
- [ ] `signals/__init__.py`: `register_collector()`, `gather_signals()` orchestrator
- [ ] `gather_signals(transcript, audio_path=None, metadata=None) -> list[CandidateRegion]`
  - Runs all registered collectors
  - Merges overlapping results into `CandidateRegion` with `SignalSet`

### Tests (`tests/test_signals_base.py`)
- Register a dummy collector, call `gather_signals()`, verify it runs
- Multiple collectors merge into `SignalSet` correctly
- Collector that returns empty list → no crash
- Region overlap merging works

---

## Milestone 5: Text signals (`signals/text.py`)

**Goal:** First real signal collectors — purely text-based, no API calls, no audio needed.

### Tasks
- [ ] `HostReactionCollector` — detects exclamatory phrases ("wow", "that's incredible", etc.)
- [ ] `StructuralPatternCollector` — Q&A density, speaker-change rate, monologue length
- [ ] `LinguisticMarkerCollector` — strong-claim language, narrative markers, specificity
- [ ] `AdDetectionCollector` — sponsor reads, intro/outro boilerplate → suppression flag

### Tests (`tests/test_signals_text.py`)
- Host reactions: feed transcript with "wow, that's incredible" → scored region around it
- Host reactions: feed boring transcript → no detections
- Structural: long monologue → flagged
- Structural: rapid Q&A exchange → flagged
- Ad detection: "brought to you by" → suppressed
- Ad detection: normal content → not suppressed
- Each collector returns properly normalized 0.0–1.0 scores

---

## Milestone 6: Audio signals (`signals/audio.py`)

**Goal:** Energy, speech rate, and laughter signals from audio + transcript.

### Tasks
- [ ] `EnergyEnvelopeCollector` — RMS loudness in 1s windows, normalized per-episode
- [ ] `SpeechRateCollector` — WPM from word timestamps, 30s smoothing window
- [ ] `LaughterDetectionCollector` — simple frequency-based or stub (placeholder for YAMNet)
- [ ] `MusicDetectionCollector` — non-speech segment detection for suppression

### Tests (`tests/test_signals_audio.py`)
- Energy: synthetic loud burst → high score at that region
- Energy: constant volume → uniform low scores
- Speech rate: construct transcript with varying word density → deceleration detected
- Laughter: at minimum, stub returns empty list gracefully
- Music: synthetic tone → detected as non-speech

---

## Milestone 7: Signal fusion (`signals/fusion.py`)

**Goal:** `fuse_and_rank()` combines signal sets into ranked highlights.

### Tasks
- [ ] `DEFAULT_WEIGHTS: dict[str, float]` — initial uniform weights
- [ ] `fuse_and_rank(candidates, weights=None) -> list[Highlight]`
  - Weighted average over available signals (missing excluded, not zeroed)
  - Confidence = fraction of signals that agreed
  - Rationale string generation
  - Suppressed regions excluded
- [ ] Sort by fused_score descending

### Tests (`tests/test_fusion.py`)
- Two candidates with different signal coverage → ranked correctly
- Missing signal excluded from average (not treated as 0)
- Suppressed region excluded from results regardless of other scores
- Custom weights override defaults
- Confidence computation is correct
- Rationale string includes contributing signal names

---

## Milestone 8: Selection (`select.py`)

**Goal:** `knapsack_select()` picks highlights within a time budget.

### Tasks
- [ ] `knapsack_select(highlights, budget_seconds, constraints=None) -> list`
  - Greedy fallback (no `pulp` dependency required)
  - `min_per_source`, `max_per_source_pct` constraints
  - Dedupe by overlap threshold
- [ ] Ordering: best-first (default), chronological, interleaved

### Tests (`tests/test_select.py`)
- Budget constraint respected
- Higher-scored highlights preferred
- min_per_source forces at least one from each source
- max_per_source_pct limits single-source dominance
- Overlapping highlights deduplicated

---

## Milestone 9: Clip extraction & assembly (`clip.py`, `assemble.py`)

**Goal:** Cut audio segments and concatenate them. Requires `ffmpeg`.

### Tasks
- [ ] `extract_clip(source_path, highlight, transcript=None) -> Clip`
  - Uses ffmpeg to cut audio segment
  - Snaps boundaries to nearest sentence break via transcript
  - 50ms fade in/out
- [ ] `assemble_compilation(clips, output_path, add_intros=False) -> Compilation`
  - Concatenate clips with loudness normalization
  - Write ID3 chapter markers
- [ ] `snap_to_sentence_boundary(timestamp, transcript, direction) -> float`

### Tests (`tests/test_clip.py`, `tests/test_assemble.py`)
- Extract clip from fixture audio → output file exists, duration is correct (±0.5s)
- Sentence boundary snapping works
- Fade in/out applied (check first/last samples aren't at full volume)
- Assembly produces valid MP3 with correct total duration
- Chapter markers present in output metadata

---

## Milestone 10: End-to-end integration test

**Goal:** The 5-line hello world from the README actually works.

### Tasks
- [ ] Wire up `__init__.py` exports: `transcribe`, `gather_signals`, `fuse_and_rank`, `fetch_audio`, `knapsack_select`, `extract_clip`, `assemble_compilation`
- [ ] `examples/5_line_hello.py` runs successfully
- [ ] `examples/signals_only_no_llm.py` runs without API key

### Tests (`tests/test_integration.py`, marked `@pytest.mark.integration`)
- Full pipeline: audio file → transcribe (fake backend) → gather signals → fuse → rank → select → clip → assemble
- Output MP3 exists and is non-empty
- At least 1 highlight detected
- Full pipeline with real Whisper API (separate marker, `@pytest.mark.slow`)

---

## Milestone 11: Eval framework (`eval.py`)

**Goal:** Score pipeline output against hand-labeled episodes.

### Tasks
- [ ] YAML label format: `source_url`, `highlights`, `suppressions`, `genre`
- [ ] `load_labels(yaml_path) -> EvalLabels`
- [ ] `score_against_labels(highlights, labels) -> EvalResult`
  - Precision, recall, F1
  - Timestamp-overlap IoU metric
- [ ] `eval/schema.yaml` defining the label format
- [ ] Hand-label first 3 episodes (use the test fixture + 2 others)

### Tests (`tests/test_eval.py`)
- Perfect predictions → P=R=F1=1.0
- No predictions → P=0, R=0
- Partial overlap scored correctly via IoU
- YAML parsing round-trips correctly

---

## Audio acquisition helpers

These support development and testing throughout all milestones.

### `scripts/download_test_audio.py`

```
Usage:
  uv run python scripts/download_test_audio.py                    # default podcast clip
  uv run python scripts/download_test_audio.py <youtube_url>      # any YouTube video
  uv run python scripts/download_test_audio.py <podcast_mp3_url>  # direct mp3 URL

Output: tests/fixtures/short_clip.mp3 (or specified filename)

Requires: yt-dlp (for YouTube), ffmpeg
Install: uv pip install clipfuse[youtube]
```

### YouTube download (for development/testing only)

`yt-dlp` is already in `[youtube]` optional deps. Quick usage:
```bash
# Download audio only from a YouTube podcast clip
yt-dlp -x --audio-format mp3 -o "tests/fixtures/%(title)s.%(ext)s" "https://youtube.com/watch?v=..."

# Download first 5 minutes only (for smaller test fixtures)
yt-dlp -x --audio-format mp3 --download-sections "*0:00-5:00" -o "tests/fixtures/short_clip.mp3" "URL"
```

### Direct podcast MP3 download

Many podcast episodes have direct MP3 URLs in their RSS feeds. Grab one:
```bash
curl -L -o tests/fixtures/short_clip.mp3 "https://traffic.megaphone.fm/..."
```

---

## Build order summary

```
M0  Test infra & fixtures          ← do this first, everything depends on it
M1  models.py                      ← pure data, no deps
M2  source.py                      ← fetch audio (needed by everything below)
M3  transcribe.py                  ← transcript (needed by all signals)
M4  signals/base.py + registry     ← collector protocol
M5  signals/text.py                ├─ can parallel with M6
M6  signals/audio.py               ├─ can parallel with M5
M7  signals/fusion.py              ← depends on M4–M6
M8  select.py                      ← depends on M7
M9  clip.py + assemble.py          ← depends on M2, M8
M10 integration test               ← depends on all above
M11 eval framework                 ← can start after M7
```

Milestones 5 and 6 can be worked in parallel. Milestone 11 can start as soon as fusion works (M7).

---

## What's deferred (Phase 1+)

- LLM scoring (`signals/llm.py`) — implement after text+audio baseline is measured
- External signals (`signals/external.py`) — YouTube heatmap, chapters, etc.
- TTS intros (`tts.py`)
- Reference app (`poddigest`)
- MCP server
- Claude Skill
- LP solver for knapsack (greedy is fine for Phase 0)
