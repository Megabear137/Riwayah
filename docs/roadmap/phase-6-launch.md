# Phase 6: Launch and improvement loop

**Week 20+ · Both tracks · [Back to roadmap](README.md)**

Release publicly, then improve the model continuously using consented user
recordings, without ever shipping a model that's worse than the last one.

## 1. Pre-launch checklist

### Store requirements
- [ ] **Privacy policy** (hosted URL) covering microphone use, on-device processing,
      optional recording donation, retention period, and deletion
- [ ] Apple **privacy nutrition label** and Google Play **Data safety** form, matching
      what the app actually does
- [ ] Microphone permission text explaining why it's needed
- [ ] Age rating questionnaire. If children may use the app, check the store and
      legal requirements for children's data before enabling donation for them
- [ ] Store listing in Arabic and English: screenshots of memorise mode and feedback

### Content
- [ ] Hadith texts and translations reviewed by someone qualified, with editions and
      sources credited in the app
- [ ] License attributions (e.g. CC-BY model checkpoints, fonts, data sources) on an
      "About / Credits" screen

### Operations
- [ ] Crash reporting and basic analytics dashboards working
- [ ] Remote config for model version, feature flags and strictness defaults
- [ ] Backups for donated recordings and metadata; the deletion process tested
- [ ] A support email or in-app feedback link

### Release
- Use **staged rollout** (Play Console percentage rollout; App Store phased release),
  starting at ~10%. Watch crashes and false-flag reports for a few days before
  expanding.

## 2. The improvement loop

Run it every **2–3 months** (or once enough new data arrives):

```
consented recordings ─► auto-label ─► filter ─► human spot-check ─► retrain
        ▲                                                            │
        │                                                            ▼
  staged model rollout ◄── release gate (golden set, no regressions) ◄┘
```

1. **Auto-label.** Every donated recording comes with its hadith ID. Run the Phase 3
   pipeline from Stage 4 (alignment) with the *current* model against the vocalised text.
2. **Filter.** Keep clips where every word aligns confidently. Low-confidence spans
   are likely **real user mistakes**:
   - Don't train on them as correct text.
   - Send a sample to human review and label what was actually said. These become
     new mistake test cases, and, once labelled, **training examples of real mistakes**.
3. **Prioritise "this was wrong" reports** (Phase 5). Each one is either a model
   error (a training example) or a user who was actually wrong (a UI/explanation
   problem). A human decides which.
4. **Spot-check** 1–2% of new clips. Keep the label error rate below 2%.
5. **Retrain** from the previous best checkpoint on old + new data, with the same
   speaker-based split rules (new speakers go into train/val/test by speaker).
6. **Release gate.** A new model ships only if, on the **golden set** and the
   **teacher-corrections set**:
   - No metric is worse than the current model by more than a small margin
     (e.g. 1 point), **and**
   - Tashkeel precision stays ≥ 85% at the standard setting, **and**
   - On-device speed and size stay within limits.
7. **Staged rollout** through the remote model switch (10% → 50% → 100%), watching
   false-flag reports per model version. Roll back immediately if they rise.

Keep a `docs/models/CHANGELOG.md`: version, date, data added, scorecard, and rollout notes.

## 3. Growing test sets

The golden set stays **fixed** so results remain comparable over time. Add new,
**separately reported** sets as they appear:
- Real user mistakes labelled from donations
- Teacher side-by-side sessions (Phase 5)
- New collections: record a small golden set whenever you add a collection with
  unfamiliar vocabulary or long isnads

## 4. Product growth (after the loop is running)

- More collections (e.g. full Riyad as-Salihin, Bulugh al-Maram, selections from the
  six books), each with measured vocalisation coverage (Phase 0 script).
- Teacher/class mode: a teacher assigns hadith and sees students' progress.
- Isnad practice tools: narrator name drills, since names are where learners and
  models make the most mistakes.
- More translation languages.

## 5. When to consider a server-side model

Only if **both** hold:
- On-device accuracy has stopped improving across 2+ retraining cycles, **and**
- A much larger model measurably beats it on the golden set.

Even then, keep on-device as the default (offline, private, free to run), and offer
the server model as an optional, clearly disclosed upgrade.

## Ongoing metrics

| Metric | Why it matters |
|---|---|
| False-flag reports per 100 attempts, per model version | The main trust signal |
| Share of users on each strictness level | Too many on lenient means tashkeel precision is too low |
| Attempt completion rate | Low rates suggest tracking is losing people's place |
| Donation opt-in rate and hours per month | Fuel for the improvement loop |
| Distinct donating speakers | Variety matters more than hours |
| Crash-free sessions, battery complaints | On-device model health |
