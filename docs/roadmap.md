# Lectern — Android Development and Implementation Roadmap

> **Project:** Lectern  
> **Platform:** Android (Kotlin + Jetpack Compose)  
> **Purpose:** Turn locally imported book chapters into listenable audio and provide a dependable, battery-efficient audiobook player.  
> **Approach:** Learn Android development by completing small, testable milestones.  
> **Document status:** Initial project plan — 17 November 2026

## 1. Project definition

Lectern is a local-first Android application for importing books or chapters, extracting readable text, converting it into spoken audio, caching that audio, and playing it with normal audiobook controls. Playback must work when the screen is locked or the user switches apps. Generated audio should remain usable without an internet connection.

### Core principles

1. **Player first.** Build a reliable screen-off audio player before adding book conversion.
2. **Offline by default.** No account, backend, analytics SDK, or paid API is required for the first release. Installed TTS engines may have their own voice-download or network requirements; explicitly test offline synthesis.
3. **Generate once, play many times.** Do not synthesize speech continuously during playback if a cached audio file is available.
4. **Separate concerns.** Keep UI, book parsing, speech generation, storage, and playback in distinct components.
5. **Ship in vertical slices.** Every milestone ends with a working feature and an objective acceptance test.
6. **Learn deliberately.** Understand each Android API before integrating it; write short experiments and keep implementation notes.
7. **Respect source rights.** Work with books and documents the user is authorized to process. DRM circumvention is outside scope.

### Target use case

1. Open Lectern and import a TXT file or EPUB book.
2. Choose a chapter and a voice.
3. Generate audio and monitor its status.
4. Play or pause the chapter; seek, change speed, and skip backward/forward.
5. Lock the phone; continue playback with notification, headset, and lock-screen controls.
6. Return later and resume near the last saved position.

### In scope for v1.0

- Local library of books and chapters.
- TXT import, followed by EPUB import.
- Text cleaning and deterministic chunking.
- Android `TextToSpeech` synthesis to local audio files.
- Cached, sequential chapter audio playback through Media3.
- Screen-off playback via `MediaSessionService`.
- Lock-screen, notification, headset, and Bluetooth media controls.
- Resume position, playback speed, next/previous chapter, and sleep timer.
- Conversion status, retry/cancel, and cache management.
- Accessible controls and basic automated tests.

### Explicitly out of scope for v1.0

- Cloud synchronization, user accounts, subscriptions, paid TTS APIs.
- Built-in bookstore, web scraping of paywalled content, or DRM removal.
- OCR for scanned PDFs; arbitrary PDF layout reconstruction.
- Voice cloning, multi-character narration, or high-cost neural speech models.
- iOS, web, wearables, or Android Auto-specific features.

### Possible v1.1+ enhancements

PDF text import; bookmarks and notes; EPUB chapter artwork; export of generated audio when supported; alternative on-device TTS engines; improved pronunciation dictionaries; search and text highlighting; smarter chapter segmentation; backup/restore.

## 2. Definition of success

Lectern v1.0 is complete only when all conditions below are met:

- [ ] Import a valid TXT or EPUB book and display its chapter list.
- [ ] Convert a chapter to an ordered, reusable sequence of local audio segments.
- [ ] Play the chapter end-to-end without gaps caused by repeated manual actions.
- [ ] Continue playing when the display is off and the app is in the background.
- [ ] Pause/resume from system media controls and Bluetooth/headset buttons.
- [ ] Restore the book, chapter, and listening position after app/process restart.
- [ ] Play already generated audio with airplane mode enabled.
- [ ] Do not regenerate successful segments when reopening the chapter.
- [ ] Show clear recovery paths for invalid input, unavailable TTS engine, insufficient storage, and failed synthesis.
- [ ] Pass the release test matrix in Section 10 on a physical Android device.

**Quality targets (project goals, not Android guarantees):** cold launch without an unnecessary network request; no audio interruptions caused by navigation or screen lock; no CPU-heavy TTS running during cached playback; no unbounded storage growth; controls usable with TalkBack. Establish battery and memory baselines before optimizing; avoid inventing fixed battery-life claims.

## 3. Prerequisites and development setup

### Knowledge baseline

Java and general programming knowledge are sufficient to begin. Kotlin syntax, Android lifecycles, and Compose are the immediate learning priorities. No backend or machine-learning knowledge is required for the MVP.

### Install

- Current stable **Android Studio** (bundled compatible JDK, Gradle tooling, and Android SDK tools).
- Android SDK platform and Build Tools appropriate to the installed Android Studio version.
- Android emulator image **or**, preferably, a physical Android phone for background playback testing.
- Git and a GitHub repository (optional remote, recommended for version history).

### Create the project

1. Android Studio → **New Project** → **Empty Activity**.
2. Name: `Lectern`; language: **Kotlin**; UI: **Jetpack Compose**.
3. Example application ID: `dev.lectern.app` (choose a unique ID before distributing the app).
4. Suggested `minSdk`: **26 (Android 8.0)**, pending dependency support. This is a project choice, not a requirement to match the phone's Android version.
5. Use a current installed `compileSdk`. Set `targetSdk` according to current Android/Google Play requirements when preparing a release; do not pin outdated SDK numbers in copied examples.
6. Select Gradle Kotlin DSL, keep dependency versions in `gradle/libs.versions.toml` where available.
7. Build, launch on device, rotate the device, inspect Logcat, and commit the clean scaffold.
8. Use the latest **compatible stable** library versions from official release notes; do not paste unverified version numbers from tutorials.

### Dependency introduction order

| When | Dependency / API | Why |
| --- | --- | --- |
| Project creation | Compose Material 3 / Activity Compose | UI and activity integration |
| UI/navigation stage | Lifecycle ViewModel, Lifecycle Compose, Navigation Compose | State and navigation |
| Player stage | `androidx.media3:media3-exoplayer`, `media3-session` | Playback and background media session |
| Storage stage | Room Runtime, Room KTX, Room compiler via KSP | Library and progress metadata |
| Settings stage | Preferences DataStore | Small user preferences |
| Persistent jobs stage | WorkManager | Deferred, interruption-tolerant work only |
| EPUB stage | A maintained EPUB ZIP/XML parser, or focused ZIP/XML processing | Extract structured text safely |

Add libraries at the stage that needs them. For file selection, Android's Storage Access Framework is available without a third-party picker.

### Suggested repo layout

```text
Lectern/
├── app/
│   ├── src/main/
│   │   ├── java/dev/lectern/app/
│   │   │   ├── MainActivity.kt
│   │   │   ├── ui/
│   │   │   │   ├── library/
│   │   │   │   ├── chapters/
│   │   │   │   ├── player/
│   │   │   │   ├── settings/
│   │   │   │   └── navigation/
│   │   │   ├── domain/model/
│   │   │   ├── data/
│   │   │   │   ├── local/
│   │   │   │   └── repository/
│   │   │   ├── importers/
│   │   │   ├── speech/
│   │   │   ├── playback/
│   │   │   └── jobs/
│   │   └── res/
│   ├── src/test/
│   └── src/androidTest/
├── docs/
│   ├── ROADMAP.md
│   ├── ARCHITECTURE.md
│   ├── DECISIONS.md
│   └── TEST_PLAN.md
├── README.md
└── gradle/libs.versions.toml
```

The folder structure is a destination, not a day-one requirement. Create packages as functionality appears. Start with one app module; a multi-module build is not necessary.

## 4. Learning method and milestone rules

Use the same cycle for every milestone:

1. **Read:** Learn the specific API or concept from official documentation.
2. **Spike:** Make a small throwaway example in a scratch screen or branch.
3. **Implement:** Add one functional slice to Lectern.
4. **Test:** Run the acceptance checklist on a real device.
5. **Explain:** Record what the component owns, its lifecycle, and how failures are handled.
6. **Commit:** Keep the project buildable and give the commit a descriptive message.

**Milestone exit rule:** Do not mark a milestone complete while a core acceptance test fails. Record deliberate deferrals under `docs/DECISIONS.md` rather than silently removing requirements.

**Recommended pace:** 5–8 weeks at about 1–2 focused hours per day for someone learning Android as they go; this varies substantially with prior experience. A usable mini-player should arrive well before v1.0.

## 5. Implementation roadmap

### M0 — Environment and project skeleton

**Learn:** Kotlin/Java interoperability, Android Studio, Gradle, modules, `AndroidManifest.xml`, `MainActivity`, `Context`, `minSdk`/`targetSdk`/`compileSdk`, Logcat, physical device debugging.

**Build:** Blank Compose app titled Lectern; Git repository; `README.md`; `docs/ROADMAP.md`.

**Tasks**

- [ ] Install Android Studio and SDK components.
- [ ] Create the Kotlin + Compose project and verify Gradle sync.
- [ ] Enable USB debugging or wireless debugging on the phone.
- [ ] Run the app on the physical Android 16 device.
- [ ] Modify screen text and confirm hot/rebuild workflow.
- [ ] Identify manifest, Gradle files, source folders, and generated resources.
- [ ] Initialize Git; add `.gitignore`; create baseline commit.

**Checkpoint:** A fresh clone builds and launches; Logcat displays an app log statement; no machine-specific secrets or signing keys are committed.

### M1 — Kotlin essentials through domain models

**Learn:** `val`/`var`, nullable types, safe calls, classes, data classes, enums/sealed types, collections, extension functions, lambda expressions, error handling, basic coroutines.

**Build:** Pure Kotlin models for `Book`, `Chapter`, `AudioSegment`, and `ConversionStatus`.

**Tasks**

- [ ] Write sample book and chapter objects.
- [ ] Implement word count and estimated narration duration (labeled as an estimate).
- [ ] Implement a chapter-ordering function.
- [ ] Create explicit status values: `NOT_STARTED`, `QUEUED`, `GENERATING`, `READY`, `FAILED`.
- [ ] Add unit tests for empty text, whitespace, Unicode, and chapter ordering.

**Checkpoint:** Model logic can be tested without launching Android; basic Kotlin null-safety and collection operations are understandable.

### M2 — Compose UI and navigation (mock data)

**Learn:** Composable functions, `Column`, `Row`, `Box`, `LazyColumn`, `Modifier`, Material 3, state hoisting, recomposition, accessibility labels, Navigation Compose.

**Build:** Library screen → book chapters screen → player screen, initially using in-memory sample data.

**Tasks**

- [ ] Implement library with two fake books.
- [ ] Implement book detail / chapter list.
- [ ] Implement player interface (title, play/pause, seekbar, skip, speed).
- [ ] Add empty, loading, and error states.
- [ ] Add navigation with IDs rather than passing full mutable objects.
- [ ] Set content descriptions and sensible touch target sizes.
- [ ] Test dark mode, font scaling, and rotation.

**Checkpoint:** All screens navigate correctly; UI reflects button presses; no audio or database is required yet.

### M3 — ViewModel and UI state

**Learn:** Activity lifecycle, configuration changes, ViewModel lifetime, `StateFlow`, `collectAsStateWithLifecycle`, unidirectional data flow, repository pattern, coroutine scopes.

**Build:** Screen-specific ViewModels that expose immutable UI state and handle user actions.

**Tasks**

- [ ] Move sample data and interaction state out of Composable bodies.
- [ ] Define UI state classes for library, chapters, and player.
- [ ] Observe `StateFlow` in Compose using lifecycle-aware collection.
- [ ] Keep playback ownership out of the ViewModel (eventually the service owns the player).
- [ ] Verify navigation selection survives rotation.

**Checkpoint:** Screen state survives configuration changes appropriately; no business logic depends on a Composable being continuously on screen.

### M4 — Local audio playback (first usable build)

**Learn:** ExoPlayer, `MediaItem`, player states, playlists, audio focus, event listeners, seeking, resources and release, media controls.

**Build:** Play a bundled sample audio file from `res/raw` or an app-owned file.

**Tasks**

- [ ] Add compatible Media3 ExoPlayer/session dependencies.
- [ ] Create a single player owner (avoid one player per Composable).
- [ ] Implement play, pause, seek, ±15 seconds, playback speed.
- [ ] Show duration, position, loading, and failure states.
- [ ] Listen to player events instead of guessing state from button clicks.
- [ ] Release owned player resources at the correct lifecycle boundary.
- [ ] Test with headphones and speaker.

**Checkpoint:** One real audio track plays reliably and controls reflect actual playback state.

**Release marker:** `v0.1` — local audio mini-player.

### M5 — Screen-off playback and Android media controls

**Learn:** Services vs Activities, foreground service types, `MediaSession`, `MediaSessionService`, `MediaController`, notification/lock-screen integration, headset and Bluetooth media buttons, audio focus.

**Build:** Move ExoPlayer ownership into `PlaybackService : MediaSessionService`; UI issues commands using a MediaController.

**Tasks**

- [ ] Register service with `mediaPlayback` foreground service type and the Media3 session intent filter.
- [ ] Declare required foreground-service permissions (including media playback type) and any applicable notification permission behavior.
- [ ] Create the session, return it from `onGetSession()`, release it on destruction.
- [ ] Connect the UI with an asynchronous MediaController lifecycle.
- [ ] Provide meaningful metadata (title/chapter/book and optional cover art).
- [ ] Handle focus loss, noisy audio output, headphones, headset buttons, and Bluetooth controls.
- [ ] Verify pause and stop behavior; avoid restarting playback unexpectedly.

**Checkpoint:** Music continues after locking the phone or navigating away; lock-screen/system/headset controls operate the same session. Force-stopping an app is not a supported continuation scenario.

**Release marker:** `v0.2` — background-capable player.

### M6 — File picker and TXT import

**Learn:** Storage Access Framework, `ACTION_OPEN_DOCUMENT` / `ActivityResultContracts.OpenDocument`, `Uri`, `ContentResolver`, `InputStream`, character encodings, IO dispatching, MIME types, error handling.

**Build:** Choose a TXT file and present its extracted content in the reader/chapters screen.

**Tasks**

- [ ] Open Android's document picker; request only required file types.
- [ ] Read input through `ContentResolver`, not by assuming a filesystem path.
- [ ] Process files off the main thread; set size limits and fail clearly.
- [ ] Normalize line endings and trim irrelevant whitespace without destroying paragraph structure.
- [ ] Copy selected text/book into app-owned storage or persist the selected URI permission if appropriate.
- [ ] Handle revoked permissions, missing files, empty data, and non-UTF-8 inputs gracefully.
- [ ] For v0.3, treat an imported TXT file as one chapter; manual chapter splitting can come later.

**Checkpoint:** A TXT chapter imported from Downloads can be read after closing/reopening the app; an invalid file produces a message rather than a crash.

### M7 — Speech synthesis and chunk pipeline

**Learn:** Android `TextToSpeech`, asynchronous initialization, installed voices/languages, `synthesizeToFile`, `UtteranceProgressListener`, `getMaxSpeechInputLength()`, callbacks, coroutine coordination, cancellation, atomic file publishing.

**Build:** Convert a TXT chapter into an ordered set of local audio segments.

**Tasks**

- [ ] Initialize TTS asynchronously and expose ready/unavailable status.
- [ ] List and preview supported voices; indicate unavailable voice data.
- [ ] Split text on sensible sentence/paragraph boundaries with a strict character cap below the engine limit.
- [ ] Assign deterministic IDs and ordering to segments.
- [ ] Submit one segment at a time initially; wait for completion callbacks before advancing.
- [ ] Write output to a temporary path; validate before marking complete and renaming/publishing it.
- [ ] Record failures and retry only the failed/incomplete segments.
- [ ] Keep `TextToSpeech` cleanup and callback thread safety explicit.
- [ ] Check whether chosen voices synthesize with airplane mode enabled; do not assume all engines are offline.
- [ ] Store the engine/voice/settings identity with conversion metadata so cached audio remains reproducible.

**Checkpoint:** A short 3–5-page chapter converts to files that play in the correct order; cancelled/failed conversions leave no segment falsely marked ready.

**Release marker:** `v0.3` — TXT-to-audio prototype.

### M8 — Join the converter to the player

**Learn:** Media3 playlists, segment boundaries, duration mapping, per-segment playback position, asynchronous playlist refresh, cache validity.

**Build:** Generated audio appears as one logical chapter in the player.

**Tasks**

- [ ] Build MediaItems from ready audio segments in stored order.
- [ ] Move automatically between segments and show chapter-level progress.
- [ ] Implement skip ±15 seconds across segment boundaries.
- [ ] Implement resume using segment ID/index plus local offset; calculate chapter-level time when segment durations are available.
- [ ] Prevent incomplete segments from being treated as complete audio.
- [ ] Allow playback while remaining segments are generated, only if the queue is safe and generation state is visible; otherwise require full-chapter completion for v0.4.
- [ ] Ensure replay uses cached files without invoking TTS.

**Checkpoint:** The reader can import a TXT file, generate it once, and listen to it with the screen off.

**Release marker:** `v0.4` — first end-to-end Lectern flow.

### M9 — Room persistence and DataStore settings

**Learn:** relational tables, Room entities/DAOs, primary/foreign keys, transactions, migrations, Flow-backed queries; DataStore for preferences.

**Build:** Durable library, conversion metadata, resume position, and preferences.

**Tasks**

- [ ] Persist books, chapters, segments, and conversion runs in Room.
- [ ] Store file *references* in the database; keep audio bytes in app-owned files.
- [ ] Persist progress periodically and on pause/stop/chapter change, not on every player tick.
- [ ] Add resume behavior on app relaunch.
- [ ] Store voice, playback speed, and skip interval in DataStore.
- [ ] Handle database migrations; test upgrade from an older local database.
- [ ] Clean up orphaned audio and database entries when removing a book.

**Checkpoint:** Force-close/relaunch after normal use preserves books, voice preferences, and an approximately correct playback position; successful cached audio still plays offline.

### M10 — EPUB support and improved import

**Learn:** ZIP containers, XML/XHTML parsing, document spine/order, metadata, HTML text extraction, resource limits, secure parsing.

**Build:** Import non-DRM EPUB books and extract chapters correctly.

**Tasks**

- [ ] Validate EPUB ZIP contents; enforce archive entry count, total inflated size, and path safety limits.
- [ ] Read package metadata, spine order, and document resources.
- [ ] Extract reading-order text while preserving headings and paragraph boundaries.
- [ ] Remove navigation and boilerplate where reliably identifiable.
- [ ] Preserve chapter titles; allow manual chapter renaming or splitting when the source is imperfect.
- [ ] Test on at least three legally usable EPUB files with different formatting.
- [ ] State that DRM-protected sources are unsupported; do not bypass DRM.

**Checkpoint:** Multiple EPUBs import with correct reading order and understandable spoken text; invalid/hostile archives are rejected without excessive memory use.

### M11 — Robust conversion jobs and storage management

**Learn:** structured concurrency, cancellation, idempotence, retry/backoff, WorkManager constraints/quotas, foreground execution restrictions, job progress, storage lifecycle.

**Build:** A conversion queue with statuses, cancellation, retries, and safe restart behavior.

**Tasks**

- [ ] Model conversion as a persistent sequence of independently verifiable segments.
- [ ] Store queue state so interrupted work can be resumed or retried.
- [ ] Choose execution based on the task: coroutine while UI is active; WorkManager for appropriate deferrable persistent work; evaluate Android foreground-service restrictions before promising long-running off-screen generation.
- [ ] Do not attach TTS generation to `mediaPlayback` simply to evade background execution rules.
- [ ] Add clear progress, cancel, and retry controls.
- [ ] Limit conversion concurrency to avoid device overheating and TTS engine contention.
- [ ] Add cache usage screen and safe delete/regenerate actions.
- [ ] Implement disk-space checks and clean failure if storage runs out.

**Checkpoint:** Killing the app during conversion cannot corrupt previously completed audio; resuming/retrying does not duplicate completed work.

**Android 16 note:** Long-running WorkManager workers can consume JobScheduler job quota even when foreground execution is used. Do not assume an unlimited background conversion job.

### M12 — Reader experience and polish

**Learn:** accessibility semantics, TalkBack, responsive layouts, preference state, animations (minimal), state restoration.

**Build:** Daily-use audiobook controls.

**Tasks**

- [ ] Add sleep timer with explicit end behavior.
- [ ] Add next/previous chapter; optionally auto-play next chapter.
- [ ] Offer speeds such as 0.75×, 1.0×, 1.25×, 1.5× and 2.0×.
- [ ] Add listening position and conversion progress indicators.
- [ ] Implement clear empty states, error recovery, and import status.
- [ ] Test large text and display scaling, dark/light modes, and TalkBack labels.
- [ ] Keep screen awake only when genuinely necessary (normally never during audio playback).

**Checkpoint:** A new user can import, convert, listen, pause, and resume without developer assistance.

### M13 — Automated tests, performance, and release

**Learn:** JVM unit tests, instrumented tests, Compose UI tests, fakes, Logcat, profiler, memory leaks, Baseline Profiles (optional), APK signing.

**Build:** Stable v1.0 candidate and distributable APK.

**Tasks**

- [ ] Add tests for chunker, ordering, resume mapping, statuses, import validation, and repository logic.
- [ ] Add UI tests for library → chapter → player navigation and empty/error states.
- [ ] Test background playback and real TTS on a physical device (cannot be proved by pure unit tests).
- [ ] Compare CPU/battery behavior of cached playback vs live synthesis.
- [ ] Investigate audio interruptions, stuck services, leaked TTS instances, and incomplete cache cleanup.
- [ ] Check notification, foreground service and SDK policy requirements.
- [ ] Generate a signed release APK; verify install/update path and release build behavior.
- [ ] Update `README.md`, screenshots, limitations, test results, and release notes.

**Checkpoint:** All Section 2 success conditions and Section 10 release checks pass; tag `v1.0.0`.

## 6. Architecture and responsibilities

```text
Android document picker
        |
        v
Importer (TXT / EPUB) --> normalized chapters --> Room metadata
        |                                         |
        v                                         v
Text chunker --> conversion queue --> TTS engine --> app-owned audio cache
                                                   |
                                                   v
Compose UI <--> ViewModels <--> repositories / playback controller
                                                   |
                                                   v
                                     MediaSessionService + ExoPlayer
                                                   |
                         notification / lock screen / headset / Bluetooth
```

### Component contracts

| Component | Owns | Must not own |
| --- | --- | --- |
| Compose screens | Presentation and click handlers | TTS lifecycle, ExoPlayer instance, database connection |
| ViewModels | UI state and orchestration | Long-lived playback engine |
| Importer | Parsing and normalized text | Navigation or audio playback |
| Chunker | Stable chunk boundaries and IDs | Android UI APIs |
| SpeechGenerator | TTS initialization, synthesis, and callback results | Book library UI |
| Conversion coordinator | Queue, cancellation, retries, cache state | Media session |
| PlaybackService | ExoPlayer, MediaSession, player state | File parsing and book import |
| Repositories | Data access / persistence abstractions | Direct Compose rendering |

### Proposed metadata (conceptual schema)

- **Book:** `id`, `title`, `author?`, `sourceType`, `importedAt`, `coverPath?`.
- **Chapter:** `id`, `bookId`, `index`, `title`, `textPath`, `status`, `textHash`.
- **AudioSegment:** `id`, `chapterId`, `index`, `textStart`, `textEnd`, `audioPath`, `durationMs?`, `status`, `audioHash?`.
- **ConversionRun:** `id`, `chapterId`, `engineId`, `voiceId`, `speechRate`, `createdAt`, `status`, `error?`.
- **ListeningProgress:** `bookId`, `chapterId`, `segmentIndex`, `segmentPositionMs`, `updatedAt`.
- **Preferences (DataStore):** default speed, voice, skip interval, auto-next, sleep timer defaults.

Do not store entire audio blobs in Room. Keep book text and audio in app-owned storage, with metadata paths/URIs in Room. Use generated IDs and avoid treating untrusted book titles as filenames.

### Audio-cache rules

1. A segment is `READY` only after successful synthesis, file validation, and an atomic publication step.
2. Conversion identity includes the input text hash, engine/voice/settings, and chunker version.
3. Reuse existing valid segments for identical conversion identity.
4. Never overwrite audio the player is currently reading; use a replacement/commit approach.
5. Delete audio only when the associated book/conversion is no longer referenced or the user explicitly requests cleanup.
6. Reconcile missing/orphaned files on startup or via deferred maintenance.

## 7. Important Android implementation notes

### Playback manifest (illustrative, validate against current Media3 docs)

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<application ...>
    <service
        android:name=".playback.PlaybackService"
        android:exported="true"
        android:foregroundServiceType="mediaPlayback">
        <intent-filter>
            <action android:name="androidx.media3.session.MediaSessionService" />
        </intent-filter>
    </service>
</application>
```

On supported Android releases, handle runtime notification permission as applicable. MediaSession media notifications have version-dependent behavior; test actual devices. The manifest snippet is not a complete manifest and must be adapted to the project's package and installed Media3 release.

### Separate playback from conversion

A media foreground service supports user-noticeable *playback*. It should not be used as a general-purpose unlimited background-computation service. Voice engines may be slow, may require voice data downloads, and may restrict offline or file synthesis. Design for resumption, retries, and appropriate execution windows rather than guaranteeing continuous conversion with the phone locked.

### EPUB safety

Treat imported books as untrusted. Defend against compressed ZIP bombs, `../` extraction paths, enormous files, malformed XML, and unsupported embedded resources. Avoid extracting arbitrary entries into writable paths derived from the archive.

### Battery strategy

- Prefer cached audio playback with ExoPlayer instead of live TTS.
- Do not poll playback position dozens of times per second; update UI at a reasonable interval only while visible.
- Persist resume state periodically and on important transitions rather than on every frame.
- Avoid wakelocks unless platform guidance specifically requires them; rely on Media3's service and media playback patterns.
- Cancel unnecessary conversion when the user requests it; limit concurrency.
- Avoid scanning the entire library at every app launch.
- Verify improvements using Android Studio profilers and on-device battery diagnostics; measure rather than assume.

## 8. Learning reference map

| Topic | Read / practice |
| --- | --- |
| Kotlin | https://kotlinlang.org/docs/getting-started.html |
| Compose course | https://developer.android.com/courses/android-basics-compose/course |
| App architecture | https://developer.android.com/topic/architecture |
| Lifecycle and ViewModel | https://developer.android.com/topic/libraries/architecture/viewmodel |
| Navigation Compose | https://developer.android.com/develop/ui/compose/navigation |
| File selection / SAF | https://developer.android.com/training/data-storage/shared/documents-files |
| Android TTS | https://developer.android.com/reference/android/speech/tts/TextToSpeech |
| TTS callbacks | https://developer.android.com/reference/android/speech/tts/UtteranceProgressListener |
| Media3 player | https://developer.android.com/media/implement/playback-app |
| MediaSessionService | https://developer.android.com/media/media3/session/background-playback |
| Room | https://developer.android.com/training/data-storage/room |
| DataStore | https://developer.android.com/topic/libraries/architecture/datastore |
| WorkManager | https://developer.android.com/develop/background-work/background-tasks/persistent |
| Long-running workers | https://developer.android.com/develop/background-work/background-tasks/persistent/how-to/long-running |
| Testing | https://developer.android.com/training/testing |
| Android 16 SDK | https://developer.android.com/about/versions/16/setup-sdk |
| Play target API requirements | https://support.google.com/googleplay/android-developer/answer/11926878 |

**Study technique:** For each milestone, write a 5–10-line note answering: What problem does the Android component solve? Who creates it? Who destroys it? How does the UI communicate with it? What happens on failure or process death?

## 9. Planned releases

| Release | Scope | Required demonstration |
| --- | --- | --- |
| `v0.0` | Project scaffold | Installs and starts on a real phone |
| `v0.1` | Mock library and real local audio player | Bundled audio with play/pause/seek |
| `v0.2` | Background media session | Screen-off, notification, Bluetooth controls |
| `v0.3` | TXT importer + TTS proof of concept | Convert a short TXT chapter to files |
| `v0.4` | Integrated audiobook pipeline | Import → convert → cached background playback |
| `v0.5` | Room + settings | Library, resume position, preferences survive restart |
| `v0.6` | EPUB import | Reads chapters in the correct order |
| `v0.7` | Conversion queue and cache management | Retry, cancellation, interruption recovery |
| `v0.9` | UX and accessibility polish | Sleep timer, errors, TalkBack testing |
| `v1.0.0` | Stable local-first release | Release matrix and documentation complete |

## 10. Testing and acceptance matrix

| ID | Test | Expected outcome | Milestone |
| --- | --- | --- | --- |
| T01 | Launch on physical Android 16 phone | Clean launch; no crash | M0 |
| T02 | Rotate during navigation | Selection preserved appropriately | M3 |
| T03 | Play/pause/seek local sample audio | Correct audible and visible state | M4 |
| T04 | Lock phone while playing | Playback continues | M5 |
| T05 | Use notification/lock-screen controls | Same session responds | M5 |
| T06 | Use wired/Bluetooth headset controls | Play/pause routes correctly | M5 |
| T07 | Unplug headphones / audio focus interrupted | Audio behaves safely and predictably | M5 |
| T08 | Import valid UTF-8 TXT | Content displayed correctly | M6 |
| T09 | Import malformed/empty file | Clear recoverable error | M6 |
| T10 | Convert punctuation-heavy, Unicode text | Ordered files without truncation | M7 |
| T11 | Disable network and replay cached chapter | Works without connection | M8 |
| T12 | Skip backward across segment boundary | Correct chapter-level position | M8 |
| T13 | Restart app after listening | Resume close to saved position | M9 |
| T14 | Import multiple EPUB formats | Chapter order and text sensible | M10 |
| T15 | Feed malformed/oversized EPUB | Safe rejection | M10 |
| T16 | Cancel conversion partway | No incomplete segment marked ready | M11 |
| T17 | Interrupt conversion and retry | No duplicate successful segments | M11 |
| T18 | Run out of disk space | Useful error; existing audio intact | M11 |
| T19 | TalkBack + large font | Core controls accessible | M12 |
| T20 | Cached audio for an extended listening session | No unnecessary synthesis or wakeful busy loop | M13 |
| T21 | Signed release APK installed as upgrade | Data preserved and app works | M13 |

**Testing environments:** Android 16 physical device is mandatory for the project's personal use case. If possible, include at least one lower supported API emulator/device and one different OEM phone. Emulators alone are insufficient for Bluetooth, audio focus, battery, and real TTS behavior.

## 11. Suggested work sessions

For 60–120-minute sessions:

1. **15–25 min:** Read one narrow concept and capture notes.
2. **15–25 min:** Build a minimal experiment.
3. **30–60 min:** Integrate into Lectern.
4. **10 min:** Run a checkpoint, log a bug, and commit.

At the end of each week, record what works, what is broken, what was learned, and the next smallest task. Avoid spending entire sessions reorganizing packages or redesigning architecture before a working feature exists.

### First seven work sessions

1. Install tools, create Lectern, run on physical phone, commit.
2. Refresh Kotlin types, data classes, null safety; write book/chapter models and tests.
3. Build library list with static book data.
4. Build chapter list and navigation.
5. Build player UI with fake state, then wire ViewModel state.
6. Add Media3 and play bundled audio.
7. Implement play/pause/seek correctly; begin the background-service milestone.

**First immediate deliverable:** A real MP3/M4A file plays inside Lectern. Do not begin complex TTS integration before this is working.

## 12. Decision log (fill during development)

Use `docs/DECISIONS.md` to record choices, the reason for each, alternatives considered, and revisit triggers.

Decisions to make:

- [ ] Final package/application ID.
- [ ] Minimum supported SDK after checking chosen dependencies.
- [ ] Which installed TTS engines/voices are supported initially.
- [ ] Whether each chapter must finish conversion before playback begins.
- [ ] Where source text and generated audio are stored and how deletion works.
- [ ] Whether imported source files are copied or referenced by persisted URI.
- [ ] Which EPUB parser/library is maintained and safe for local input.
- [ ] When and how conversion may continue outside the foreground UI.
- [ ] Which formats and export behaviors are available without violating source rights.

## 13. Final release checklist

- [ ] Release build produces a signed, installable APK.
- [ ] Application name, icon, version, and package ID are correct.
- [ ] Runtime permissions and foreground service declarations are minimal and accurate.
- [ ] Local-only behavior is documented; no unexpected network calls.
- [ ] Privacy statement explains local book/audio storage and any installed TTS engine dependencies.
- [ ] Storage usage and deletion options are visible.
- [ ] Corrupted EPUB/TXT and inaccessible files do not crash the app.
- [ ] All playback controls work with the screen locked.
- [ ] Process restart, app update, low-storage, and offline tests pass.
- [ ] TalkBack and large-font basics pass.
- [ ] `README.md` has installation, usage, known limitations, screenshots, and build instructions.
- [ ] `docs/TEST_PLAN.md` records device versions and actual results.
- [ ] No debug logs containing book text, tokens, or sensitive file paths are included in release builds.
- [ ] Git tag `v1.0.0` and release notes created.

## 14. Known risks and how to reduce them

| Risk | Mitigation |
| --- | --- |
| System TTS sounds robotic | Keep the speech generator behind a replaceable interface; improve engine later |
| TTS cannot export from a selected voice | Detect unsupported synthesis and show an actionable message |
| Different TTS engines behave inconsistently | Test actual engine availability, callbacks, output formats, and offline behavior |
| Long conversions are interrupted | Store per-segment progress; resume idempotently; respect Android background limits |
| EPUB chapter boundaries are inconsistent | Use reading-order spine and allow manual correction later |
| Generated audio consumes too much storage | Show usage; delete/regenerate safely; optional storage policy |
| Playback stops on app dismissal | Keep player in MediaSessionService and test service lifecycle |
| User rotates / Android reclaims process | ViewModel for UI lifecycle and Room/DataStore for durable state |
| Audio chunks sound discontinuous | Split on paragraph/sentence boundaries; test pauses, cross-boundary seeking, and voice consistency |
| Scope expands indefinitely | Keep v1.0 requirements fixed; move extras to v1.1+ |

## 15. External references and validation notes

Official Android documentation should be checked again when implementing APIs, because API behavior and Google Play target requirements change:

- **MediaSessionService background playback:** https://developer.android.com/media/media3/session/background-playback
- **TextToSpeech API (limits and file synthesis):** https://developer.android.com/reference/android/speech/tts/TextToSpeech
- **Document selection / persistable URI permissions:** https://developer.android.com/training/data-storage/shared/documents-files
- **Long-running WorkManager workers and Android 16 quotas:** https://developer.android.com/develop/background-work/background-tasks/persistent/how-to/long-running
- **Google Play target API policy:** https://support.google.com/googleplay/android-developer/answer/11926878

---

**Development rule:** A small, working, tested feature is more valuable than a large unfinished subsystem. Build the player first, add import and conversion second, then improve resilience and narration quality.
