# Audio Highlight Extraction — Project Plan (v3)

*A composable building block for AI-powered long-form audio highlight extraction, with a podcast weekly-digest as the reference application.*

---

## 1. Project overview

### The problem

A typical podcast listener follows 5–15 shows. Episodes run 60–120 minutes each. That's 10–30 hours of new audio per week, far more than anyone has time for. Listeners end up either falling behind, cherry-picking episodes by guest or title (and missing the best moments in shows they skipped), or abandoning the medium entirely.

The same problem exists in adjacent domains: knowledge workers drowning in recorded meetings, students with hours of lecture footage, sales teams sitting on call recordings, researchers with conference talks they'll never get to. The underlying need is identical — *given long-form audio, find the parts worth my time.* The technical primitive is the same. Only the framing changes.

Existing tools are either creator-side (OpusClip, Descript, Choppity, SendShort — they help producers cut clips from their own content for social media) or consumer-side per-episode tools (Snipd, Podwise, Snipcast, Podsnacks — they summarize one episode at a time, which still leaves the listener doing the curation work). None of them deliver a *cross-source, time-budgeted, automatically-assembled* highlight reel. That's the gap this project targets.

### The hard problem at the center

The deepest challenge in this project is not transcription, not feed polling, not audio assembly — those are solved problems. **The hard problem is: what does "interesting" mean, and how do you detect it across wildly different genres of content?**

A brilliant insight in an economics interview, a perfectly timed joke in a comedy podcast, a shocking revelation in a true crime show, and a moving personal confession in a memoir episode are all "interesting" — but for completely different reasons, detectable through completely different signals, and invisible to any single scoring heuristic.

This project's core architectural bet is that **"interesting" is best detected by aggregating many weak signals rather than relying on any single strong one.** A transcript alone can't hear laughter. An energy meter alone can't understand context. An LLM alone can't judge comedic timing. YouTube replay data alone is only available for some content. But layered together — text patterns, audio features, LLM judgment, and external engagement data where available — these signals reinforce each other and degrade gracefully when any one is missing.

The signal aggregation architecture described in this document is the intellectual core of the project. Everything else — the library packaging, the MCP server, the reference app — is infrastructure around it. Phase 0 exists primarily to discover which signals actually predict what humans find interesting, and in what combinations.

### The end-user goal (reference application)

> An AI solution that clips the most interesting moments from the podcasts I follow and gives me a single 60–90 minute audio digest of the best content each week. I subscribe to it like a regular podcast in whatever app I already use. I never have to think about it.

This is the launch product. It is built on top of a more general primitive — see below — and the primitive is the actual deliverable. The podcast digest is the proof point.

### The build approach

This project follows the **building block economy** model articulated by Mitchell Hashimoto: rather than building a polished mainline app first, we build a high-quality, well-documented, composable primitive that other people (and AI agents) can assemble into many different applications. The hosted consumer product is downstream of, and built on top of, the same primitive.

Concretely, this means three layered artifacts, built in this order:

- **Phase 1 — Core library (`clipfuse`).** A small, generally-applicable Python library for extracting and assembling highlights from long-form audio. Open-source, Apache 2.0. Domain-agnostic by design: nothing in the core knows what a podcast is.
- **Phase 2 — Reference application (`poddigest`) + MCP server + Skill.** A working podcast weekly-digest application built on top of `clipfuse`, plus an MCP server and a Claude Skill that make the workflow accessible to non-Python users via Claude Code / Claude Desktop.
- **Phase 3 — Hosted consumer app.** A managed web/iOS service that sells the removal of setup friction (no install, no cron, no API keys) and absorbs inference costs in exchange for a subscription fee. Same library underneath.

Each phase is a strict superset of the previous one. The hosted app uses the same library that the open-source users use; quality parity is a feature, not a compromise.

The naming split matters. Calling the core library something podcast-shaped (and putting `Episode` and `show` in the data model) would cut the addressable developer audience by an order of magnitude. The actual primitive is useful for meeting recordings, lecture review, sales-call analysis, conference-talk highlights, and audiobook-best-of just as much as for podcasts. By keeping the core neutral and shipping podcasts as a reference adapter, we get the best of both: a real working consumer product *and* a building block that other developers will fork for unrelated use cases.

---

## 2. Goals and non-goals

### Goals

- Ship a small, polished, well-documented Python library that does highlight extraction on long-form audio better than any alternative — measured against a published eval set.
- Treat highlight detection as a **signal aggregation problem**, not a single-mechanism problem. The system should get better when more signals are available and degrade gracefully when fewer are.
- Ship a working reference application (the podcast weekly digest) that anyone can run on their own laptop with a Claude or OpenAI key for $0–10/month in marginal cost.
- Make every stage of the pipeline a standalone primitive that downstream builders can assemble into different products. Nothing in the core library assumes the input is a podcast.
- Make the library agent-friendly: clean function signatures, predictable cost per call, no hidden state, examples that an LLM can learn from in one read.
- Maintain quality parity between open-source and hosted versions of the consumer product. The hosted version's moats are infrastructure, shared cache, and onboarding UX — not code.
- Output digests via private RSS so the listener consumes them in their existing podcast app, with no new app to install.

### Non-goals

- We do not host or redistribute audio. All audio streams from the original publisher's CDN; we only process timestamps and transcripts.
- We do not build a custom podcast player.
- We do not train custom models in phase 1. A learned reranker enters the picture only when phase 3 has accumulated real feedback data.
- We do not build a creator-side clip tool. Producers using OpusClip / Descript / etc. are not the target user.
- We do not target enterprise / B2B in phase 1. The consumer product is for individual listeners.
- We do not build an LLM fine-tuning pipeline.
- We do not build a proprietary scoring algorithm hidden from the open-source version.

---

## 3. The end-user experience (target state)

The phase 3 hosted-app experience, which is the north star for what the open-source library must enable:

1. User signs up, pastes their OPML subscription export from any existing podcast app.
2. The system polls each subscribed feed for new episodes throughout the week.
3. For each new episode, the system fetches the audio, transcribes it, and gathers every available signal about what's interesting — text patterns, audio features (energy, laughter, speech rate), LLM genre-aware judgment, and external engagement data (YouTube replay heatmap, creator chapters, timestamped comments) when available.
4. Once a week (configurable: any day, any time, any length), the system fuses those signals into a ranked highlight list, selects the best clips across all the user's subscriptions subject to a total-duration budget and fairness constraints, and assembles them into a single MP3 with smooth transitions and short TTS intros announcing each clip's source.
5. The digest is published to a private RSS feed unique to that user. The user has subscribed to that feed in their podcast app of choice. The new episode appears automatically.
6. The user listens. They thumb-up or thumb-down individual clips, and that signal personalizes future digests.

The phase 1+2 open-source experience is the same, except the user wires up the pieces themselves on their own laptop: install the library, install the reference app, set their API key, run the pipeline as a local cron job, and publish the resulting MP3 to a personal RSS file they host on Dropbox, S3, or GitHub Pages.

---

## 4. System architecture

The pipeline has five stages. In the core library, each is a small set of pure functions over typed data structures. In the reference application, they are wired together into a working podcast digest pipeline.

### Stage 1: Source

Fetches audio bytes from somewhere. The core library exposes a generic `fetch_audio(url) -> Path` and a small `AudioSource` type. Podcast RSS, OPML parsing, and feed polling live in the reference application, not the core. YouTube and other adapters are explicit non-goals for v1 of the core — they enter as contributed adapters once the surface is stable.

### Stage 2: Transcribe

Converts audio to a structured transcript with sentence-level segments and word-level timestamps. v1 ships one backend: OpenAI's Whisper API. The protocol is small enough that adding `whisper.cpp`, Deepgram, or AssemblyAI is a contributed PR — not core surface.

### Stage 3: Gather signals — the brain of the system

This is the stage where the project either succeeds or fails. Everything else is plumbing.

The core insight is that "interesting" is not a single measurable property. It's a convergence of multiple independent signals, each of which is individually weak but collectively powerful. The system gathers every signal it can for each candidate region of audio, then fuses them into a single score.

Signals fall into four tiers based on availability and cost:

**Tier 1 — Text signals (always available, free).** These run on the transcript with no external calls. They are the baseline that every piece of audio gets, no matter how niche.

- **Host reaction markers.** When a host says "wow," "wait, say that again," "that's incredible," "no way," "I've never heard that before" — those are explicit editorial signals that the *host* found something interesting. Trivially detectable from transcript text. High-precision, low-recall: not every interesting moment triggers a host reaction, but nearly every host reaction marks an interesting moment.
- **Structural patterns.** Question-answer pairs where the answer is long and the questioner doesn't interrupt (the guest is holding the floor — likely because what they're saying is compelling). Speaker-change density: a rapid back-and-forth exchange often signals engagement. Monologue length: an unusually long uninterrupted segment from a guest often means they're on a roll.
- **Linguistic markers.** Strong-claim language ("the single most important thing," "this is the part nobody talks about," "here's what actually happened"). Narrative markers ("so here's what happened next," "and then everything changed"). Specificity: concrete numbers, names, and dates signal substance over filler.
- **Ad and filler detection.** The inverse signal: segments that match ad-read patterns ("brought to you by," "use code [X] at checkout"), generic intros/outros, and throat-clearing filler should be suppressed. Rules-based for v1; small classifier if precision is too low.

**Tier 2 — Audio signals (always available when we have the audio file, cheap).** These run on the raw audio waveform. They catch things the transcript can't: delivery, energy, laughter, pacing.

- **Energy envelope (RMS loudness over time).** Peaks correlate with excitement, emphasis, arguments heating up. A sustained loud segment in an otherwise calm episode is almost always interesting. Computationally trivial with `librosa` or `numpy` directly.
- **Speech rate variation.** People slow down when making an important point and speed up during filler or recaps. Measurable from word timestamps already in the transcript. A sharp deceleration after a fast passage is a strong "this matters" signal.
- **Laughter detection.** Laughter from the host, guests, or a studio audience is one of the strongest signals that something just landed — especially on comedy and interview shows. Pretrained classifiers exist (e.g., `pyAudioAnalysis`, YAMNet) that can detect laughter events with ~70–80% recall. Even noisy detection is useful because laughter is high-precision: people rarely laugh at boring content.
- **Music and jingle detection.** Presence of background music or jingles usually indicates intros, outros, ad transitions, or segment breaks — things to *exclude*, not include. Useful as a suppression signal.

**Tier 3 — LLM scoring (available when the caller has an API key, expensive).** This is the most powerful single signal but also the most expensive. It's intentionally the third tier, not the first, because the system should produce useful results even without it.

- **Genre-aware open-ended scoring.** Instead of a rigid rubric with five predetermined axes, the LLM receives the transcript chunk plus genre context and an open-ended prompt: *"You are reviewing a segment from a [genre] podcast. Identify whether this segment would be one of the moments a fan of this show would most want to hear if they only had 10 minutes. Score 0.0–1.0 and explain your reasoning."* The key design choice is to let the LLM use its own internalized understanding of genre conventions rather than constraining it to axes that may not apply. A comedy podcast gets judged on comedic criteria; a true crime podcast gets judged on narrative tension; an interview gets judged on insight density — all from the same prompt, because the LLM already knows the difference.
- **Genre detection.** The LLM infers genre from the first few minutes of transcript. Alternatively, genre is passed in as metadata from the RSS feed's iTunes category tag (which most feeds have). This genre context shapes the scoring prompt.
- **Boundary refinement.** The LLM adjusts candidate start/end timestamps so clips begin and end at natural points: after a question is asked, before a topic change, at a sentence boundary. This is one of the few things the LLM does better than any heuristic.

**Tier 4 — External engagement signals (available for some content, free or nearly free).** These are proxy signals from public platforms that tell you what *other humans* found interesting. When available, they are often the strongest signal of all — but they're only available for a subset of content.

- **YouTube "Most Replayed" heatmap.** Many popular podcasts publish full episodes on YouTube. YouTube exposes a public engagement heatmap showing which parts of the video were most replayed by viewers. This is crowdsourced "interesting" data from a massive audience, available for free. For any content that's also on YouTube, the replay-heatmap peaks are extremely strong priors for where the interesting segments are. This is likely the single most valuable external signal available.
- **YouTube timestamped comments.** Viewers frequently comment "32:15 this part was gold" or "the section at 1:04:00 about supply chains is incredible." These are parseable, high-signal, and free.
- **Creator-provided chapters.** Many podcasts now include chapter markers in their RSS feeds or YouTube descriptions. Chapters don't tell you which chapters are *best*, but they give you topic boundaries that the scorer can use to evaluate self-contained segments rather than arbitrary windows.
- **Show notes and episode descriptions.** Creators often highlight key topics with timestamps in their show notes. This is the creator's own editorial judgment about what's worth calling out.
- **Social media clips.** Podcasters increasingly post short clips on Twitter/X, TikTok, Instagram. These are the creator's own highlight picks — their answer to "what was the best moment." Scraping a show's social accounts for episode-adjacent clips tells you what the creator thought was best.

### The signal availability matrix

Not every episode gets every signal. The system must degrade gracefully.

| Signal | Tim Ferriss (popular, on YT) | Niche history pod (audio only) | Internal meeting recording |
|---|---|---|---|
| Host reactions | ✓ | ✓ | ✓ (different markers) |
| Structural patterns | ✓ | ✓ | ✓ |
| Linguistic markers | ✓ | ✓ | ✓ |
| Energy envelope | ✓ | ✓ | ✓ |
| Speech rate | ✓ | ✓ | ✓ |
| Laughter | ✓ | Rare | Rare |
| LLM scoring | ✓ (if API key) | ✓ (if API key) | ✓ (if API key) |
| YouTube replay | ✓ | ✗ | ✗ |
| YouTube comments | ✓ | ✗ | ✗ |
| Creator chapters | ✓ | Maybe | ✗ |
| Show notes timestamps | ✓ | Maybe | ✗ |
| Social media clips | ✓ | ✗ | ✗ |

The popular show might have 10+ signals converging. The niche podcast has 5–6. The meeting recording has 4–5. The system should produce useful results in all three cases, and should be *better* for the popular show because more signals mean more confidence — not because the popular show gets special treatment.

The library communicates this honestly to the caller via a `confidence` field on each highlight: higher confidence when more signals agree, lower when the result rests on fewer signals or when signals disagree.

### Signal fusion

Each signal collector returns a `SignalSet` for each candidate region: a dict mapping signal names to scores (0.0–1.0) plus a boolean indicating whether that signal was available at all (absent vs. scored-zero are different). The fusion step combines available signals into a single `fused_score`.

v1 fusion is a **weighted average over available signals**, where the weights are hand-tuned during phase 0 against the eval set. Missing signals are excluded from the average (not treated as zero). The weights are exported as `DEFAULT_WEIGHTS` and are overridable by the caller.

Phase 3 fusion replaces the hand-tuned weights with a **learned model** (XGBoost on signal features + user preference features) trained on aggregate engagement data from paying users. The open-source version keeps the hand-tuned weights indefinitely; the learned model is the hosted app's moat that compounds over time.

The critical property of this design: **the system gets better when you add signals and never gets worse.** Each signal collector is independently optional. If someone writes a new signal collector (say, "detect applause in live-recorded podcasts"), they can add it to the pipeline and the fusion layer will incorporate it without changing anything else. This is what makes it a genuine building block — downstream developers can add domain-specific signal collectors for their use case (meeting recordings, lectures, sales calls) without forking the core.

### Stage 4: Select across sources

Once every source has its fused and ranked highlights, the cross-source selector decides which actually go in the final compilation. This is a constrained knapsack: maximize total fused score subject to a duration budget, with optional fairness constraints (minimum picks per source, maximum percentage per source, near-duplicate detection). One small module, one function, optional dependency on `pulp` for the LP solver with a greedy fallback.

### Stage 5: Extract and assemble

ffmpeg cuts each selected highlight from the original audio with a short fade in/out. Boundaries snap to the nearest sentence break from the transcript (or to the LLM-refined boundary if LLM scoring was used). Optionally, a short TTS-generated intro is prepended ("Next, from *Acquired*…") so the listener has orientation when driving. Clips are concatenated into a single MP3 with consistent loudness normalization and ID3 chapter markers.

The reference application then handles RSS publishing. The core library returns an MP3 path and chapter metadata; what the caller does with it is their problem.

### Architecture diagram

```
┌────────────┐    ┌──────────────┐
│  Source     │───▶│  Transcribe  │
│  (fetch)    │    │  (Whisper)   │
└────────────┘    └──────┬───────┘
                         │
              ┌──────────┼──────────────────────┐
              │          │                      │
              ▼          ▼                      ▼
     ┌──────────┐  ┌───────────┐  ┌────────────────────┐
     │  Text    │  │  Audio    │  │  External signals  │
     │ signals  │  │  signals  │  │  (YT, chapters,    │
     │ (free)   │  │  (cheap)  │  │   social) (free)   │
     └────┬─────┘  └─────┬─────┘  └─────────┬──────────┘
          │              │                   │
          │    ┌─────────┼───────────────────┘
          │    │         │
          ▼    ▼         ▼
     ┌─────────────────────┐
     │   LLM scoring       │ ← optional, expensive
     │   (genre-aware)     │
     └──────────┬──────────┘
                │
                ▼
     ┌─────────────────────┐
     │   Fusion             │ ← weighted avg (v1) or learned model (v3+)
     │   → ranked highlights│
     └──────────┬──────────┘
                │
                ▼
     ┌─────────────────────┐     ┌──────────────┐
     │   Select            │────▶│   Assemble   │
     │   (knapsack)        │     │   (ffmpeg)   │
     └─────────────────────┘     └──────┬───────┘
                                        │
                                        ▼
                               ┌──────────────────┐
                               │ Caller's problem  │
                               │ (RSS, hosting,    │
                               │  scheduling, UX)  │
                               └──────────────────┘
```

---

## 5. The core library (`clipfuse`) — the building block

The library is the foundational artifact of this project. Everything else (reference app, MCP, Skill, hosted app) is built on top of it. The library's quality determines the project's quality.

### Design principles

1. **Signal aggregation is the core.** The `signals/` package is the intellectual center of the library. Everything else (source, transcribe, select, clip, assemble) is plumbing around it. The quality of highlight detection is determined by how many signals converge and how well they're fused. The library's architecture makes it easy to add new signal collectors and hard to regress existing ones.
2. **Domain-agnostic core.** Nothing in the core knows what a podcast is. The data model is `AudioSource`, `Transcript`, `SignalSet`, `Highlight`, `Clip` — not `Episode`, `Show`, `Feed`. Podcast-specific concepts live in the reference application.
3. **Pure functions over stateful classes.** Each stage is a function that takes typed inputs and returns typed outputs. No constructor dance, no hidden state, no `self`. This makes the library trivial to test, trivial for an agent to compose, and trivial to read. Cost is legible because each function is one operation.
4. **Graceful degradation over hard requirements.** The system should produce useful results with only a transcript and an audio file (tier 1 + tier 2 signals). It should produce *better* results when an LLM key is provided (tier 3). It should produce the *best* results when external engagement data is available (tier 4). No signal is required. Every signal helps.
5. **Sentences are the atom; words are the drill-down.** Most callers want to operate on sentence-level segments, not words. The `Transcript` type makes sentences first-class with words available via `.words()`.
6. **Small surface, deeply polished.** v1 ships the signal architecture, one transcription backend, and no `feedback/` or `publish/` modules. Each signal collector is small and testable in isolation. Contributed signal collectors from the community are the primary growth vector.
7. **No magic, no framework.** The library is a vocabulary of primitives, not an orchestration framework. Composition is the user's job. Example scripts in `/examples/` demonstrate common compositions.
8. **Stable typed interfaces.** Every public function has type hints. Core data structures are versioned dataclasses. Semver. Public API is `stable`; everything else is marked `experimental` and subject to break.
9. **Cost is legible.** Functions that make API calls are clearly named and separated from functions that are local-only. The caller always knows what costs money.

### v1 module structure

```
clipfuse/
├── models.py              # AudioSource, Transcript, Segment, SignalSet,
│                          # CandidateRegion, Highlight, Clip, Compilation
├── source.py              # fetch_audio(url) — generic URL fetcher
├── transcribe.py          # transcribe(audio, backend="whisper-api")
│
├── signals/               # ← The heart of the library
│   ├── __init__.py        # gather_signals() — orchestrates available collectors
│   ├── base.py            # SignalCollector protocol
│   ├── text.py            # Host reactions, structural patterns, linguistic markers
│   ├── audio.py           # Energy envelope, speech rate, laughter detection
│   ├── llm.py             # Genre-aware LLM scoring, boundary refinement
│   ├── external.py        # YouTube heatmap, chapters, show notes, social clips
│   └── fusion.py          # fuse_and_rank() — weighted combination → Highlights
│
├── select.py              # knapsack_select() with fairness constraints
├── clip.py                # extract_clip(), snap_to_sentence_boundary()
├── assemble.py            # assemble_compilation(), add_chapter_markers()
├── tts.py                 # synthesize_intro() — optional, behind [tts] extra
├── eval.py                # score_against_labels(), eval dataset format
└── __init__.py            # explicit re-exports of the stable public API

examples/
├── 5_line_hello.py            # The README hook — works with zero config
├── podcast_digest.py          # Full weekly digest pipeline (reference app core)
├── meeting_highlights.py      # Same library, different domain
├── score_one_episode.py       # Single-episode analysis
├── signals_only_no_llm.py     # Highlights using only free signals (no API key)
└── custom_signal_collector.py # How to write and register your own signal

eval/
├── README.md                  # How to contribute labeled episodes
├── schema.yaml                # Eval label format definition
├── episodes/                  # The published eval set
│   ├── interview_tim_ferriss_001.yaml
│   ├── interview_lex_fridman_001.yaml
│   ├── comedy_conan_001.yaml
│   ├── comedy_smartless_001.yaml
│   ├── true_crime_serial_001.yaml
│   ├── narrative_radiolab_001.yaml
│   ├── news_hard_fork_001.yaml
│   ├── educational_huberman_001.yaml
│   └── ... (20+ episodes across genres)
└── baselines/                 # Published baseline scores per signal config
    ├── text_only.json
    ├── text_plus_audio.json
    ├── text_plus_audio_plus_llm.json
    └── all_signals.json

tests/
docs/
```

The `signals/` package is ~6 files and is the core IP. The rest of the library (source, transcribe, select, clip, assemble) is plumbing that changes rarely. Most community contributions will be new signal collectors or improvements to existing ones — which is exactly where contributions are most valuable.

The `eval/baselines/` directory publishes scores for different signal configurations, so anyone can see exactly how much each signal tier contributes. This is the forcing function for quality: if adding a signal collector doesn't move the eval numbers, it doesn't ship.

### Core data types

```python
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path
from typing import Any

@dataclass(frozen=True)
class AudioSource:
    """Anything that has audio bytes and a stable identity.

    The core library treats podcasts, YouTube videos, meeting recordings,
    and local files identically. The `metadata` dict carries domain-specific
    extras the caller needs to round-trip (show name, guest, genre, YouTube
    URL, chapter list, etc.) but the core never reads it — signal collectors
    may optionally read specific keys from it.
    """
    id: str               # stable hash; caller's responsibility to make it stable
    url: str              # where to fetch audio from
    duration_seconds: int | None = None
    title: str | None = None
    metadata: dict[str, Any] | None = None
    # Metadata keys that signal collectors look for (all optional):
    #   "genre": str            — e.g. "interview", "comedy", "true_crime"
    #   "youtube_url": str      — enables YouTube engagement signals
    #   "chapters": list[dict]  — creator-provided chapter markers
    #   "show_notes": str       — episode description / show notes text

@dataclass(frozen=True)
class Word:
    text: str
    start: float          # seconds
    end: float
    speaker: str | None = None

@dataclass(frozen=True)
class Segment:
    """A sentence-ish chunk. The atom most callers operate on."""
    text: str
    start: float
    end: float
    speaker: str | None = None
    words: tuple[Word, ...] = ()   # drill-down when needed

@dataclass(frozen=True)
class Transcript:
    source_id: str
    segments: tuple[Segment, ...]   # primary unit
    language: str
    backend: str                    # which transcriber produced this

    def text(self) -> str: ...
    def words(self) -> list[Word]: ...
    def slice(self, start: float, end: float) -> "Transcript": ...

@dataclass(frozen=True)
class SignalSet:
    """All signals gathered for a single candidate region.

    Each signal is a float 0.0–1.0. Absent signals (the collector wasn't
    run or the data wasn't available) are omitted from the dict entirely.
    This distinction matters: a score of 0.0 means "this signal says it's
    not interesting." Absent means "we don't know."
    """
    scores: dict[str, float]            # signal_name → 0.0–1.0
    available: frozenset[str]           # which collectors ran (superset of scores.keys)
    details: dict[str, Any] = field(default_factory=dict)
    # details holds per-signal extras:
    #   "llm_rationale": str
    #   "llm_genre": str
    #   "host_reaction_phrases": list[str]
    #   "youtube_replay_percentile": float

    @property
    def coverage(self) -> float:
        """Fraction of available signals that returned a score."""
        return len(self.scores) / len(self.available) if self.available else 0.0

@dataclass(frozen=True)
class CandidateRegion:
    """A region of audio that has been scored by signal collectors."""
    source_id: str
    start: float
    end: float
    signals: SignalSet
    refined_start: float | None = None   # LLM-adjusted boundary
    refined_end: float | None = None

@dataclass(frozen=True)
class Highlight:
    """A CandidateRegion that has been through fusion and ranking."""
    source_id: str
    start: float                        # final boundary (refined if available)
    end: float
    fused_score: float                  # 0.0–1.0, output of fusion
    confidence: float                   # how many signals agreed; higher = more trust
    signals: SignalSet                  # full signal detail preserved for debugging
    rationale: str                      # human-readable explanation of why this ranked

@dataclass(frozen=True)
class Clip:
    highlight: Highlight
    audio_path: Path
    intro_text: str | None = None

@dataclass(frozen=True)
class Compilation:
    sources: tuple[AudioSource, ...]
    clips: tuple[Clip, ...]
    output_path: Path
    chapter_markers: tuple[tuple[float, str], ...]
    total_duration_seconds: int
```

### The 5-line hello world

This is the README's first code block. Works with only free signals — no API key needed. If this isn't possible, the API is wrong.

```python
from clipfuse import transcribe, gather_signals, fuse_and_rank

transcript = transcribe("episode.mp3")
candidates = gather_signals(transcript, audio_path="episode.mp3")
highlights = fuse_and_rank(candidates)

for h in highlights[:5]:
    print(f"{h.start:.0f}s–{h.end:.0f}s  ({h.fused_score:.2f}): {h.rationale}")
```

Note: this uses only tier 1 (text) and tier 2 (audio) signals. No API key, no cost, runs entirely locally. The results are decent. Adding an LLM makes them better:

```python
from clipfuse import transcribe, gather_signals, add_llm_scoring, fuse_and_rank

transcript = transcribe("episode.mp3")
candidates = gather_signals(transcript, audio_path="episode.mp3")
candidates = add_llm_scoring(candidates, transcript, llm="claude-sonnet-4-6")  # ← the expensive step
highlights = fuse_and_rank(candidates)
```

Adding external signals makes them better still:

```python
candidates = gather_signals(
    transcript,
    audio_path="episode.mp3",
    metadata={"youtube_url": "https://youtube.com/watch?v=...", "genre": "interview"},
)
```

Each line is optional. Each line makes the output better. That's the building-block property: you snap together the pieces you have.

### A complete pipeline composition

The longer example showing how every stage fits together. This is what `examples/podcast_digest.py` looks like — it's also the reference application's core loop:

```python
from clipfuse import (
    fetch_audio, transcribe,
    gather_signals, add_llm_scoring, fuse_and_rank,
    knapsack_select, extract_clip, assemble_compilation,
)

# 1. Caller is responsible for knowing what audio they care about.
sources = my_app.get_new_podcast_episodes_this_week()  # not clipfuse's job

# 2. Transcribe each one.
transcripts = {}
for s in sources:
    audio = fetch_audio(s.url)
    transcripts[s.id] = transcribe(audio)

# 3. Gather signals per source. Each source gets as many signals as possible.
all_candidates = []
for s in sources:
    # Free signals: text + audio (always available)
    candidates = gather_signals(
        transcripts[s.id],
        audio_path=fetch_audio(s.url),
        metadata=s.metadata,    # may contain youtube_url, genre, chapters, etc.
    )
    # Expensive signal: LLM scoring (optional but recommended)
    candidates = add_llm_scoring(
        candidates,
        transcripts[s.id],
        llm="claude-sonnet-4-6",
        context={"user_interests": ["startups", "biology"]},
    )
    all_candidates.extend(candidates)

# 4. Fuse signals into ranked highlights.
all_highlights = fuse_and_rank(all_candidates)

# 5. Select across sources subject to a budget.
picks = knapsack_select(
    all_highlights,
    budget_seconds=90 * 60,
    constraints={"min_per_source": 1, "max_per_source_pct": 0.3},
)

# 6. Extract and assemble.
clips = [extract_clip(s, h) for s, h in picks]
compilation = assemble_compilation(
    clips,
    output_path="weekly-digest.mp3",
    add_intros=True,
)

# 7. Caller does whatever they want with the result.
my_app.publish_to_private_rss(compilation)
```

Note: every step is a pure function. Cost is obvious: `add_llm_scoring` is the expensive one. `gather_signals` is free (local only). Everything else is local. An agent or downstream developer can call any subset in any order.

### Signal collector protocol

Anyone can write a new signal collector. The protocol is deliberately small:

```python
from clipfuse.signals.base import SignalCollector, SignalResult

class MyCustomSignal(SignalCollector):
    """Detect applause in live-recorded podcasts."""

    name = "applause_detection"

    def collect(
        self,
        transcript: Transcript,
        audio_path: Path | None = None,
        metadata: dict | None = None,
    ) -> list[SignalResult]:
        """Return a list of (start, end, score) for regions where this signal fires."""
        ...
```

Register it and it's automatically included in `gather_signals()`:

```python
from clipfuse.signals import register_collector
register_collector(MyCustomSignal())
```

This is the primary extension point for the building-block economy. A sales-call analysis tool adds a "customer objection detection" signal. A lecture tool adds a "key definition" signal. A comedy tool adds a better laughter classifier. Each extends the library without forking it.

### What the library MUST do (v1 acceptance criteria)

**Source**
- Fetch audio bytes from any HTTPS URL. Stream to disk, don't load into memory.
- Compute a stable `AudioSource.id` from the URL when the caller doesn't provide one.
- Cache fetched audio in a configurable temp directory; never persist or redistribute it.

**Transcribe**
- Produce sentence-level segments with word-level drill-down. Word timestamps accurate to ±200ms.
- Identify speaker turns when the backend supports diarization. Mark `speaker=None` when not.
- Handle audio files up to 4 hours long without OOM.
- Return a `Transcript` whose `backend` field records which backend produced it (so callers know what to trust).
- v1 ships one backend (Whisper API). The protocol is small and documented; adding a second backend is a clean PR.

**Signal collection — text signals (v1, required)**
- Host reaction detection: identify exclamatory responses ("wow," "incredible," "say that again," etc.) with timestamp mapping. Precision above 85% on the eval set.
- Structural pattern scoring: Q&A density, speaker-change rate, monologue length. Normalized to 0.0–1.0.
- Linguistic marker scoring: strong-claim language, narrative markers, specificity markers. Normalized to 0.0–1.0.
- Ad and filler detection: identify sponsor reads, intro/outro boilerplate. Precision above 90% on the eval set. These segments get a suppression flag, not a zero score (so the fusion layer knows to exclude them rather than merely down-rank them).

**Signal collection — audio signals (v1, required)**
- Energy envelope: RMS loudness computed in 1-second windows, normalized per-episode. Peaks mapped to candidate regions with scores.
- Speech rate: words-per-minute computed from word timestamps, smoothed over 30-second windows. Deceleration events (sharp slowdown) flagged as "emphasis" signals.
- Laughter detection: at minimum, a simple classifier (pretrained YAMNet or similar) that labels audio frames as laughter/not-laughter. Recall above 60% is acceptable in v1; precision above 70% is required (false positives are more damaging than false negatives because they surface unfunny content).
- Music/jingle detection: identify non-speech audio segments for suppression. Rules-based is fine for v1.

**Signal collection — LLM scoring (v1, optional)**
- Genre-aware open-ended prompt, not a rigid rubric. The prompt adapts to detected or declared genre.
- Genre detection: infer from first 2 minutes of transcript, or accept from `metadata["genre"]`.
- Boundary refinement: adjust candidate start/end to natural breakpoints.
- Return rationale string per candidate.
- The `llm` argument accepts a string model name *or* a callable conforming to a tiny `LLMClient` protocol, so the library is not coupled to any specific provider.
- Must work with Claude and OpenAI out of the box. Other providers via the protocol.

**Signal collection — external signals (v1, best-effort)**
- YouTube "Most Replayed" heatmap extraction: given a YouTube URL in metadata, fetch the engagement heatmap and map peaks to candidate scores. If the heatmap API changes or is unavailable, fail silently (this is a best-effort signal, not a required one).
- Creator chapter extraction: if `metadata["chapters"]` is provided, use chapter boundaries to align candidate regions. Score chapters that are explicitly called out in show notes higher.
- YouTube timestamped comment extraction: parse comments for timestamp references, map to candidate regions, score by comment sentiment and upvotes.
- Show notes timestamp extraction: parse `metadata["show_notes"]` for timestamp patterns, map to candidate regions.
- Social media clip matching: deferred to v1.x. Worth doing but the scraping complexity is high for v1.

**Signal fusion (v1, required)**
- `fuse_and_rank(candidates, weights?)` takes candidate regions with signal sets and returns ranked highlights.
- Default weights (`DEFAULT_WEIGHTS`) hand-tuned during phase 0 against the eval set. Exported and overridable.
- Missing signals excluded from the weighted average, not treated as zero.
- Confidence score computed as a function of how many signals agreed and how many were available.
- Rationale string generated by summarizing which signals contributed most to each highlight's score. If LLM rationale is available, include it. Otherwise, generate a template: "High energy + host reaction + structural peak."
- Suppressed regions (ads, intros, outros) are excluded regardless of other signal scores.
- Published baseline weights with justification in the docs: which signals matter most and why.

**Select**
- `knapsack_select(highlights, budget_seconds, constraints)` solves the constrained knapsack: maximize total fused score subject to budget.
- Constraint dict supports `min_per_source`, `max_per_source_pct`, `dedupe_threshold`, and `ordering` (best-first, chronological, interleaved).
- Detects near-duplicate highlights via embedding cosine similarity above a threshold.

**Clip and assemble**
- `extract_clip(source, highlight)` cuts the audio with sentence-aligned boundaries and 50ms fade in/out.
- `assemble_compilation(clips, output_path, add_intros=False)` concatenates with consistent loudness normalization and writes ID3 chapter markers.
- Optional TTS intros via `clipfuse.tts.synthesize_intro()` behind a `[tts]` extra (default: OpenAI TTS).
- Embeds source title and original URL in chapter metadata.

**Eval**
- `clipfuse.eval` ships a labeled dataset format (YAML: `source_url + labeled_highlights + labeled_suppressions + genre + notes`).
- `score_against_labels(highlights, labels) -> EvalResult` returns precision, recall, F1, and timestamp-overlap metrics.
- The eval set covers at least 5 genres with at least 3 episodes each: interview, comedy, true crime / narrative, news / analysis, educational / explainer. 20 episodes minimum for v1.
- Published baseline scores per signal configuration: text-only, text+audio, text+audio+LLM, all-signals. These baselines are the gating criterion for release and the yardstick for every future PR.
- Eval labels include genre tags and signal-availability annotations so that baselines can be computed per-genre and per-signal-configuration.
- CI integration: every PR that touches `signals/` runs the eval suite and posts before/after scores as a PR comment. No PR that regresses the eval score merges without documented justification.

**Cross-cutting**
- Every long-running operation accepts an optional `progress` callback.
- Every module emits structured logs through the standard library's `logging`. Silent by default.
- Zero hard runtime dependencies beyond Python stdlib + ffmpeg + `numpy` (for audio signals).
- Optional extras: `pip install clipfuse[whisper-api]`, `[anthropic]`, `[openai]`, `[tts]`, `[lp]`, `[youtube]` (for yt-dlp and engagement data extraction).
- Type hints throughout. Public API documented with examples. Test coverage above 80% on core modules.
- Apache 2.0 license. Decided. No revisiting.

### What the library MUST NOT do

- Host, copy, or redistribute source audio.
- Bundle credentials for any third-party API.
- Assume any specific LLM or transcription provider.
- Phone home with telemetry. The library is silent unless the user explicitly enables logging.
- Make decisions about UX, branding, scheduling, persistence, or feed publishing — those belong in the surfaces built on top.
- Define what a "podcast" is. The reference application does that.
- Bundle a `feedback` module with no users yet to provide feedback.
- Bundle a `publish` module — RSS is an application concern.
- Rely on any single signal for quality. If the LLM is unavailable, the system still works. If YouTube data is unavailable, the system still works. Only the transcript and audio file are truly required.

### Stability policy

- **Public API** (everything exported from `clipfuse/__init__.py`) follows semver. Breaking changes only at major versions. Deprecation warnings for at least one minor version before removal.
- **`experimental` API** (modules marked in their docstring as experimental) is subject to break at any minor version. Currently: `signals/external.py` in v1 (YouTube APIs are unstable and the scraping approach may need to change).
- **Internal API** (anything not re-exported from `__init__.py`, or names starting with `_`) is fair game to change at any time. Don't import it.

### Streaming and incremental processing (deferred to v1.x)

A 4-hour episode shouldn't block scoring on full transcription. The right shape is async generators: `transcribe_stream(audio) -> AsyncIterator[Segment]`, then signal collectors that emit as enough context arrives. v1 ships sync-only versions to keep the surface small. The streaming variants are sketched in the design notes and will land in v1.x as `experimental` before being promoted.

---

## 6. How quality improves over time

The system improves through four distinct mechanisms, each affecting a different artifact. This section makes explicit what gets better, how, and who can contribute — because a building block that can't improve is a building block that gets forked and abandoned.

### Mechanism 1: Signal collector code (open-source, PRs)

Each signal collector in `signals/` is a small Python file with clear inputs and outputs. Improvements look like: better host-reaction keyword lists, a more accurate laughter classifier, smarter structural-pattern heuristics, tighter ad-detection rules. A contributor changes the code, runs `clipfuse eval`, shows the numbers went up, and submits a PR.

The eval suite is the quality ratchet. CI posts before/after scores on every PR. No regression merges without justification.

**Artifact changed:** Python source files in `clipfuse/signals/`.

### Mechanism 2: Fusion weights (open-source, PRs)

The `DEFAULT_WEIGHTS` dict in `fusion.py` controls how signals are combined. During phase 0, these are hand-tuned against the eval set. A contributor can propose new weights with eval evidence. The weights are also published as per-genre defaults, so interview podcasts can weight LLM scoring higher while comedy podcasts weight laughter detection higher.

**Artifact changed:** `DEFAULT_WEIGHTS` in `clipfuse/signals/fusion.py`.

### Mechanism 3: Eval set (open-source, PRs)

The eval set is a collection of YAML files, each representing one hand-labeled episode. Contributing a labeled episode is the lowest-barrier, highest-value contribution: listen to an episode, mark the timestamps of what was good and what was bad, submit a PR adding the YAML file.

More labeled episodes means the eval is harder to overfit and more representative of real-world content. It also means the baselines are more trustworthy, which means the quality ratchet (CI checks) is tighter.

The eval set also captures which external signals are available for each episode (does it have a YouTube version? Does it have creator chapters?), so baselines can be computed per signal configuration.

```yaml
# eval/episodes/interview_tim_ferriss_042.yaml
source_url: "https://traffic.megaphone.fm/example.mp3"
title: "Tim Ferriss — How to Learn Any Skill in 30 Days"
genre: interview
external_signals_available:
  youtube_url: "https://youtube.com/watch?v=example"
  has_chapters: true
  has_show_notes_timestamps: true

highlights:
  - start: 1823.4
    end: 1987.2
    quality: excellent
    notes: "Core framework for skill acquisition — the 'minimum effective dose' concept"
  - start: 3012.0
    end: 3145.8
    quality: good
    notes: "Surprising personal anecdote about failing at Brazilian jiu-jitsu"

suppressions:
  - start: 0
    end: 95
    reason: intro
  - start: 2400
    end: 2520
    reason: ad_read
  - start: 4800
    end: 4900
    reason: outro

labeled_by: "github_username"
date_labeled: "2026-04-14"
```

**Artifact changed:** YAML files in `eval/episodes/`.

### Mechanism 4: New signal collectors (open-source, PRs)

The most powerful contribution path for domain experts. Someone who works in sales call analysis writes a "customer objection detection" signal collector. Someone in education writes a "key definition" signal. Someone in comedy writes a better laughter-plus-timing detector.

New signal collectors automatically integrate via the `register_collector()` API. They get evaluated against the eval set. If they improve scores, they ship.

**Artifact changed:** New Python files in `clipfuse/signals/`.

### Mechanism 5: Learned reranker (phase 3 only, proprietary)

Once the hosted app has enough users generating feedback data (thumbs up/down, skip rates, replay rates, drop-off points), a small model (XGBoost on signal features + user preference features) learns the mapping from signals to "actually interesting for this user." This is trained on aggregate engagement data from paying users.

The learned model lives in the hosted app's infrastructure, not in the open-source repo. The open-source version keeps the hand-tuned weights indefinitely. The gap between hand-tuned and learned widens over time as more feedback accumulates — this is the hosted app's compounding moat.

**Artifact changed:** Serialized model file in hosted infrastructure. Not open-sourced (trained on private user data).

### Summary

| What improves | How | Who can contribute | Artifact | Open source? |
|---|---|---|---|---|
| Signal collectors | Code PRs with eval proof | Any developer | `signals/*.py` | Yes |
| Fusion weights | Weight PRs with eval proof | Any developer | `fusion.py` | Yes |
| Eval coverage | Labeled episode PRs | Anyone who listens to podcasts | `eval/episodes/*.yaml` | Yes |
| New signal types | New collector PRs with eval proof | Domain experts | New `signals/*.py` | Yes |
| Personalized reranker | Aggregate user feedback | Only the hosted app | Model weights | No |

---

## 7. The reference application (`poddigest`)

A separate package that builds the actual podcast weekly digest on top of `clipfuse`. This is what end-users run. It's also the working example that proves `clipfuse` solves a real problem.

### What it adds on top of clipfuse

- **URL resolver**: takes Apple Podcasts links, Spotify links, Overcast links, Pocket Casts links, raw RSS URLs, or OPML files and normalizes them down to RSS feed URLs. This is the "paste any podcast link and it just works" layer.
- **OPML parsing** for subscription import from any major podcast app.
- **RSS feed polling**, episode dedup by GUID, episode metadata extraction. This is where the `Episode` and `Show` concepts actually live. Extracts `genre` from iTunes category tags, `chapters` from RSS chapter extensions, `show_notes` from episode descriptions, and `youtube_url` when the show links to YouTube — all passed through to clipfuse as `metadata`.
- **A scheduler** (a thin cron wrapper) that runs the pipeline weekly.
- **Private RSS publishing**: takes a `Compilation` from clipfuse and produces a valid RSS 2.0 + iTunes-namespace feed with the digest as a new episode.
- **Pluggable hosting backends**: local filesystem, S3, any HTTPS-reachable static host. Generates an unguessable URL per user.
- **Personalization storage**: a small JSON/SQLite store for user preferences and feedback, fed back into clipfuse's `context` dict on subsequent runs.
- **A CLI**: `poddigest init`, `poddigest add <any-podcast-url>`, `poddigest run`, `poddigest publish`.

### Why it lives outside the core

Every concept in the bullet list above is podcast-specific or application-specific. None of it belongs in a library that should also be useful for meeting recordings or lecture review. By extracting it, we get two artifacts that are each polished for their actual audience: a clean library for builders and a working app for podcast listeners. Same code repository at first; can split later if it makes sense.

---

## 8. The MCP server (`poddigest-mcp`)

A separate package that wraps the reference application's workflow and exposes it as MCP tools, making the whole thing usable from Claude Code, Claude Desktop, Cursor, and any other MCP-compatible client.

### Design principle

The MCP exposes **opinionated workflow tools, not raw library calls.** Agents do better with fewer, more meaningful tools than with a one-to-one mapping of every library function. The MCP knows about end-to-end workflows like "build a digest from this OPML"; the Skill teaches Claude when to use which.

### Tools exposed

```
fetch_new_episodes(opml_path_or_url, since_iso)
    → list of episodes across all subscribed feeds, deduped
    → accepts any podcast URL, OPML file, or RSS feed URL (resolver built in)

transcribe_episode(episode_id, backend?)
    → transcript object (cached if seen before)

analyze_episode(episode_id, include_llm_scoring?, context?)
    → gathers all available signals, fuses, returns ranked highlights
    → includes which signals were available and their individual contributions

build_digest(episode_ids, budget_minutes, output_dir, add_intros?)
    → end-to-end: select across episodes, extract, assemble, return compilation path

publish_digest(compilation_path, feed_url, hosting_backend?)
    → updates the private RSS feed with the new episode

record_feedback(clip_id, signal, value)
    → persists thumbs / skip / rewind events for next run's personalization
```

`analyze_episode` replaces the previous `find_episode_highlights`. The name change reflects the signal-aggregation architecture: the tool doesn't just "find highlights," it gathers and fuses multiple signals and returns the full picture including per-signal breakdowns.

`build_digest` is intentionally a single tool that does selection + extraction + assembly together. That's three library calls, but exposing them as three tools would let an agent stop halfway and produce nothing useful. One tool, one outcome.

### What an agent does with this

A developer in Claude Code says: *"Build my weekly podcast digest from my OPML."*

Claude (with the Skill loaded):
1. Calls `fetch_new_episodes(opml_path="~/podcasts.opml", since_iso=...)` — gets the new episodes.
2. Loops `transcribe_episode` over each. (The MCP returns immediately if cached.)
3. Loops `analyze_episode` over each, with `include_llm_scoring=True`.
4. Calls `build_digest` once with all the episode IDs and a 90-minute budget.
5. Calls `publish_digest` with the user's feed URL.

The user did one step. Claude did five tool calls. The signal aggregation is invisible to the user — it just works.

---

## 9. The Claude Skill (`poddigest-skill`)

A markdown document loaded by Claude when the user asks for podcast-digest-related help. Pure documentation that happens to be machine-readable.

### Contents

- **Mission statement**: what the project does and what the user is trying to accomplish.
- **Signal architecture overview**: a brief explanation of how quality works — so Claude can explain to the user why some episodes produce better highlights than others (more signals available) and set expectations accordingly.
- **Workflow recipes**: step-by-step orchestrations of the MCP tools for "build my weekly digest", "analyze this one episode and show me the signal breakdown", "rebuild last week's digest with different preferences", "find the best 20 minutes from this single 4-hour episode."
- **Known pitfalls**: skip intros and outros, snap boundaries to sentence breaks, never include two clips making the same point, default budget is 90 minutes, episodes with YouTube versions produce better results, etc.
- **Examples**: 3–5 sample interactions showing the user request, Claude's tool-call sequence, and the resulting metadata.
- **Troubleshooting**: what to do when transcription fails, when scoring returns low-confidence highlights, when the feed URL is unreachable, when YouTube engagement data can't be fetched.

The Skill is what makes Claude *competent* at this on the first try, not just *capable* of figuring it out eventually.

---

## 10. Phased roadmap

### Phase 0: Signal discovery (6–8 weeks part-time)

**Goal**: discover which signals actually predict what humans find interesting, and in what combinations. This is the research phase. The output is a working pipeline for personal use and a calibrated understanding of signal quality, not a public release.

This is the longest and most important phase. The temptation will be to rush past it and ship the library. Resist that. The library's value is entirely determined by how good the highlight detection is, and highlight detection quality is determined by work done in this phase.

**Week 1–2: Baseline infrastructure.**
- Implement core plumbing: `models`, `source`, `transcribe` (Whisper API), `clip`, `assemble`.
- Build the eval framework: label format, `score_against_labels()`, eval runner.
- Hand-label 10 episodes across 3 genres (interview, comedy, narrative). This is slow and important. These labels are ground truth.

**Week 3–4: Text and audio signals.**
- Implement `signals/text.py`: host reactions, structural patterns, linguistic markers, ad detection.
- Implement `signals/audio.py`: energy envelope, speech rate, laughter detection.
- Implement `signals/fusion.py` with uniform weights as starting point.
- Run eval. Record baseline: "text + audio signals only, no LLM."
- Hand-label 5 more episodes. Notice where the text+audio baseline fails.

**Week 5–6: LLM and external signals.**
- Implement `signals/llm.py`: genre-aware open-ended scoring. Iterate on the prompt.
- Implement `signals/external.py`: YouTube replay heatmap, creator chapters, show notes timestamps.
- Run eval at each signal-tier level: text-only → text+audio → text+audio+LLM → all signals.
- Publish internal baseline table. See which signals are actually moving the numbers.

**Week 6–8: Fusion weight tuning and end-to-end testing.**
- Hand-tune fusion weights against the eval set. Try per-genre weights.
- Hand-label 5 more episodes in under-represented genres.
- Build the reference app: OPML parse + scheduler + RSS publish.
- Use the digest weekly for at least 2 weeks. Fix everything that's embarrassing.
- Write down: which signals mattered, which didn't, which surprised you, what the weight distribution looks like. This is the most valuable output of phase 0.

**Phase 0 exit criteria:**
- At least 20 hand-labeled episodes across 5 genres.
- Published baseline scores for text-only, text+audio, text+audio+LLM, all-signals configurations.
- A clear answer to: "does adding audio signals meaningfully improve over text-only?" If not, the architecture is wrong and we need to rethink before going public.
- A clear answer to: "does the YouTube replay heatmap correlate with human labels?" If yes, it's probably the single highest-value external signal and should be prioritized.
- A personal weekly digest that you look forward to listening to.

### Phase 1: Public library release

**Goal**: ship a polished, documented, installable Python package that other developers will fork.

- Refactor based on what phase 0 taught. Some signals may be cut (not worth the complexity). Some may be promoted from optional to default. The module structure may shift.
- Pin the v1 surface. Mark anything provisional as `experimental` (likely: `signals/external.py`).
- Write comprehensive docs, type hints, and tests. Test coverage above 80% on core modules.
- Apache 2.0 license. Publish to PyPI.
- Publish the eval set and all baseline scores as part of the repo. Make `CONTRIBUTING.md` explain how to contribute labeled episodes.
- Ship with `examples/` populated: the 5-line hello world, podcast digest, meeting highlights, signals-only-no-LLM, custom signal collector.
- Write a launch post. The post's hook should be the signal architecture and the published baselines: "here's exactly how much each signal contributes, here's the eval set, here's how to make it better." This is what differentiates the project from a weekend hack.
- Submit to relevant communities: r/podcasts, r/selfhosted, Hacker News, the Claude Code community, Show HN.

### Phase 2: MCP server, Skill, and reference app polish

**Goal**: make the workflow accessible to Claude Code users who don't write Python.

- Build `poddigest-mcp` as a wrapper around the reference application.
- Build `poddigest-skill` as a markdown document with workflows and examples.
- Polish the reference app's CLI to the point that someone can go from `pip install poddigest` to a published digest in 5 minutes with their own API key.
- Add the URL resolver (`poddigest add <any-podcast-url>`).
- Document the agent-native workflow in a follow-up post.
- Watch what people do with it. Listen for friction points that tell you what phase 3 needs.

### Phase 3: Hosted app

**Goal**: sell the removal of setup friction to non-technical listeners.

- Build a web onboarding flow: paste any podcast URL or OPML, set preferences, get a private feed URL.
- Run the pipeline on managed infrastructure with a shared transcription cache and shared signal cache across users (the only way the unit economics close at consumer pricing).
- Absorb inference costs in exchange for a flat monthly fee. Target $15–25/month for the standard tier; higher tiers for power users with 25+ subscriptions.
- Pre-compute external signals (YouTube heatmaps, chapters, social clips) for popular shows and cache across users. This is a quality advantage the hosted version has over BYOK users: the hosted version always has external signals for popular shows; the BYOK user has to configure YouTube access themselves.
- Build a feedback loop UI: thumbs-up/down on clips inside the digest, fed back into the user's preference profile.
- Once enough feedback data accumulates (months out, not at launch), train the XGBoost reranker to replace hand-tuned fusion weights for personalization. This is the moat that compounds.

### What we explicitly do not build in any phase

- A custom podcast player.
- A creator-side clip-making tool.
- An LLM fine-tuning pipeline.
- A proprietary scoring algorithm hidden from the open-source version. The hosted app's moats are shared signal cache, pre-computed external signals for popular shows, learned personalization, and onboarding UX — not closed-source code.

---

## 11. Open questions and risks

### Open questions to resolve during phase 0

- **Do audio signals matter?** Phase 0 must answer this empirically. If energy envelope + speech rate + laughter detection don't meaningfully improve over text-only signals on the eval set, they add complexity for no value and should be cut. Don't ship them on faith.
- **How good is YouTube replay data?** If the heatmap correlates strongly with human-labeled highlights, it's worth significant engineering effort to make extraction robust. If it doesn't correlate, it's a distraction. Phase 0 tests this.
- **Genre-specific vs. universal fusion weights.** Does a single weight vector work across genres, or does each genre need its own? Phase 0 eval should test both. If genre-specific weights are significantly better, ship per-genre defaults.
- **LLM prompt design.** Open-ended ("what would a fan of this show want to hear?") vs. semi-structured ("score on these N axes"). Phase 0 should try both against the eval set. The hypothesis is that open-ended is better because it lets the LLM use genre knowledge, but this must be verified.
- **Ad detection approach.** Rules-based (skip first/last 90 seconds + watch for "brought to you by") gets us 95% of the way. Whether to add a small classifier in v1 or defer to v1.x depends on how visible the failures are during phase 0.
- **Cold-start onboarding (reference app).** A new user has no preference data. Do we ask 3–4 calibration questions at signup, infer preferences from their OPML, or just ship a generic-but-good default? Likely all three.
- **Laughter classifier quality.** Off-the-shelf classifiers (YAMNet, pyAudioAnalysis) may be good enough, or may produce too many false positives on low-quality audio. Phase 0 tests this.

### Known risks

- **Signal architecture may be over-engineered for v1.** It's possible that text + LLM alone is 90% as good as text + audio + LLM + external, and the audio/external signals add complexity for marginal gain. Phase 0 exists to test this hypothesis. If it's true, ship text + LLM only and add signals later. The architecture supports it either way — that's the point of graceful degradation. But be honest about the result; don't ship audio signals that don't move the eval numbers.
- **YouTube engagement data may be fragile.** YouTube's internal APIs are undocumented and change without notice. The "Most Replayed" heatmap is extracted via unofficial methods. This signal could break at any time. Mitigation: mark `signals/external.py` as `experimental`, test continuously, and never make it a hard dependency. The system must be good without it.
- **Cost per user scales linearly with listening hours.** A user who follows 25 shows costs roughly twice what an average user costs and there's no clean way to price for this. Mitigated by tiered plans and shared caching across users for popular shows.
- **Open-source forks compete with the hosted app.** This is by design and explicitly accepted. The hosted app's moats are shared signal cache, pre-computed external signals, learned personalization, and onboarding UX — not code.
- **AI agents prefer open-source over commercial software.** Hashimoto specifically flags this as the unsolved commercialization problem in the building-block economy. Implication: the hosted app must be marketed to humans who don't want to set anything up, not to agents who would happily configure the open-source version themselves. These are two different markets and both are legitimate.
- **Legal risk from copyright is low but nonzero.** As long as audio streams from the publisher's CDN and we never copy or redistribute the source files, we operate the same way every podcatcher app does. Transcripts as derivative works are widely treated as fair use for search/accessibility purposes. Sharing snippets externally (clips leaving the ecosystem) is the riskiest user action; we mitigate by keeping the digest in a private RSS feed scoped to one user.
- **Quality bar for the open-source release is high.** Hashimoto's footnote: building blocks must be high quality, robust, and well documented. A janky reference implementation that captures mindshare and then disappoints is worse than no release. Phase 0 exists specifically to make sure we don't ship until the output is actually good. The published eval baselines are the forcing function.
- **Surface-area creep.** The main failure mode of building-block libraries is bloating the surface to satisfy every user's special case. The discipline is to say no and to point people at writing a contributed signal collector or fork. Custom signal collectors are the pressure valve: "we won't add that to core, but here's how to write your own and register it."

---

## 12. Reference: the cost picture

Approximate marginal costs per heavy user per week, before any optimization:

| Item | Cost | Notes |
|---|---|---|
| Whisper API transcription (15h audio) | $5.00 | Biggest cost by far |
| LLM scoring (Claude/GPT) | $1.00 | Per-episode genre-aware scoring |
| Audio signal extraction | $0.00 | Local compute, negligible |
| Text signal extraction | $0.00 | Local compute, negligible |
| YouTube engagement scraping | $0.00 | Free, but fragile |
| TTS intros | $0.10 | |
| Storage and bandwidth | $0.05 | |
| **Total per heavy user / week** | **~$6** | |

With shared transcription cache and shared signal cache across users for popular shows, the amortized cost at scale drops to roughly $2–3/week/user. Self-hosting whisper.cpp on a single GPU brings transcription to near-zero at the cost of operational overhead — worth doing past ~200 users, not before.

**The "no LLM" path**: a user who doesn't provide an API key for LLM scoring (using only text + audio + external signals) has a marginal cost near zero for everything except transcription. If they also self-host Whisper, the entire pipeline is free. This is the building-block promise: the library is useful at every price point, and gets better as you spend more.

Phase 0, 1, and 2 (open-source library + reference app + MCP) have no per-user cost to the project at all because users bring their own API keys and run on their own machines. The cost to *build* phase 0–2 is roughly $500–800 in API spend during prototyping and eval iteration. The cost to *operate* is zero.

Phase 3 (hosted app) needs to price for at least $15–20/month to leave meaningful margin after blended inference costs. Higher tiers for power users with many subscriptions.

---

## 13. Reference: where this fits in the existing landscape

| Tool | Type | Listener-side? | Open source? | Highlight extraction? | Multi-signal? | Cross-source digest? |
|---|---|---|---|---|---|---|
| Snipd | Mobile podcast app | Yes | No | Per-episode | Crowdsourced clips | No (closest competitor) |
| Podwise | Web summarizer | Yes | No | Text only | No | No |
| Snipcast | Web summarizer | Yes | No | Text only | No | No |
| Podsnacks | Email digest | Yes | No | Text only | No | No |
| OpusClip / Descript / Choppity | Creator clip tools | No | No | Yes | Some | No |
| `mcp-podcast-scraper` | MCP server | Yes | Yes | No | No | No |
| `dingkwang/podcast-transcriber-mcp` | MCP server | Yes | Yes | No | No | No |
| Podscan MCP | Commercial MCP | Yes (research) | No | No | No | No |
| `PodcastAudioClipper` (notebook) | Jupyter notebook | Yes | Yes | Yes | No | Partial |
| **`clipfuse` + `poddigest`** | **Library + ref app + MCP + hosted app** | **Yes** | **Yes** | **Yes** | **Yes (signal aggregation)** | **Yes** |

Snipd's closest analogue to our signal architecture is their crowdsourced clip data — what users snip most is a strong "interesting" signal. We can't replicate that without a user base. But we can approximate it with YouTube engagement data (which is crowdsourced at a much larger scale), combine it with text/audio/LLM signals they don't use, and produce a different product (cross-source digest rather than per-episode highlights) that Snipd doesn't ship. The competition is real but the product is different.

The closest existing project to the library itself is `PodcastAudioClipper`, a single-author Jupyter notebook that does roughly the right thing but uses a single LLM call for scoring (no signal aggregation), is not packaged, not composable, and not maintained as a building block. The slot for "a polished, domain-agnostic, multi-signal audio-highlight library" is empty.

---

## 14. Summary of changes from v2

For reference if you're comparing to the previous draft:

- **Replaced the two-stage scorer (generate_candidates + rank_with_llm) with a multi-tier signal aggregation architecture.** This is the largest change. The old design put all weight on a single LLM rubric; the new design treats highlight detection as a convergence of many weak signals. The library now has four signal tiers: text (free, always available), audio (cheap, always available), LLM (expensive, optional), and external engagement data (free, sometimes available). Each tier is independently useful. More signals = better results.
- **Replaced the rigid 5-axis rubric with a genre-aware open-ended LLM prompt.** The old rubric scored every podcast on "insight density" — which is meaningless for comedy and true crime. The new design detects genre and asks the LLM an open-ended question that lets it use its own internalized genre conventions.
- **Added the `signals/` package** as the architectural center of the library, with a small protocol for contributed signal collectors. This is the primary extension point for the building-block economy.
- **Added audio signal collection to v1.** Previously deferred. Energy envelope, speech rate, and laughter detection are now v1 requirements because a transcript alone is literally deaf to half of what makes audio interesting.
- **Added external engagement signals to v1 (best-effort).** YouTube replay heatmaps, creator chapters, show notes timestamps, timestamped YouTube comments. Marked `experimental` but shipped in v1 because when available they may be the strongest signal of all.
- **Added the signal availability matrix** that shows how signal coverage varies by content type and why degradation is graceful.
- **Added `CandidateRegion` and `SignalSet` data types** to carry per-signal breakdowns through the pipeline. Added `confidence` to `Highlight` so callers know how much to trust a result.
- **Restructured phase 0 as a signal-discovery phase** with explicit per-signal baselines and exit criteria. Phase 0 is now 6–8 weeks (up from 4–6) because the signal work is the hard part and deserves the time.
- **Added Section 6 (How quality improves over time)** with an explicit table of what improves, how, and who can contribute. Replaces the hand-waved "publish the eval set so the community can help" from v2.
- **Added the URL resolver** to the reference app spec, so users can paste any podcast link (Apple, Spotify, Overcast, etc.) and it just works.
- **Added the `custom_signal_collector.py` example** and the `register_collector()` API to make the building-block extension point concrete.
- **Added `eval/baselines/`** with per-signal-configuration baseline scores, published as part of the repo.
- **Added open questions** for the empirical hypotheses that phase 0 must test: "do audio signals matter?", "how good is YouTube replay data?", "genre-specific vs. universal weights?"
- **Added the risk that the signal architecture may be over-engineered** — because it might be, and phase 0 is where you find out.

---

*Document version: 3.0 — signal aggregation architecture as the core, multi-tier signals, genre-aware LLM scoring, external engagement data, phase 0 restructured as signal discovery.*
