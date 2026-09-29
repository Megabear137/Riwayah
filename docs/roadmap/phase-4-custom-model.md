# Phase 4: Custom model v1

**Weeks 10–14 · Track B · [Back to roadmap](README.md)**

Fine-tune a speech model on the [Phase 3](phase-3-data-pipeline.md) dataset so it
outputs **fully vocalised Arabic** and hears what was actually said, including
tashkeel. Then shrink it to run on a phone.

Code and configs live in `training/model/`.

## 1. Choose the base model

**Recommended: NVIDIA FastConformer (CTC or hybrid transducer-CTC), fine-tuned
with NVIDIA NeMo.**

- NVIDIA has published Arabic FastConformer checkpoints, including variants trained
  with diacritics. Check the NVIDIA NGC and Hugging Face catalogues for the current
  names and their licenses (often CC-BY-4.0, which requires attribution).
- Why not Whisper: Whisper predicts text one word at a time with a built-in
  language model, so it tends to "correct" mistakes into the expected text. CTC
  decoding judges each moment of audio more independently, which is exactly what
  mistake detection needs.
- The encoder weights carry over, even if you change the output vocabulary.

**Streaming:** true streaming FastConformer checkpoints exist mainly for English and
multilingual models. For v1, it's fine to run a non-streaming model on short
overlapping chunks (1–2 s) or VAD segments. Measure the reveal latency (target:
median under ~1 s). Train a cache-aware streaming variant later only if latency is
a real problem.

## 2. Define the output vocabulary

Pick **one** and document it in `training/model/VOCAB.md`:

| Option | Pros | Cons |
|---|---|---|
| **Characters** (letters + diacritics + space) | Simple, directly supports diacritic-level grading and confidence, ~50 tokens | Longer output sequences |
| BPE on vocalised text | Shorter sequences, often slightly more accurate | Diacritics are hidden inside subword units, making grading harder |
| Phonemes (v2) | Handles pauses and connected speech naturally | Needs a text→phoneme converter (see [Phase 5](phase-5-tashkeel-grading.md)) |

**For v1 use characters.** Build the list from the training manifest, sort it and
freeze it. Every character in any label must be in the list, or the label is rejected.

```python
model = nemo_asr.models.ASRModel.from_pretrained("<arabic fastconformer checkpoint>")
model.change_vocabulary(new_vocabulary=chars)      # char-based CTC models
# BPE / hybrid models: build a tokenizer instead, then
# model.change_vocabulary(new_tokenizer_dir=..., new_tokenizer_type="bpe")
```

Changing the vocabulary resets the output layer, but the pretrained encoder is kept.

## 3. Assemble training data

- **Main:** your Phase 3 `manifest_train.jsonl`.
- **Optional extra speakers:** Quran recitation datasets that come with **vocalised**
  text (use a standard vocalised Imla'i script such as Tanzil's, not Uthmani
  orthography). Cap them at **~20–30%** of training hours, because recitation style
  (tajweed, elongation) differs from hadith reading.
- Don't mix in datasets with **unvocalised** labels. They teach the model to drop
  diacritics.
- **Splice augmentation (recommended):** using the Phase 3 word timings, build
  artificial clips by cutting and joining word segments to create **skipped,
  repeated or swapped words**, with labels that match the new audio. This directly
  teaches the model not to fill in the "expected" text. Where the same word appears
  with different i'rab in different clips, you can splice those in too, creating
  realistic tashkeel variations.

## 4. Augmentation

- **Speed perturbation** 0.9× / 1.0× / 1.1× (prepare offline, or use on-the-fly
  augmentors in NeMo).
- **Noise:** mix in background noise at 5–20 dB SNR (e.g. from MUSAN, checking its
  license).
- **Room echo:** convolve with room impulse responses.
- **Phone quality:** random low-pass filtering, compression, gain changes.
- SpecAugment (on by default in NeMo configs). Keep it moderate.

## 5. Train

- **Hardware:** one A100/H100 (40–80 GB) or an L40S. Rent it (RunPod, Lambda, Vast, etc.).
- **Script:** NeMo's example scripts (`examples/asr/asr_ctc/speech_to_text_ctc.py`
  or the hybrid equivalent) with `init_from_pretrained_model` pointing at the base
  checkpoint, plus your manifests.
- **Starting hyperparameters** (tune from here):
  - AdamW, peak LR 1e-4 to 3e-4, warmup ~1–2k steps, cosine or Noam decay
  - Duration-bucketed batches, bf16 mixed precision
  - Maximum clip duration 20 s
  - Run until validation DER stops improving, then keep the best checkpoint
- **Track experiments** (TensorBoard or Weights & Biases): log WER, DER, and loss on the
  validation set each epoch.
- **First run small:** 10% of the data for a few epochs, to verify that the
  pipeline works end to end before spending on the full run.

Rough cost: tens of GPU-hours per full run on a few hundred hours of audio, which
is tens to a few hundred dollars. Budget for 3–5 full runs.

## 6. Evaluate

Use `training/eval/score.py` from Phase 1, running the **full listener** (tracker +
grader) with the new model on:

1. **Golden test set** (Phase 0): the headline numbers, compared with the Phase 1
   baseline.
2. **Teacher corrections** (Phase 3): real tashkeel and word mistakes.
3. **Phase 3 test split**: WER and DER on unseen speakers.

### Memorisation check
Compare results on **hadith seen in training** vs. **held-out hadith** (same
unseen speakers):

- Similar accuracy on both = good.
- Much better on seen hadith, and **lower mistake recall on seen hadith** = the
  model is memorising text. Fixes: fewer epochs, more splice augmentation, more
  hadith variety, stronger regularisation.

### Scorecard (add to `docs/model-v1.md`)

| Metric | Baseline (Phase 1) | v1 | Target |
|---|---|---|---|
| Tracking accuracy (clean) | | | ≥ 95% |
| Word-mistake precision / recall | | | ≥ 90% / ≥ 80% |
| Tashkeel-mistake precision / recall | | | ≥ 70% / ≥ 50% (improved in Phase 5) |
| False flags on clean reads | | | ~0 |
| DER (test speakers) | | | as low as possible, tracked over time |
| Reveal latency (median) | | | < 1 s |

## 7. Export and run on device

1. Export to ONNX. The sherpa-onnx repo has scripts for exporting NeMo CTC models
   with the metadata and `tokens.txt` it expects. Use those, not a bare
   `model.export()`.
2. **Quantise** to int8 with `onnxruntime.quantization.quantize_dynamic`.
   Re-run the evaluation on the quantised model, because quantisation can cost accuracy.
3. Load it in the Phase 2 app behind a developer setting and measure:
   - **Real-time factor** (processing time ÷ audio time) on a mid-range Android phone
     and an older iPhone. Target **< 0.3**.
   - Model download size (target < ~150 MB), memory use, and battery drain over a
     20-minute session.
4. If it's too slow: try a smaller FastConformer size, a larger chunk step, or
   fewer threads (to stop thermal throttling).

## 8. Version and document

- Name the model `riwayah-asr-v1.0` and store it with its **model card**: base
  checkpoint and license, training data summary, vocabulary, scorecard,
  known weaknesses.
- Keep the exact manifests, config and code commit for every released model, so you
  can rebuild it.

## Decision point

- **v1 beats the baseline, and tashkeel precision is promising (≥ 70%)** → go to
  Phase 5 with this model.
- **Words are good but tashkeel is weak** → plan **v2 with phoneme output** in
  Phase 5, and possibly more vocalised data.
- **Words are no better than the baseline** → the data is the likely problem:
  revisit Phase 3 label quality and speaker variety before training again.

## Deliverables checklist

- [ ] Training configs + scripts in `training/model/`
- [ ] Frozen vocabulary in `VOCAB.md`
- [ ] Scorecard in `docs/model-v1.md`, including the memorisation check
- [ ] Quantised ONNX model running in the app with on-device measurements
- [ ] Model card and the decision recorded for Phase 5
