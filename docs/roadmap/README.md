# Riwayah roadmap

A hadith memorisation app: download collections, recite from memory, and have the app
follow along, revealing words and flagging mistakes (including tashkeel).
Background research is in [`../research/voice-listener.md`](../research/voice-listener.md).

The project runs as **two parallel tracks** that meet in Phase 5:

- **Track A: App.** Ship something usable early with an off-the-shelf speech model.
- **Track B: Model.** Turn the classroom videos into training data and build the
  custom model that makes tashkeel grading possible.

Timings assume one or two developers working steadily. They're estimates, not commitments.

```
Weeks        0    2    4    6    8   10   12   14   16   18   20+
Phase 0      ████
Phase 1 (A)       ████
Phase 2 (A)            ████████████
Phase 3 (B)         ███████████████
Phase 4 (B)                          ████████████
Phase 5 (A+B)                                  ████████████
Phase 6                                                   ███████→
```

## Phases

| # | Phase | Track | Weeks | Done when |
|---|---|---|---|---|
| 0 | [Foundations](phase-0-foundations.md) | both | 0–2 | Framework and text chosen, rights confirmed, golden test set recorded |
| 1 | [Listener prototype](phase-1-listener-prototype.md) | A | 2–4 | Baseline scorecard on the golden set |
| 2 | [App MVP](phase-2-app-mvp.md) | A | 4–10 | Beta users can memorise a collection end to end |
| 3 | [Data pipeline](phase-3-data-pipeline.md) | B | 3–10 | Clean training dataset (<2% label errors) + teacher-corrections test set |
| 4 | [Custom model v1](phase-4-custom-model.md) | B | 10–14 | Beats the baseline and runs in real time on a mid-range phone |
| 5 | [Integration & tashkeel grading](phase-5-tashkeel-grading.md) | A+B | 14–20 | Tashkeel flags ≥85% precision; beta users trust the feedback |
| 6 | [Launch & improvement loop](phase-6-launch.md) | both | 20+ | Public release; retraining on a regular schedule |

## Repo layout (target)

```
app/                 Flutter app
listener/            Tracking + grading logic (Python reference, later ported to Dart)
training/
  data_prep/         Phase 3 pipeline
  model/             Phase 4 training configs and scripts
  eval/              Scoring scripts shared by every phase
data/                Local only, never committed: videos, audio, clips, recordings
docs/
```

## Key risks

| Risk | Mitigation | Phase |
|---|---|---|
| The model "hears" the correct text over real mistakes (memorisation) | CTC decoding without text biasing, hold out whole hadith from training, and measure with deliberate-mistake test sets | 4 |
| Tashkeel errors are too subtle to detect reliably | Show confidence levels and a strict/lenient setting; switch to phoneme output (v2) if needed | 5 |
| Too few distinct readers in the videos | Track speaker count in the Phase 3 report, and top up with donated recordings from the beta | 3, 2 |
| Hadith text isn't fully or consistently vocalised | Choose the edition and measure its coverage in Phase 0 | 0 |
| Rights or consent problems with videos or text | Resolve before any training | 0 |
| Model too slow or large for phones | Smaller model, int8 quantisation, and on-device testing in Phase 4 | 4 |

## Fixed costs to expect

- GPU rental for training: tens to a few hundred dollars per run.
- Apple Developer Program: $99/year. Google Play: $25 one-time.
- Hosting for model and collection downloads: a CDN or object storage (low cost at small scale).
