# Phase 3: Data pipeline

**Weeks 3–10 · Track B · [Back to roadmap](README.md)**

Turn the classroom videos (a student reads a hadith shown on screen, and the teacher
sometimes lectures) into clean, labelled training clips. **This is most of the
work in building the custom model**, and label quality here decides how good the
model can be.

Code lives in `training/data_prep/`, one script per stage. Each stage reads the
previous stage's output, so any stage can be re-run on its own.

```
videos/ ─► 1 extract ─► 2 screen OCR ─► 3 find reading ─► 4 align ─► 5 cut clips ─► manifest
                                             │                  │
                                             └─► lecture (discard)
                                                                └─► 6 teacher corrections (test set)
```

## Stage 0: Inventory

Before processing anything, build `inventory.csv`: one row per video, with duration,
date, class/teacher, and which students appear (if known). You'll need the student
list to split training and test data by speaker.

## Stage 1: Extract audio and screen changes

```bash
# Audio: 16 kHz mono WAV
ffmpeg -i video.mp4 -vn -ac 1 -ar 16000 audio/{video_id}.wav

# Frames only when the screen changes significantly, with their timestamps
ffmpeg -i video.mp4 -vf "select='gt(scene,0.2)',showinfo" -vsync vfr \
       frames/{video_id}/%05d.png 2> frames/{video_id}/showinfo.log
```

Parse `pts_time` from `showinfo.log` to get each frame's timestamp. Tune the `0.2`
scene threshold on a few videos: too high misses slide changes, and too low
captures every camera movement. If the text region is a fixed position on screen,
crop to it first so camera movement doesn't trigger false changes.

## Stage 2: Identify the hadith on screen

1. **OCR** each changed frame with an Arabic-capable engine (e.g. PaddleOCR with
   `lang='ar'`, or Surya). Crop to the text region for better accuracy.
2. **Match** the OCR text against your hadith database:
   - `strip()` both sides (from Phase 1).
   - Use `rapidfuzz.process.extractOne` with `fuzz.token_set_ratio`, which tolerates
     OCR errors and partial text.
   - Accept a match at score ≥ 80. Put lower scores on a review list.
3. **Build time windows:** consecutive frames matching the same hadith become one
   window: `(video_id, hadith_id, start_s, end_s)`.
4. If a slide shows only **part** of a hadith, record which words are visible.
   Otherwise the student may be reading text the label doesn't include.

**The label is the database's clean, fully vocalised text, never the OCR output.**
OCR only tells you *which* hadith is on screen.

Hadith that don't match anything (not in your database) go to `review/unmatched.csv`.
Either add them to the database (vocalised and checked) or skip them.

## Stage 3: Separate reading from lecture

Inside each window, both the student reading and the teacher lecturing may be heard.

1. Run **voice activity detection** (VAD, e.g. Silero VAD or sherpa-onnx's VAD)
   to split the window into speech segments.
2. Transcribe each segment with the best off-the-shelf model from Phase 1.
3. Score each segment against the hadith text: the proportion of its stripped words
   that fuzzy-match words from the hadith, **in order**.
4. Segments scoring high (≥ 0.6) are **reading**. Low scores are **lecture**, and
   are discarded.
5. Merge adjacent reading segments.

**If the teacher also reads the hadith aloud** (e.g. modelling it first), add
speaker diarisation (pyannote or sherpa-onnx speaker embeddings) and tag each
reading segment as `student` or `teacher`. Both are useful training data, but the
speaker must be known for the train/test split.

## Stage 4: Force-align words to audio

Goal: a start time, end time and confidence score for **each word** of the reading.

- **First pass** (before your own model exists): use a CTC forced aligner, e.g.
  [`ctc-forced-aligner`](https://github.com/MahmoudAshraf97/ctc-forced-aligner)
  (multilingual, handles Arabic) or torchaudio's `forced_align` with an Arabic
  CTC model. Align against the **stripped** text, because these off-the-shelf aligners
  can't judge tashkeel.
- **Later passes** (after Phase 4 v1): re-align with your own model against the
  **vocalised** text. That gives per-character confidence, including on diacritics.

Handling the student's own mistakes:
- A student who misreads a word produces a **low-confidence** stretch there.
- Mark words below a confidence threshold as `suspect`.
- Clips containing a `suspect` word are **not** used for training, because their
  labels would be wrong.
- Tune the threshold by listening: sample 30 suspect words and 30 non-suspect
  words, and check how many are real misreads.

## Stage 5: Cut clips and write the manifest

- Cut at **word boundaries** (at a pause between words where possible) into clips
  of **5–20 seconds**.
- Pad each clip with ~100 ms of audio on either side so first and last sounds aren't
  cut off.
- Skip clips containing suspect words or overlapping speech.
- Write a JSON Lines manifest (NeMo's format, also easy to load in Hugging Face):

```json
{"audio_filepath": "clips/v012_nawawi40-1_003.wav", "duration": 11.4,
 "text": "إِنَّمَا الْأَعْمَالُ بِالنِّيَّاتِ", "speaker": "s07",
 "video_id": "v012", "hadith_id": "nawawi40:1", "role": "student"}
```

**Label text:** fully vocalised, with Unicode order normalised (shadda before
harakah), no tatweel and no punctuation. Use one consistent convention everywhere.

### Split by speaker

- **Train / validation / test = ~80 / 10 / 10 by speaker**, never by clip.
- Also hold out **a few whole hadith** from training. This is how Phase 4 detects
  whether the model is memorising text instead of listening.
- Write the split lists to `splits/*.txt` and never shuffle them afterwards.

## Stage 6: Teacher-corrections test set

When a student misreads and the teacher corrects them, you get a **real mistake**
recording:

1. Find `suspect` spans followed within a few seconds by teacher speech (lecture or
   a different speaker) that repeats the correct words.
2. Put these on a review list. A person listens to each and writes down what the
   student actually said, fully vocalised, plus the mistake type (use the categories
   from the [Phase 0 golden set](phase-0-foundations.md#mistake-script-categories)).
3. Save as `teacher_corrections.jsonl`, and **never train on it**.

Expect this to be slow, careful work. Even 100–200 labelled real mistakes (weighted
towards i'rab and tashkeel) is a valuable test set.

## Stage 7: Quality report

Produce `data_report.md` after each full run:

- Total video hours → speech hours → reading hours → **clean clip hours**
- Number of **distinct speakers** in each split, and hours per speaker (watch for
  one student dominating)
- Hadith coverage: how many distinct hadith, and clips per hadith
- Clips rejected at each stage, and why
- **Spot-check:** listen to a random 1–2% of clips (at least 200) and record how many
  labels are wrong. Target **< 2%**. If it's higher, tighten thresholds and re-run.

## Deliverables checklist

- [ ] `training/data_prep/` stages 1–6, each re-runnable on its own
- [ ] `manifest_{train,val,test}.jsonl` with speaker-based splits and held-out hadith
- [ ] `teacher_corrections.jsonl` (human-verified, never trained on)
- [ ] `data_report.md` with hours, speakers, rejection reasons, and spot-check error rate

## Pitfalls

- **Using OCR text as labels.** Always map to the clean database text.
- **Label drift from the student's mistakes.** A student's misreading, labelled as
  the correct text, teaches the model to "hear" correct text. Be strict with
  suspect spans. Losing hours is better than training on wrong labels.
- **Splitting by clip instead of speaker.** Test scores look great, then real users
  get worse results.
- **Too few distinct speakers.** Hours from one student are worth far less than hours
  from many. If speakers are few, prioritise donated recordings from the Phase 2 beta.
