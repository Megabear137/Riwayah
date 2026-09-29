# Phase 0: Foundations

**Weeks 0–2 · Both tracks · [Back to roadmap](README.md)**

Everything later depends on four things settled here: the app framework, the hadith
text, permission to use the data, and a fixed test set for measuring every model.
Skipping any of them causes rework later.

## 1. Choose the app framework

**Recommendation: Flutter.**

| | Flutter | React Native |
|---|---|---|
| On-device speech (sherpa-onnx) | Official package (`sherpa_onnx` on pub.dev), maintained with the engine | Community TurboModule only |
| Arabic / RTL text | Good. Text shaping and tashkeel rendering are solid with the right font | Good |
| Team familiarity | Dart | JavaScript/TypeScript |

Choose React Native only if your team is much stronger in JavaScript. In that case,
build a small test with the community sherpa-onnx module first, and confirm it can
stream microphone audio on both iOS and Android before you commit.

**Font:** pick an Arabic font that renders full tashkeel clearly at small sizes
(e.g. Amiri, Scheherazade New or KFGQPC-style naskh fonts, checking each license).
Test a heavily vocalised hadith on a small phone screen.

## 2. Choose the hadith text source

Requirements:
- **Fully vocalised**: every letter carries its harakah, shadda, sukun or tanween.
  Tashkeel grading is impossible without it.
- Consistent numbering (so hadith IDs are stable across app releases).
- A license that allows your use, including commercial use if you plan to charge.

**Start with one small collection**, e.g. al-Arba'in an-Nawawiyyah (42 hadith)
or a Riyad as-Salihin selection. It's small enough to check by hand and widely
memorised, and it probably overlaps with what's read in the classroom videos.

Candidate sources:
- [`fawazahmed0/hadith-api`](https://github.com/fawazahmed0/hadith-api) (public domain
  code/data; check the license of each translation separately).
- Printed/digital editions the teacher in the videos uses. **Matching the on-screen
  edition is valuable** because Phase 3 labels will line up exactly.

### Measure vocalisation coverage

Don't trust "has tashkeel" claims. Measure the percentage of Arabic letters followed by a
diacritic:

```python
import re
LETTER = re.compile(r'[ء-غف-ي]')
DIAC   = set(chr(c) for c in range(0x064B, 0x0653)) | {'ٰ'}

def coverage(text: str) -> float:
    letters = vocalised = 0
    for i, ch in enumerate(text):
        if LETTER.match(ch):
            letters += 1
            if i + 1 < len(text) and text[i + 1] in DIAC:
                vocalised += 1
    return vocalised / max(letters, 1)
```

Some letters are legitimately bare, such as the alif in long vowels and the lam in
al- before sun letters. So aim for **≥85% coverage**, then spot-check 10 hadith
by eye. Where coverage is low, either choose another edition or vocalise the gaps
with CAMeL Tools and **have someone qualified review the result**.

### Store it in a canonical format

One JSON file per collection, with stable IDs and the isnad/matn split marked (the app
will let users practise them separately):

```json
{
  "collection": "nawawi40",
  "edition": "source + version",
  "hadith": [
    {
      "id": "nawawi40:1",
      "isnad": "عَنْ أَمِيرِ الْمُؤْمِنِينَ أَبِي حَفْصٍ عُمَرَ بْنِ الْخَطَّابِ ...",
      "matn": "إِنَّمَا الْأَعْمَالُ بِالنِّيَّاتِ ...",
      "translation_en": "...",
      "grade": "sahih",
      "reference": "Bukhari 1, Muslim 1907"
    }
  ]
}
```

## 3. Confirm rights and consent

Get these **in writing**, before any training:

- **Classroom videos:** permission from the owner (institution or teacher) to use the
  recordings for training a commercial or non-commercial model, as applicable.
- **People in the videos:** confirm students and teacher consented to recording and
  this use. If any are **minors**, get guardian consent and take extra care (e.g.
  never ship raw audio, and keep the data access-controlled).
- **Hadith text and translations:** license terms for each.
- **Base models:** note the license of each checkpoint you'll fine-tune (Whisper:
  MIT. NVIDIA NeMo checkpoints: often CC-BY-4.0, which requires attribution).

Keep a `docs/legal/` note summarising what was agreed and with whom. Don't commit the
signed documents themselves.

## 4. Build the golden test set

This is the single most important deliverable of Phase 0. Every model and every
change to the listener is scored against it, so **it must not change once built**.

### Who and what
- **5–10 readers who do NOT appear in the training videos.** Mix genders, ages,
  and fluency (confident memorisers and learners).
- **20–30 hadith** from your starting collection, including long isnads with many names.
- Each reader records each hadith **twice**:
  1. **Clean read:** their best reading.
  2. **Mistake read:** reading with 3–5 **planned** mistakes from a script you
     give them.

### Mistake script categories
Plan the mistakes so every category is covered across the set:

| Category | Example |
|---|---|
| Skipped word | Drop a word from the matn |
| Extra/repeated word | Say a word twice or insert one |
| Swapped words | Reverse two adjacent words |
| Wrong word | Substitute a similar-sounding word |
| Wrong isnad name | Swap a narrator's name |
| I'rab error | Wrong case ending (e.g. -u → -a) |
| Internal vowel error | Wrong harakah inside a word |
| Shadda error | Drop or add a shadda |
| **Correct pause (control)** | Stop on sukun at a natural pause. This must **NOT** be flagged |

### Recording
- Record on phones (the real use case), in ordinary rooms. Include a few with
  background noise.
- Save as WAV, 16 kHz or higher, mono.
- File naming: `golden/{reader_id}/{hadith_id}_{clean|mistake}.wav`

### Labelling
For each mistake read, write down **what was actually said** (fully vocalised) plus a
list of mistakes:

```json
{
  "audio": "golden/r03/nawawi40:1_mistake.wav",
  "hadith_id": "nawawi40:1",
  "said": "إِنَّمَا الْأَعْمَالَ بِالنِّيَّاتِ ...",
  "mistakes": [
    {"word_index": 1, "type": "irab", "expected": "الْأَعْمَالُ", "said": "الْأَعْمَالَ"}
  ]
}
```

Have a second person who knows Arabic check the labels. Planned mistakes aren't
always said as planned, and readers make unplanned ones too.

Keep this data out of git (store it privately, e.g. in a bucket), and never use it for training.

## 5. Set up the repository

- Create the layout from the [roadmap overview](README.md#repo-layout-target).
- Add `data/` to `.gitignore`. Audio and video never go in git.
- Pick a Python version (3.11+) and add a `training/requirements.txt`.
- Pick shared storage for large files (cloud bucket or a NAS) and document the paths.

## Deliverables checklist

- [ ] Framework decision recorded in `docs/decisions.md`
- [ ] Starting collection chosen, vocalisation coverage measured (≥85%), stored in canonical JSON
- [ ] Written permissions for videos, text, and translations; licenses of base models noted
- [ ] Golden test set: 5–10 readers × 20–30 hadith × (clean + mistake), labelled and double-checked
- [ ] Repo skeleton with `.gitignore` for data

## Pitfalls

- **Using training-video readers in the golden set.** Scores will look better than
  real-world performance. Keep them strictly separate.
- **Changing the golden set later.** Scores become incomparable across phases. If
  you must extend it, add a *new* set and keep reporting on the original.
- **Assuming the text is fully vocalised.** Measure it.
