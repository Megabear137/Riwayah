# Voice listener research (Sept 2026)

Goal: a Tarteel-style "follow along" listener for hadith. The user recites, the app
recognises the Arabic speech, works out where they are in a known hadith text, and
reveals/checks the words as they go.

## 1. What TarteelAI publishes on GitHub

The [TarteelAI org](https://github.com/TarteelAI) has 15 public repos. None of them
contains the recognition engine the Tarteel app uses.

| Repo | What it actually is | License | Useful to us? |
|---|---|---|---|
| `voice` | Fork of `@react-native-voice/voice`, a wrapper around the phone's built-in speech APIs (Apple `SFSpeechRecognizer`, Android `SpeechRecognizer`). No Tarteel model. Last push Dec 2023. | MIT | Only as a thin OS-speech wrapper, which isn't good enough for classical Arabic (see §3) |
| `NeMo`, `Speech` | Unmodified forks of NVIDIA NeMo, the training toolkit Tarteel uses. No Tarteel code, configs or weights (grep for "tarteel"/"quran" finds nothing). | Apache-2.0 | Yes, as the toolkit for training our own model |
| `nemo2riva` | Fork of NVIDIA's NeMo → Riva model export tool | MIT | Only if we self-host on Riva |
| `tarteel-ml` | 2021-era experiments: DeepSpeech/Keras training scripts on the Tarteel dataset. Abandoned. | MIT | Historical reference only |
| `tkseem`, `tnkeeh` | Arabic tokenisation and text normalisation libraries | MIT | Yes, `tnkeeh` for normalising hadith text (diacritics, alef/ya variants) before matching |
| `quranic-universal-library` | Quran text, audio and metadata resources | MIT | No, it's Quran content |
| others | fastlane, celery-exporter, 1click-hpc, pinax-badges, quran-ttx, quran-assets, marafiq | – | No |

**How Tarteel's listener works (from public case studies):** a custom Arabic ASR model
trained in NVIDIA NeMo, optimised with TensorRT and served in real time via NVIDIA
Riva/Triton on GPU servers. The app streams mic audio to that server. The production
model and server are closed source.

**Open weights they did release:** on Hugging Face, `tarteel-ai/whisper-base-ar-quran`
and `tarteel-ai/whisper-tiny-ar-quran`, both Apache-2.0. They're Whisper models fine-tuned
on Quran recitation, so they're tuned to Quranic vocabulary and tajweed-style delivery.
Hadith is spoken more like normal Classical/MSA prose, with isnad name chains
("حدثنا فلان عن فلان…"), so these models are a starting point to fine-tune from, not
something to use as-is.

## 2. Key design insight: we don't need open-ended transcription

The app always knows **which hadith** the user is reciting. So the problem is
*"align speech to a known text"*, not *"transcribe arbitrary Arabic."* That changes
what we need:

1. Run ASR (streaming) → partial Arabic text **with tashkeel** (or phonemes, see §2a).
2. **Tracking:** make a stripped copy of both sides (no tashkeel; unify ا/أ/إ/آ,
   ى/ي, ة/ه; remove tatweel) and fuzzy-align it against the expected hadith words
   (e.g. a Levenshtein / Needleman-Wunsch alignment over words, or over characters for
   robustness), starting from the current cursor. This only decides *where the
   user is*, so it should tolerate tashkeel errors.
3. **Grading:** for each matched word, compare the full vocalised form (harakat,
   shadda, sukun, tanween) against the expected form, so tashkeel mistakes count as
   mistakes.
4. Reveal matched words; flag a word as a mistake only after N subsequent words match
   past it (avoids false negatives from ASR noise).

### 2a. Grading tashkeel

Tashkeel mistakes (e.g. i'rab errors) count as mistakes, which affects the model and
the grader:

- **Model output must include tashkeel.** Train on fully vocalised labels with a
  character vocabulary that includes the diacritics. A better option is to predict
  *pronunciation* (a phoneme sequence with explicit short vowels), which is what Quran
  mispronunciation-detection research does.
- **Compare pronunciation, not spelling.** Generate the expected pronunciation from the
  vocalised text with rules for hamzat al-wasl, sun letters, and **waqf**: stopping on a
  sukun at a pause is correct, not a missing case ending. Comparing raw diacritics
  would flag these as false mistakes.
- **No language-model help on tashkeel.** Use CTC-style decoding without text biasing
  or an external LM, otherwise the model "corrects" the user's i'rab to the expected
  one.
- **Report confidence.** Short vowels are acoustically subtle, so diacritic errors will
  always be less reliable than whole-word errors. Grade them separately (e.g. "check
  this vowel" vs. "wrong word") and consider a strict/lenient setting.
- **Evaluate separately:** word error rate and diacritic error rate, plus
  precision/recall on real tashkeel mistakes (teacher corrections in the training
  videos are a good source).
- **Labels need consistent full tashkeel.** Many hadith databases are only partly
  vocalised. Prefer a fully vocalised edition that matches what's shown on screen, and
  review any gaps filled by an automatic diacritiser (e.g. CAMeL Tools).

Even a mediocre general Arabic model becomes quite usable with this, because we only
need to answer "did they say roughly the next expected word?" Several open-source Quran
trackers do exactly this (see §5).

Optional upgrade: bias decoding toward the expected text, using hotwords/contextual
biasing in sherpa-onnx or a Whisper `initial_prompt` containing the hadith text.
Be careful: heavy biasing can make the model "hear" the correct text even when the user
got it wrong, which defeats memorisation testing. Prefer alignment-after-decoding for
mistake detection.

## 3. Options for the recogniser

### A. OS built-in speech APIs: free, weakest for this

- iOS `SFSpeechRecognizer` / iOS 26 `SpeechAnalyzer`, Android `SpeechRecognizer`,
  browser Web Speech API.
- Free and zero model shipping. But they're tuned for dialect and modern dictation, have
  poor accuracy on classical text and names, and often need a network connection.
  Developers report Arabic listed but not downloadable for iOS 26 `SpeechTranscriber`.
  They also auto-correct toward common words.
- **Use only for a quick prototype**, e.g. via `@react-native-voice/voice`.

### B. On-device open models: free, recommended

| Engine | License | Notes |
|---|---|---|
| **[sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx)** (next-gen Kaldi + onnxruntime) | Apache-2.0 | True streaming, VAD, hotword biasing. Runs Whisper, NeMo CTC/transducer, Zipformer models. Official Flutter package + community React Native TurboModule. Best fit for a live "follow along" UX. |
| **[whisper.cpp](https://github.com/ggml-org/whisper.cpp)** | MIT | Runs any Whisper incl. Tarteel's fine-tunes (convert to GGML). Whisper isn't natively streaming: you run sliding windows, so latency and battery are worse. Fine for check-after-each-sentence mode. |
| **WhisperKit** (Argmax) | MIT | Optimised Whisper for Apple devices (CoreML), with streaming support |
| **Vosk** | Apache-2.0 | Small, streaming, has an Arabic model, but accuracy is dated |

Everything runs offline, which also fits the "download collections and practise
offline" feature. Models are ~40–150 MB and can be downloaded on first use like the
hadith collections.

### C. Self-hosted server (Tarteel's approach): free software, you pay for compute

Stream audio over WebSocket to a Python server (FastAPI + sherpa-onnx or
faster-whisper, or NeMo + Riva on GPU). This gives the best accuracy with large models
and lets you update models without app releases. The costs are GPU hosting, latency,
and needing a network connection.

### D. Paid cloud STT: not free

Google Cloud Speech (Chirp), Azure Speech, AWS Transcribe, OpenAI transcription, and
ElevenLabs all support Arabic and are billed per minute. They're easy to use, but cost
scales with usage, audio leaves the device, and they have the same classical-Arabic
weaknesses as option A unless you use custom-vocabulary features.

## 4. Recommended path

1. **Prototype (weeks):** React Native or Flutter + **sherpa-onnx** running a
   multilingual/Arabic Whisper (or `tarteel-ai/whisper-base-ar-quran` exported to ONNX),
   with the normalise → align → reveal pipeline from §2. Validate on ~20 hadith read by
   a few people.
2. **Collect data:** with consent, let users opt in to donate recordings of hadith
   they recite. Since the target text is known, every recording is automatically
   labelled. This is how Tarteel built its dataset.
3. **Fine-tune:** a small streaming model (NeMo FastConformer CTC/transducer or Whisper
   tiny/base) on that data plus public Arabic data (Common Voice Arabic, MGB-2, Quran
   datasets), using NVIDIA NeMo (Apache-2.0, the same toolkit Tarteel uses). Export to
   ONNX and ship via sherpa-onnx on device.
4. Only add a server tier if on-device accuracy plateaus.

## 5. Open-source projects worth reading

These are Quran-focused, but the alignment logic carries over to hadith. Check each
repo's license before reusing code.

- [yayaiu6/Real-Time-Quran-recitation-tracker-System](https://github.com/yayaiu6/Real-Time-Quran-recitation-tracker-System): real-time word-level alignment and error detection
- [blendonl/quran](https://github.com/blendonl/quran): Expo app streaming mic audio over WebSocket to a FastAPI server with VAD and phoneme models. Closest architecture to Tarteel.
- [zlatif1-source/ayahflow](https://github.com/zlatif1-source/ayahflow): follows ayah to ayah and highlights words
- [sayedmahmoud266/quran-ai-transcriping](https://github.com/sayedmahmoud266/quran-ai-transcriping): verse matching with constraint propagation on top of Tarteel Whisper
- [AbdirahmanNomad/IqraAI](https://github.com/AbdirahmanNomad/IqraAI): built on Tarteel Whisper

## 6. Hadith text sources (for the "download collections" feature)

- [fawazahmed0/hadith-api](https://github.com/fawazahmed0/hadith-api): JSON editions
  (Bukhari, Muslim, Abu Dawud, etc.), Arabic with and without diacritics, plus
  translations and grades. Served from a CDN. The repo's LICENSE is the Unlicense (public
  domain). Individual translations may carry their own copyright, so check them before
  commercial use.
- sunnah.com API: needs an API key and has its own terms.

## Sources

- TarteelAI GitHub: https://github.com/TarteelAI
- NVIDIA case study: https://www.nvidia.com/en-us/case-studies/automating-real-time-arabic-speech-recognition/
- NVIDIA blog: https://developer.nvidia.com/blog/exploring-unique-applications-of-automatic-speech-recognition-technology/
- Tarteel Whisper model: https://huggingface.co/tarteel-ai/whisper-base-ar-quran
- sherpa-onnx: https://github.com/k2-fsa/sherpa-onnx, React Native: https://github.com/XDcobra/react-native-sherpa-onnx, Flutter: https://pub.dev/packages/sherpa_onnx
- whisper.cpp: https://github.com/ggml-org/whisper.cpp, WhisperKit paper: https://arxiv.org/html/2507.10860v1
- Apple SpeechAnalyzer Arabic issue: https://developer.apple.com/forums/thread/797835
