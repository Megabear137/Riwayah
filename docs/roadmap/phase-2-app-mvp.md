# Phase 2: App MVP

**Weeks 4–10 · Track A · [Back to roadmap](README.md)**

Build a usable memorisation app using the off-the-shelf model chosen in
[Phase 1](phase-1-listener-prototype.md). Feedback is **word-level only**:
tashkeel grading waits for the custom model in Phase 5. Put it in the hands of beta
users and start collecting consented recordings.

## 1. Project setup

- `flutter create app` with iOS and Android targets.
- Suggested packages (check that each is current before adopting):
  - `sherpa_onnx`: on-device speech recognition
  - `record` (or similar): microphone streaming as 16 kHz PCM
  - `drift` or `sqflite`: local database
  - `riverpod` or `bloc`: state management (pick one and stick with it)
  - `dio` / `http`: downloads
- Set up CI (e.g. GitHub Actions) to run `flutter analyze` and `flutter test` on every push.
- Enable RTL layout and load your chosen Arabic font as an app asset.

## 2. Port the listener to Dart

Port `listener/text.py`, `tracker.py` and `grader.py` from Phase 1 into
`app/lib/listener/`.

- **Port the Python unit tests too**, and feed both implementations the same inputs.
  A small script that dumps Python outputs to JSON, checked by a Dart test, catches
  any drift between them.
- Run recognition off the UI thread (a Dart isolate or the plugin's native thread),
  so the UI never stutters while listening.
- Audio loop: microphone → 16 kHz float chunks (100–200 ms) → recogniser → tracker →
  UI state stream.

## 3. Data model

```
Collection(id, name, edition, version, downloaded_at)
Hadith(id, collection_id, number, isnad, matn, translation, grade, reference)
Progress(hadith_id, status[new|learning|memorised], ease, interval_days, due_at)
Attempt(id, hadith_id, started_at, duration_s, words_total, words_correct,
        mistakes_json, mode)
Settings(key, value)
```

Store the hadith text exactly as in the canonical JSON from Phase 0. Build stripped
versions once, at download time, not on every screen render.

## 4. Features

### Library
- A list of available collections, fetched from a small JSON index you host (a CDN or
  object store is fine).
- Download a collection for offline use, with progress, versioning, and deletion.
- Browse and search hadith (search on stripped text, so users don't need to type tashkeel).
- A reading view with full tashkeel, optional translation, and grade/reference.

### Memorise mode (the core feature)
1. The user opens a hadith and taps **Start**. The text is hidden (as blanks or
   faded words).
2. As they recite, the tracker reveals each word as `read`.
3. A word confirmed as skipped or wrong (after N later words, as in Phase 1) turns red,
   and the correct word is shown.
4. **Hint** button: reveals the next word, and records it as a hint in the attempt.
5. **Scope options:** isnad only, matn only, or full hadith. Many learners
   memorise the matn first.
6. **Chunking:** practise one sentence at a time, then join the chunks together.
7. An end-of-attempt summary shows accuracy, mistakes, hints used, and **Retry** /
   **Mark memorised**.

### Review with spaced repetition
- Use **FSRS** (an open-source algorithm with Dart ports) or the simpler SM-2.
- After each attempt, map the result to a rating: no mistakes and no hints = "good",
  a few = "hard", many = "again".
- A "Due today" screen lists hadith whose `due_at` has passed.

### Progress
- Per collection: new / learning / memorised counts.
- Per hadith: attempt history and most-missed words.

### Opt-in recording donation
- A **separate, explicit** consent screen (off by default) explaining what's
  collected, why, how long it's kept, and how to withdraw.
- When opted in: after an attempt, upload the audio + hadith ID + listener output.
  Each recording is **automatically labelled**, because the app knows which hadith
  was being read (Phase 3/6 use this).
- Upload only on Wi-Fi by default. Retry in the background.
- Backend: a signed-URL upload to object storage + a small metadata table.
  Don't build more than that yet.
- Store a random, anonymous reader ID, not personal information. Provide a
  "delete my recordings" button that actually deletes them.
- If minors may use the app, check the requirements in your target stores and
  countries before enabling donation for them.

## 5. Beta

- Distribute through **TestFlight** (iOS) and a **Play Console internal/closed
  testing** track (Android).
- 10–30 testers: include the students from the videos (with permission), plus people
  outside that class.
- Collect feedback in a simple form: what was wrong, which hadith, and a screenshot.
- Add **crash reporting** (e.g. Sentry or Firebase Crashlytics) and minimal
  analytics (attempts per day, completion rate).
- Watch for: tracking losing its place, false mistakes, microphone permission
  problems, and battery drain on long sessions.

## Deliverables checklist

- [ ] Flutter app with Library, Memorise mode, Review, and Progress
- [ ] Dart listener matching the Python reference on shared test cases
- [ ] Offline collections with versioning
- [ ] Consent flow + recording upload backend
- [ ] Beta running on both platforms with 10–30 testers
- [ ] Feedback and crash reporting in place

## Pitfalls

- **Showing tashkeel feedback with a model that can't hear it.** Keep it hidden
  until Phase 5, because early false flags destroy trust.
- **Over-building the backend.** Uploads and a metadata table are enough.
- **Mixing up UI and listener state.** Keep the listener as a pure stream of word
  states, which makes it testable and easy to swap models later.
- **Ignoring older phones.** Test on a 3–4-year-old Android device every week.
