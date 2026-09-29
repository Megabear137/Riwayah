# Phase 5: Integration and tashkeel grading

**Weeks 14–20 · Tracks A+B · [Back to roadmap](README.md)**

Ship the [Phase 4](phase-4-custom-model.md) model in the app and turn on tashkeel
grading, but only at a level users can trust. The two hard parts are:

1. **Not flagging correct readings.** A correct reading can differ from the written
   text: pausing on a sukun, dropping hamzat al-wasl, merging sun letters, etc.
2. **Expressing uncertainty.** Short vowels are subtle. The app must be honest when it
   isn't sure.

## 1. Ship the model in the app

- Host model files next to the collections, with a small index:
  `{version, url, sha256, size, min_app_version}`.
- Download on first use (or when the user enables voice practice). Verify the
  checksum and keep the previous version for rollback.
- **Remote switch:** a config value that selects the active model version. It lets you
  roll back a bad model without an app release.
- Keep the Phase 2 off-the-shelf model as a fallback for devices that are too slow.

## 2. Expected pronunciation instead of spelling

Build `listener/pronounce.py` (then port it to Dart). For each word in the hadith, it
generates the **set of acceptable pronunciations**, depending on its context.

### Rules to implement (v1)

| Rule | Written | Acceptable when pausing after this word | Acceptable when joined to the next word |
|---|---|---|---|
| **Waqf: final short vowel** | كِتَابُ | كِتَابْ (sukun) | كِتَابُ |
| **Waqf: tanween damm/kasr** | رَجُلٌ | رَجُلْ | رَجُلٌ |
| **Waqf: tanween fath** | كِتَابًا | كِتَابَا (long ā) | كِتَابًا |
| **Waqf: ta marbuta** | رَحْمَةٌ | رَحْمَهْ | رَحْمَةٌ |
| **Hamzat al-wasl** | قَالَ ابْنُ | (start of phrase: ibnu) | qāla bnu: the hamza and its vowel are dropped |
| **Sun letters** | الشَّمْسُ | ash-shams | lam silent, letter doubled |
| **Two sukuns meeting** | فِي الْمَسْجِدِ | – | fi l-masjid: the long vowel is shortened |

Rules are usually written into fully vocalised text (e.g. the helping kasra when
two sukuns meet), but check your edition. Add rules only when the golden set or
teacher corrections show that they matter. **Have someone qualified in Arabic review
the rule list.**

### Pause detection
The listener already knows each word's end time. Treat a gap of **≥ 300 ms** (tune
this) after a word as a pause, and accept that word's pausal forms. Pausing mid-phrase
isn't a tashkeel mistake. If you want to flag a badly placed pause, make it a separate,
optional check.

### Narration variants
Some hadith have known variants in wording or i'rab. Store accepted alternatives in the
hadith data (`"variants": {"word_index": ["alt form", ...]}`), and never flag them.

## 3. Two ways to compare

**Option A: characters (v1 model).** Compare the recognised vocalised word against
every acceptable written form from the rules. The best match decides the result.
Simplest, and good enough if Phase 4's tashkeel numbers were promising.

**Option B: phonemes (v2 model).** If Phase 4 showed tashkeel was weak:
1. Write a converter from vocalised text to phonemes. Vocalised Arabic maps to sounds
   almost perfectly, so this is rule-based and deterministic: short vowels, long vowels,
   shadda = doubled consonant, tanween = vowel + n, plus the rules above.
2. Convert the training labels from Phase 3 to phonemes and retrain (Phase 4
   steps 2–7 with the phoneme vocabulary).
3. Compare phoneme sequences directly. Pausal and connected-speech forms come out
   naturally.

## 4. Confidence

For each diacritic the model outputs, take the **CTC posterior probability** (how
sure the model is, per time frame) of the recognised token vs. the expected one.

- `tashkeel_error` (**confident**): the recognised diacritic is clearly more likely
  than the expected one (e.g. margin ≥ 0.3).
- `tashkeel_uncertain`: it's unclear either way. Show it softly, or not at all in
  lenient mode.
- `correct`: the expected diacritic wins.

**Calibrate thresholds** on half of the golden readers + teacher corrections, and
report the final numbers on the other half. The target is tashkeel-mistake
**precision ≥ 85%** at the standard setting. Recall can be lower: missing a
mistake is better than a false alarm.

## 5. Feedback UI

- **Word states:** read (normal), wrong/skipped word (red), tashkeel error
  (amber, with the affected letter highlighted), uncertain (dotted underline).
- **Tap a flagged word** to see the expected vs. heard forms, play **their own
  audio** for that word (from the attempt buffer), and play a reference reading if
  available.
- **Strictness setting:**
  - **Lenient:** words only.
  - **Standard** (default): words + confident tashkeel errors.
  - **Strict:** also shows uncertain tashkeel.
- **"This was wrong" button** on any flag. It records a false-flag report (with audio,
  if the user has opted in). These reports are the most valuable data for Phase 6.

## 6. Beta test the grading

- Enable it for beta testers first, and compare false-flag reports between models
  (remote switch).
- Ask a teacher to do a few sessions with students **alongside** the app and note
  every disagreement. Those disagreements go into a new test set.
- Measure how often each strictness level is used. If most users switch to lenient,
  tashkeel precision isn't high enough yet.

## Deliverables checklist

- [ ] Model distribution with checksum, versioning, remote switch, and fallback
- [ ] `pronounce.py` / Dart port with the reviewed rule list and tests
- [ ] Pause detection + narration variants in the grader
- [ ] Calibrated confidence thresholds (≥ 85% tashkeel precision at standard)
- [ ] Feedback UI with playback, strictness setting, and "this was wrong" reports
- [ ] (If needed) v2 phoneme model trained and evaluated
- [ ] Updated scorecard in `docs/model-v1.md` / `model-v2.md`

## Pitfalls

- **Flagging pauses as i'rab mistakes.** This is the most common false alarm. The
  golden set's "correct pause" control readings must score clean.
- **Tuning thresholds on the same data you report.** Always keep half aside.
- **Turning on tashkeel grading for everyone at once.** Roll it out gradually with
  the remote switch.
