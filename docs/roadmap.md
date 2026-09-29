# Riwayah roadmap

A hadith memorisation app: download collections, recite from memory, and have the app
follow along, revealing words and flagging mistakes (including tashkeel).
Background research is in [`research/voice-listener.md`](research/voice-listener.md).

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

---

## Phase 0: Foundations (weeks 0–2)

Decisions and assets that everything else depends on.

- **Pick the app framework.** Flutter is recommended: sherpa-onnx (the on-device speech
  engine) has an official Flutter package, while React Native only has a community
  module.
- **Pick the hadith text source.** It must be **fully vocalised** and consistent.
  Start with one collection (e.g. the Arba'in an-Nawawi or Riyad as-Salihin selections),
  not all six books.
- **Confirm rights.** Get written permission to use the classroom videos for model
  training, and confirm the license of the hadith text and any translations.
- **Build the golden test set.** Record 20–30 hadith read by 5–10 people who are
  *not* in the training videos (different ages, genders and fluency), including
  deliberate mistakes: skipped words, swapped words, wrong names in the isnad, and
  i'rab/tashkeel errors. Label every mistake. Every model in the project is measured
  on this set, so build it first and don't change it.
- **Set up the repo:** `app/`, `listener/` (tracking and grading logic), `training/`
  (data pipeline and training), `docs/`.

**Done when:** the framework and text source are chosen, rights are confirmed, and the
golden test set is recorded and labelled.

## Phase 1: Listener prototype (weeks 2–4) · Track A

Prove the core idea on a laptop before building the app.

- A Python script that takes an audio file and a hadith ID and outputs which words
  were read and which were wrong.
- Use an off-the-shelf Arabic model through sherpa-onnx (plus Tarteel's Whisper
  models for comparison).
- Implement the two-pass logic: **tracking** on text without tashkeel, **grading**
  on the full vocalised form.
- Run it on the golden test set and record baseline numbers: word tracking accuracy,
  word-mistake precision/recall, tashkeel-mistake precision/recall, and latency.

**Done when:** you have a baseline scorecard. Expect word tracking to be decent and
tashkeel grading to be poor. That gap is the reason for Track B.

## Phase 2: App MVP (weeks 4–10) · Track A

A usable app with the baseline model, **word-level feedback only**.

- **Library:** browse and download collections for offline use, with Arabic text and
  translation.
- **Memorise mode:** hide the text, reveal words as the user recites, and give hints
  on request. A word is flagged as a mistake only after several later words have
  matched, which prevents false alarms.
- **Practice tools:** repeat a hadith, split it into chunks (isnad / matn), and
  schedule reviews with spaced repetition.
- **Progress:** which hadith are memorised, and mistake history.
- **Opt-in recording donation** with a clear consent screen. Donated recordings are
  automatically labelled because the app knows which hadith was read.
- Tashkeel grading stays off or marked "beta" until Phase 5.
- **Closed beta** with 10–30 users (e.g. the students from the videos, with permission).

**Done when:** beta users can memorise a collection end to end, and you have feedback
on the experience.

## Phase 3: Data pipeline (weeks 3–10) · Track B

Turn the classroom videos into clean training clips.

1. Extract audio (16 kHz mono) and the frames where the screen changes.
2. Read the on-screen hadith with Arabic OCR, match it to the hadith database, and use
   the **database's clean, fully vocalised text** as the label. Output: which hadith
   is on screen, and when.
3. Roughly transcribe each window and keep only the stretches that match the hadith
   (the student reading), dropping the teacher's lecture. Add speaker detection if the
   teacher also reads.
4. Force-align the reading to the vocalised text to get word timings and confidence
   scores. Drop low-confidence spans, since those are usually the student's own mistakes.
5. **Save teacher corrections separately** as a real-world mistake test set.
6. Cut 5–20 second clips and write the training manifest. Split by student, never by
   clip.
7. Listen to 1–2% of clips and tighten thresholds until labels are reliable.

**Done when:** you have a dataset report (usable hours, number of distinct readers,
spot-check error rate below ~2%) and a teacher-corrections test set.

## Phase 4: Custom model v1 (weeks 10–14) · Track B

- Fine-tune an Arabic **FastConformer** checkpoint in NVIDIA NeMo, with a
  character vocabulary **including tashkeel**. Use CTC decoding with no text
  biasing, so the model reports what was said rather than what was expected.
- Add augmentation: background noise, phone-quality audio, slight speed changes.
- Measure it on the golden set and the teacher-corrections set: word error rate,
  **diacritic error rate**, and mistake precision/recall. Compare against the
  Phase 1 baseline.
- Check for memorisation by comparing results on hadith that were in training vs.
  held out.
- Export to ONNX, quantise to int8, and test real-time speed and battery use on a
  mid-range Android phone and an older iPhone.
- Budget: a rented A100/H100-class GPU, tens to a few hundred dollars per full
  training run.

**Done when:** v1 clearly beats the baseline on the golden set and runs in real time on
a mid-range phone. **Decision point:** if tashkeel accuracy is still weak, plan v2 with
pronunciation (phoneme) output in Phase 5.

## Phase 5: Integration and tashkeel grading (weeks 14–20) · Tracks A+B

- Ship the custom model in the app, downloaded on first use like collections.
- Add pronunciation rules so correct readings aren't flagged: **waqf** (stopping on
  sukun), hamzat al-wasl, and sun letters. Grade pronunciation, not spelling.
- Optionally train v2 with phoneme output if Phase 4 showed diacritics were the weak
  point.
- **Feedback UI:** show "wrong word" (high confidence) separately from "check this
  vowel" (lower confidence), and add a **strict / lenient** setting.
- Tune thresholds so tashkeel flags are right most of the time (e.g. ≥85% precision).
  Users stop trusting the app quickly if it cries wolf.

**Done when:** tashkeel grading meets the precision target on the golden set, and beta
users rate the feedback as trustworthy.

## Phase 6: Launch and improvement loop (week 20+)

- Public release on the App Store and Play Store.
- **Improvement loop:** consented user recordings, then automatic labelling, then
  retraining every few months, then an updated model shipped to users. Always
  re-check against the golden set before shipping.
- Add more collections and translations.
- Consider a server-side model only if on-device accuracy stops improving.

---

## Key risks

| Risk | Mitigation |
|---|---|
| The model "hears" the correct text over real mistakes (memorisation) | CTC decoding without text biasing, hold out whole hadith from training, and measure with deliberate-mistake test sets |
| Tashkeel errors are too subtle to detect reliably | Show confidence levels and a strict/lenient setting; switch to phoneme output (v2) if needed |
| Too few distinct readers in the videos | Track speaker count in the Phase 3 report, and top up with donated recordings from the beta |
| Hadith text isn't fully or consistently vocalised | Choose the edition in Phase 0, and review any automatic diacritisation |
| Rights or consent problems with videos or text | Resolve in Phase 0, before any training |
| Model too slow or large for phones | Smaller model, int8 quantisation, and on-device testing in Phase 4, not at launch |

## Fixed costs to expect

- GPU rental for training: tens to a few hundred dollars per run.
- Apple Developer Program: $99/year. Google Play: $25 one-time.
- Hosting for model and collection downloads: a CDN or object storage (low cost at small scale).
