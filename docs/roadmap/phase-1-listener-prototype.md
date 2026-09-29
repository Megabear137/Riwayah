# Phase 1: Listener prototype

**Weeks 2–4 · Track A · [Back to roadmap](README.md)**

Prove the core idea on a laptop before building any app: given audio and a hadith ID,
work out which words were read and which were wrong. The output is a **baseline
scorecard** that every later model has to beat.

Work in Python under `listener/`. Python is fast to iterate in, and Phase 2 ports the
logic to Dart once it's settled.

## 1. Set up the speech engine

```bash
pip install sherpa-onnx soundfile numpy rapidfuzz jiwer
```

Get 2–3 candidate models to compare:
- An Arabic or multilingual model from the
  [sherpa-onnx pretrained models list](https://k2-fsa.github.io/sherpa/onnx/pretrained_models/index.html)
  (streaming models preferred).
- `tarteel-ai/whisper-base-ar-quran` exported to ONNX (see sherpa-onnx's Whisper
  export scripts). It's not streaming, but it's a useful comparison.
- A general multilingual Whisper (tiny/base) as a control.

Write `listener/asr.py` with one interface for all of them:

```python
class Recogniser:
    def feed(self, samples: np.ndarray) -> None: ...   # 16 kHz float32 chunks
    def partial(self) -> str: ...                      # current hypothesis
    def reset(self) -> None: ...
```

To simulate live use, feed each golden-set file in **100–200 ms chunks** and record the
partial text after each chunk, with timestamps.

## 2. Text normalisation (`listener/text.py`)

Two functions, used everywhere later:

```python
DIACRITICS = re.compile(r'[ً-ْٰ]')
TATWEEL    = 'ـ'

def strip(text):
    """For TRACKING: remove tashkeel and unify letter variants."""
    t = DIACRITICS.sub('', text).replace(TATWEEL, '')
    t = re.sub('[إأآٱ]', 'ا', t)
    t = t.replace('ى', 'ي').replace('ة', 'ه')
    return ' '.join(t.split())

def clean_vocalised(text):
    """For GRADING: keep tashkeel, but canonicalise its order and remove tatweel."""
    # Unicode allows shadda+fatha in either order; normalise to shadda first.
    ...
```

Write unit tests for these now. Normalisation bugs cause confusing scores later.

## 3. Tracking: where is the reader?

Tracking answers one question: **which expected word is the reader on?** It uses
`strip()`ed text, so tashkeel errors never affect it.

Algorithm (runs every time the partial hypothesis updates):

1. `hyp = strip(partial).split()` and `ref = strip(hadith).split()`.
2. Align `hyp` against a **window** of `ref`, from `cursor - 2` to `cursor + 15`,
   using word-level edit distance. Two words count as a match if
   `rapidfuzz.fuzz.ratio(a, b) ≥ 80`, which tolerates small spelling differences
   from the ASR.
3. Advance `cursor` to one past the last matched `ref` word.
4. A `ref` word that was passed over without a match is **pending**, not yet wrong.
   It becomes a **mistake** once **N = 2–3 later words have matched** past it. This
   delay prevents false alarms from ASR lag.
5. If nothing in the window matches for several seconds, the reader may have jumped
   ahead or started a different hadith. Widen the window to the whole hadith.

Output per word: `unread | read | pending | skipped` plus the time it was matched.

## 4. Grading: was each word said correctly?

For each `read` word, compare the recogniser's **vocalised** version against the
expected vocalised word:

| Result | Rule |
|---|---|
| `correct` | Vocalised forms match |
| `wrong_word` | Stripped forms differ beyond the fuzzy threshold |
| `tashkeel_error` | Stripped forms match, but diacritics differ |
| `uncertain` | The recogniser gave no diacritics for this word |

**At this stage, expect `tashkeel_error` to be unreliable.** Off-the-shelf models
often output no tashkeel at all, or a guessed one. Record the numbers anyway: they're
the baseline the custom model must beat. Pause rules (waqf etc.) come in
[Phase 5](phase-5-tashkeel-grading.md), so ignore end-of-word vowels before pauses
for now.

## 5. Scoring harness (`training/eval/score.py`)

This script is reused by **every later phase**, so build it carefully.

Input: golden-set labels + listener output. Report:

| Metric | Definition |
|---|---|
| **WER** | Word error rate of the final transcript vs. what was actually said (stripped) |
| **DER** | Diacritic error rate: character error rate on the diacritic sequence of correctly recognised words |
| **Tracking accuracy** | % of `read` words the listener marked as read, in the right place |
| **Word-mistake precision / recall** | Of flagged wrong/skipped words, how many were real; of real ones, how many were flagged |
| **Tashkeel-mistake precision / recall** | Same, for tashkeel errors |
| **False flags on clean reads** | Mistakes reported on the *clean* recordings. This should be ~0 |
| **Reveal latency** | Time from the word ending in audio to it being marked read (median, p90) |
| **Real-time factor** | Processing time ÷ audio duration on your laptop CPU |

Save each run as a dated JSON in `training/eval/results/` so you can compare runs.

## 6. Run and document the baseline

- Score every candidate model and record the results in a table in `docs/baseline.md`.
- Pick the **best off-the-shelf model for the Phase 2 app**, weighing tracking
  accuracy and latency above all.
- Tune `N` (mistake delay) and the fuzzy threshold on half of the golden readers, then
  report scores on the other half. Tuning on the full set overfits.

## Deliverables checklist

- [ ] `listener/` with `asr.py`, `text.py` (+ tests), `tracker.py`, `grader.py`
- [ ] `training/eval/score.py`, reusable by later phases
- [ ] `docs/baseline.md` with the scorecard for 2–3 models
- [ ] The model chosen for the Phase 2 app, with its tuned parameters

## What "good" looks like

Rough targets for an off-the-shelf model:
- Tracking accuracy **≥90%** on clean reads.
- Word-mistake precision **≥80%** (tune N to get there).
- Near-zero false flags on clean reads.
- Tashkeel metrics: whatever they are. Low numbers here are expected and justify Track B.

If tracking itself is poor (<80%), try a different base model before starting Phase 2.
