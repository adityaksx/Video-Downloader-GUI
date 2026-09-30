# Video Downloader GUI — Master Implementation Plan

## Mission

Transform the current `Video-Downloader-GUI` into a reliable, modern, cross-platform desktop media downloader built around **yt-dlp + FFmpeg**.

Target capabilities:

- URL-first media discovery
- inspect metadata, formats, audio and subtitles before download
- single video and playlist workflows
- selective playlist downloads
- reusable download profiles
- persistent task queue
- pause/resume/retry/cancel
- multi-audio and subtitles
- metadata and thumbnail handling
- proxy/Tor routing
- playlist manager
- history
- system diagnostics
- preferences
- modern responsive UI
- robust error handling
- maintainable architecture

Primary rule: **extend and refactor the current codebase instead of blindly replacing working parts.**

---

# 1. Current Repository Baseline

The current repository already contains:

```text
main.py
app_controller.py
gui.py
models.py
config.py
services/
  audio_downloader.py
  combiner_service.py
  info_service.py
  playlist_service.py
  proxy_service.py
  queue_service.py
  setup_service.py
  subtitle_downloader.py
  thumbnail_metadata.py
  video_downloader.py
utils/
  file_utils.py
  log_utils.py
  thread_utils.py
  validators.py
  yt_dlp_builder.py
```

Existing functionality includes:

- yt-dlp invocation
- media info extraction
- format listing
- playlist discovery
- concurrent queue workers
- proxy and Tor settings
- metadata/thumbnail/subtitle flags
- multiple formats
- quality caps
- playlist item selection
- speed limiting
- settings/history persistence
- Tkinter/ttk UI

## Existing weaknesses

1. `AppController` has too many responsibilities.
2. Queue identity is URL-based instead of task-ID based.
3. Queue persistence/recovery is weak.
4. Pause/resume/cancel semantics are incomplete.
5. Progress parsing is fragile.
6. Media discovery data is not fully normalized.
7. Playlist rows are loosely structured dictionaries.
8. Settings lack strong schema/versioning.
9. Diagnostics are limited to dependency checks.
10. GUI is monolithic.
11. Discovery, planning, execution, persistence and presentation are tightly coupled.

Do not fix these by merely adding more conditionals to the existing controller.

---

# 2. Product Architecture

Refactor toward clear layers:

```text
Application
├── domain/
│   ├── models.py
│   ├── enums.py
│   ├── events.py
│   └── errors.py
│
├── application/
│   ├── download_manager.py
│   ├── discovery_manager.py
│   ├── playlist_manager.py
│   ├── profile_manager.py
│   ├── history_manager.py
│   └── diagnostics_manager.py
│
├── services/
│   ├── ytdlp/
│   │   ├── engine.py
│   │   ├── command_builder.py
│   │   ├── extractor_service.py
│   │   └── progress_parser.py
│   ├── ffmpeg/
│   │   ├── engine.py
│   │   └── capabilities.py
│   ├── network/
│   │   ├── proxy_service.py
│   │   ├── tor_service.py
│   │   └── connectivity_service.py
│   ├── persistence/
│   │   ├── database.py
│   │   ├── queue_store.py
│   │   ├── settings_store.py
│   │   ├── history_store.py
│   │   └── playlist_store.py
│   └── system/
│       ├── diagnostics.py
│       ├── filesystem.py
│       └── process.py
│
├── ui/
│   ├── main_window.py
│   ├── components/
│   ├── pages/
│   │   ├── home_page.py
│   │   ├── queue_page.py
│   │   ├── playlists_page.py
│   │   ├── history_page.py
│   │   ├── settings_page.py
│   │   └── diagnostics_page.py
│   ├── dialogs/
│   └── theme/
│
└── tests/
```

Exact folders may differ, but responsibilities must remain separated.

---

# 3. Core Domain Models

Replace the current lightweight task structures with explicit models.

## DownloadTask

Fields:

- UUID `id`
- URL
- canonical URL
- source/site
- source type
- title
- playlist ID
- playlist index
- created/updated timestamps
- status
- priority
- profile ID
- output directory
- filename template
- format selector
- target container
- maximum height
- video preferences
- audio selection
- subtitle selection
- metadata policy
- thumbnail policy
- proxy profile ID
- cookies/auth profile reference for future use
- retry count
- max retries
- error code/message
- output files
- total bytes
- downloaded bytes
- speed
- ETA
- elapsed time
- progress
- stage

## MediaInfo

Normalize:

- title
- description
- uploader/channel
- uploader URL
- webpage URL
- canonical URL
- thumbnail
- duration
- upload date
- categories
- tags
- view count
- like count
- age restriction flag
- live status
- playlist metadata
- chapters
- available formats
- audio tracks
- subtitles
- automatic captions
- extractor/site

Never assume optional metadata exists.

## StreamInfo

Fields:

- format ID
- extension
- protocol
- container
- video codec
- audio codec
- width
- height
- FPS
- bitrate
- filesize
- approximate filesize
- HDR flag
- language
- quality label
- video-only/audio-only/mixed
- note
- selected

## AudioTrackInfo

Fields:

- format ID
- language
- language label
- codec
- bitrate
- sample rate
- channels
- original/dub label when available
- default flag
- selected flag

## SubtitleInfo

Fields:

- language
- language label
- manual/automatic
- formats
- URL/reference if available
- selected flag

---

# 4. Media Discovery

The Fetch operation must be a first-class workflow.

Pipeline:

```
URL
 ↓
Validate
 ↓
Detect source/extractor
 ↓
Extract JSON metadata
 ↓
Normalize metadata
 ↓
Build stream matrix
 ↓
Build audio-track list
 ↓
Build subtitle list
 ↓
Build playlist information
 ↓
Present download plan
```

Fetch must **not download the media**.

Show:

- title
- description
- thumbnail
- uploader
- duration
- upload date
- views/likes where available
- source website
- available resolutions
- video codecs
- audio codecs
- audio languages/tracks
- subtitles
- automatic captions
- chapters
- file size when known
- live/private/login-required indication when reported

Add Refresh Info.

Cache fetched metadata for a short period so changing UI options does not repeatedly hit the source.

---

# 5. Format Selection

Create a dedicated Download Profile and custom format builder.

## Presets

- Best Quality
- Best MP4
- Best MKV
- 2160p
- 1440p
- 1080p
- 720p
- 480p
- 360p
- Video Only
- Audio MP3
- Audio M4A
- Audio FLAC
- Audio Opus
- Custom

## Custom builder

Allow:

- exact format ID
- best video + best audio
- maximum resolution
- exact resolution
- codec preference
- container preference
- FPS preference
- HDR preference
- filesize-aware fallback
- MP4 compatibility preference
- hardware-friendly codec preference

Advanced yt-dlp format expressions belong under Advanced Options.

Never expose a selector that is known to be invalid for the currently discovered media.

---

# 6. Video Download

Support:

- title preservation
- description preservation
- requested format
- requested resolution
- metadata
- chapters
- thumbnail
- subtitles
- multiple audio tracks

Default behavior should favor the highest requested quality that is actually available, with safe fallback.

Do not promise exact resolution when the source does not provide it.

---

# 7. Metadata and Output

Support:

- embed metadata
- embed thumbnail
- save thumbnail separately
- embed chapters
- save metadata JSON
- custom filename template
- custom output directory
- uploader subfolders
- playlist subfolders
- overwrite/skip policy
- resume partial
- no-clobber behavior
- safe filename sanitization
- temporary files outside final output where practical

Filename template examples:

```
%(title)s.%(ext)s
%(uploader)s/%(title)s [%(id)s].%(ext)s
%(playlist_index)03d - %(title)s.%(ext)s
```

Provide a template help dialog with supported variables.

---

# 8. Subtitles

Support:

- manual subtitles
- automatic subtitles
- selected languages
- all available languages
- subtitle-only downloads
- embedded subtitles
- sidecar files
- SRT
- VTT
- ASS
- language-aware names

UI:

```
[x] Download subtitles
[x] Include automatic captions
Language list:
[x] en
[x] hi
[x] ...
[Select All] [Clear]

Format: SRT
[x] Embed in media
[x] Keep sidecar file
```

Rules:

- never invent a language that was not discovered
- keep manual and automatic captions distinct
- when "all" is selected, include all actually available tracks
- clearly report unsupported embedding formats/containers

---

# 9. Multi-Audio

Support:

- default/best audio
- selected audio language
- multiple selected tracks
- all available tracks where technically supported
- separate audio files as fallback
- multi-track muxing when supported

UI must display discovered tracks rather than just a language dropdown.

Example:

```
Audio
○ Best / Default
○ English — Original
○ Hindi — Dub
○ Spanish — Dub

[x] Include multiple tracks
```

Never silently claim that all audio tracks were embedded when container/codec limitations prevented it.

---

# 10. Playlist Workflow

## Fetch playlist

Display:

- playlist title
- uploader
- item count
- item thumbnail
- title
- duration
- index
- availability
- selected state

## Selection controls

- select all
- deselect all
- invert
- select range
- search
- filter
- first N
- last N
- skip unavailable

## Download model

A playlist selection should expand into independent DownloadTask objects:

```
Playlist
 ├── task-001
 ├── task-002
 ├── task-003
 └── ...
```

All tasks receive the same **profile/options snapshot**.

Each task independently supports:

- progress
- pause/resume
- retry
- failure state
- cancellation
- output file
- history

Do not use one opaque playlist task where individual task control is practical.

## Large playlists

For 1,000+ items:

- use flat playlist extraction first
- lazy-load expensive metadata
- fetch full info only when needed
- virtualize/paginate UI rows
- do not block the UI

---

# 11. Queue / Task Manager

Make this the application core.

## Task states

```
Pending
Preparing
FetchingInfo
Downloading
PostProcessing
Completed
Failed
Paused
Cancelled
Skipped
Retrying
Interrupted
```

Implement an explicit state machine.

Reject illegal transitions.

## Controls

Per task:

- Start
- Pause
- Resume
- Retry
- Cancel
- Remove
- Open File
- Open Folder
- Copy URL
- Edit
- Duplicate
- Move Up
- Move Down
- Change Priority

Global:

- Start All
- Pause All
- Resume All
- Cancel All
- Retry Failed
- Clear Completed
- Clear Failed

## Concurrency

Settings:

- max simultaneous downloads
- max post-processing jobs
- global speed limit
- per-task priority

Default concurrency can remain 2.

Do not assume more workers always means better throughput.

## Queue persistence

Persist meaningful state transitions.

On startup:

- restore pending tasks
- restore queued playlist tasks
- identify interrupted downloads
- convert stale active tasks to `Interrupted`
- offer Resume / Retry / Remove

---

# 12. Pause / Resume / Cancel

Implement actual process-level control.

Cancel must:

1. stop yt-dlp
2. stop FFmpeg if active
3. preserve useful partial state where safe
4. update task state
5. release worker/process resources

Pause is inherently engine/site dependent. Treat it as best-effort and never fake it by only hiding the task.

Resume should use yt-dlp's resume behavior where supported.

---

# 13. Retry Engine

Automatically retry transient failures:

- connection reset
- temporary HTTP errors
- timeouts
- transient extractor/network failures

Do not endlessly retry:

- invalid URL
- permanently unavailable content
- authentication failures
- unsupported extractors
- unavailable format
- permission failures
- disk full

Configurable:

- maximum retries
- retry delay
- exponential backoff
- retry categories

Example:

```
retry 1 → 2s
retry 2 → 5s
retry 3 → 10s
retry 4 → 30s
```

---

# 14. Website Support

Use the installed **yt-dlp extractor ecosystem** as the primary compatibility layer.

Common target sites include:

- YouTube
- Instagram
- Facebook
- X
- Reddit
- TikTok
- Vimeo
- Twitch
- Dailymotion
- SoundCloud
- other yt-dlp-supported extractors

The application must **not** claim it works with literally every website.

Correct product language:

> Supports websites handled by the installed yt-dlp extractor set.

This also means DRM-protected or extractor-unsupported sites may fail.

For adult websites, generic extractor support can be exposed when provided by yt-dlp. The application should keep the workflow technically neutral and include a legal-use notice.

---

# 15. Proxy + Tor

Create a dedicated network abstraction.

## Proxy profiles

Support:

- HTTP
- HTTPS
- SOCKS5

Fields:

- profile name
- proxy URL
- enabled
- optional username/password
- connection status

Do not write credentials to logs.

## Tor

Modes:

- Off
- Local Tor
- Custom SOCKS endpoint

Default local endpoint:

`socks5://127.0.0.1:9050`

But do not assume Tor is installed or running.

Diagnostics must test connectivity.

## Proxy test

Show:

- reachable/unreachable
- latency
- external IP if applicable
- Tor detected/not detected

---

# 16. Authentication / Cookies Architecture

Prepare for:

- browser cookie import
- cookie file
- login-required detection
- per-site auth profiles

Phase 1:

- model/settings support
- safe error reporting

Phase 2:

- browser cookie import
- protected local storage where practical

Never log cookies, tokens or passwords.

---

# 17. Playlist Manager

Dedicated saved-playlist library.

Store:

- playlist URL
- display name
- description
- last fetched time
- item count
- thumbnail
- tags
- default profile

Actions:

- refresh
- inspect
- download selected
- download new items
- rename
- delete
- search

Automatic recurring playlist refresh should be implemented only after the core queue is reliable.

---

# 18. History

Columns:

- title
- source
- website
- status
- date
- resolution
- format
- output path

Actions:

- open file
- open folder
- copy URL
- retry
- redownload
- delete record
- clear history
- search/filter

Deleting a history record must **not** delete the physical media unless the user explicitly chooses it.

---

# 19. Preferences

Organize settings.

## General

- download directory
- theme
- startup behavior
- confirmation prompts

## Downloads

- default profile
- filename template
- overwrite policy
- retries
- concurrency
- speed limit
- resume behavior
- post-processing

## Video

- preferred container
- preferred codec
- maximum resolution
- FPS
- HDR

## Audio

- format
- quality
- language
- multi-audio

## Subtitles

- enabled
- automatic captions
- languages
- format
- embed/sidecar

## Network

- proxy profile
- Tor
- timeout
- retries
- IPv4/IPv6 preference

## Engine

- yt-dlp executable
- FFmpeg executable
- automatic engine update
- player clients
- extractor arguments
- advanced options

## Privacy

- history retention
- log retention
- telemetry OFF by default
- clear temporary files
- clear app data

Use schema-versioned settings migrations.

---

# 20. System Diagnostics

Create a real diagnostics page.

Check:

### App

- version
- Python version
- OS
- architecture
- UI backend

### yt-dlp

- installed
- version
- executable path
- version update status

### FFmpeg

- installed
- version
- ffmpeg path
- ffprobe path

### Storage

- output directory exists
- writable
- free space
- temp directory writable

### Network

- internet
- DNS
- HTTPS
- proxy
- Tor

### Runtime

- queue size
- active workers
- active processes
- memory usage if practical

Use status:

- Ready
- Degraded
- Unavailable

Buttons:

- Run Full Diagnostic
- Copy Diagnostic Report

Do not leak secrets or unnecessary sensitive paths.

---

# 21. Modern UI

Move away from a monolithic single-window form.

Recommended structure:

```
┌──────────────────────────────────────────────────────────┐
│ Video Downloader                              ● Ready    │
├──────────────┬───────────────────────────────────────────┤
│ Home         │                                           │
│ Queue        │                  Content                  │
│ Playlists    │                                           │
│ History      │                                           │
│ Settings     │                                           │
│ Diagnostics  │                                           │
└──────────────┴───────────────────────────────────────────┘
```

## Home

Primary actions:

```
Paste URL
[ Fetch Information ]

[thumbnail]  Title
             Channel
             Duration
             Source
             Description

Formats / Resolution
[ Best MP4 ▼ ] [1080p ▼]

Audio
Subtitles
Metadata

[ Download Now ] [ Add to Queue ]
```

## Queue

Table:

- thumbnail
- title
- status
- progress
- speed
- ETA
- size
- profile
- actions

Rows can expand for detailed engine logs.

## Playlists

Selectable table/grid with bulk operations.

## History

Searchable table.

## Settings

Category navigation + searchable settings.

## Diagnostics

Capability dashboard.

---

# 22. UI/UX Requirements

Use:

- consistent spacing
- modern cards/panels
- keyboard support
- accessible contrast
- responsive resizing
- tooltips
- non-blocking notifications
- empty states
- loading states
- useful error dialogs
- status badges
- expandable advanced controls

Keyboard shortcuts:

- Ctrl/Cmd+V — paste URL
- Enter — fetch
- Ctrl/Cmd+Enter — download
- Ctrl/Cmd+F — search
- Space — pause/resume selected task
- Delete — remove selected task

No emoji-heavy controls. Use icons with text where useful.

---

# 23. Progress / Telemetry

Parse structured or stable yt-dlp progress output.

Track:

- percent
- downloaded bytes
- total bytes
- speed
- ETA
- elapsed
- stage

Stages:

```
Preparing
Downloading
Merging
Post-processing
Finalizing
```

Progress updates should be throttled so the UI does not redraw thousands of times per minute.

Keep raw logs behind a Details panel.

---

# 24. Error Model

Create typed errors:

- InvalidURL
- ExtractorUnavailable
- AuthenticationRequired
- NetworkError
- ProxyError
- TorError
- FormatUnavailable
- SubtitleUnavailable
- FFmpegMissing
- DiskFull
- PermissionDenied
- Cancelled
- EngineError
- PostProcessingError

Primary UI error should contain:

1. what failed
2. reason if known
3. retry suitability
4. suggested action

Example:

```
Format unavailable

The requested 4K MP4 combination is not available.

Available:
• 2160p WebM
• 1440p MP4
• 1080p MP4

[Choose Another Format]
[Retry with Best Available]
```

Do not expose raw stack traces by default.

---

# 25. yt-dlp Engine Abstraction

Create `YtDlpEngine`.

Responsibilities:

- executable discovery
- version detection
- command execution
- stdout/stderr streaming
- progress events
- cancellation
- timeout
- exit code handling
- error classification
- environment setup

Use argument arrays.

Do not invoke a shell for ordinary command execution.

---

# 26. Command Builder

Make yt-dlp command construction pure and testable.

Input:

`DownloadTask + DownloadProfile + EngineSettings`

Output:

`list[str]`

Cover:

- format selection
- merge output
- output template
- subtitles
- metadata
- thumbnail
- chapters
- playlist selection
- proxy
- Tor
- speed limits
- retries
- archive
- cookies/auth in later phase
- extractor args

Every option should have tests.

---

# 27. Download Archive / Duplicate Prevention

Implement a persistent download archive keyed by source/media ID where available.

Behavior:

```
media identified
 ↓
archive contains it?
 ├─ yes → skip
 └─ no  → download
```

Support:

- archive enabled/disabled
- inspect archive
- remove archive entry

Do not rely only on filename existence because filenames can change.

---

# 28. Temporary Files / Cleanup

Use clean storage areas:

```
downloads/
temp/
logs/
database/
cache/
```

On success:

- finalize media
- finalize metadata
- record history
- clean temp state

On failure:

- preserve resumable data where useful
- clean obvious orphaned temp files
- expose cleanup action

Add:

**Clean Temporary Files**

with approximate size.

---

# 29. Persistence

Move from ad-hoc JSON toward SQLite once queue/history complexity grows.

Tables:

- tasks
- task_events
- history
- profiles
- playlists
- playlist_items
- settings
- download_archive

Use schema versioning and migrations.

A future release should support:

- crash recovery
- indexed search
- history filtering
- durable queue
- playlist persistence

Do not silently discard existing JSON settings/history. Write migration logic.

---

# 30. Logging

Structured logs:

- timestamp
- level
- component
- task ID
- message

Levels:

- DEBUG
- INFO
- WARNING
- ERROR

Separate application logs from engine logs.

Redact:

- passwords
- cookie data
- authorization headers
- API tokens
- proxy credentials

Use log rotation.

---

# 31. Security

Treat arbitrary URLs and filenames as untrusted input.

Requirements:

- safe subprocess execution
- no unnecessary shell usage
- sanitized file paths
- filename sanitization
- secret redaction
- no credentials in logs
- safe temporary files
- no automatic telemetry by default

Do not interpret UI text fields as arbitrary shell commands.

---

# 32. Cross-Platform Support

Target:

- Windows 10/11
- Linux
- macOS

Test:

- Unicode filenames
- path separators
- process termination
- executable discovery
- filesystem permissions
- proxy/Tor
- FFmpeg discovery

Create platform utilities for:

- open file
- open folder
- executable detection
- config directories
- data directories
- process signaling

---

# 33. Packaging

Prepare:

- Windows executable
- Linux binary/package
- macOS app bundle where feasible

Decide explicitly whether FFmpeg is:

- bundled
- externally installed
- installed by setup

Do not silently download large dependencies.

Show version information in the app.

---

# 34. yt-dlp Updates

Show:

- installed version
- check for update
- update button
- optional auto-update

Update rules:

- do not update during active downloads
- preserve configuration
- verify updated executable
- report/rollback failures where possible

---

# 35. Testing

## Unit tests

Cover:

- format selectors
- command builder
- filename templates
- playlist indices
- subtitle selection
- audio selection
- retry classification
- proxy routing
- state transitions
- progress parsing
- settings migrations
- error classification

## Integration tests

Cover:

- info extraction
- playlist extraction
- representative video download
- audio extraction
- subtitle download
- metadata embedding
- FFmpeg merging
- cancellation
- retry
- resume

Use controlled/public test sources where practical.

## UI tests

Cover:

- Fetch
- Add to Queue
- Download
- Playlist selection
- pause/resume
- settings persistence
- restart recovery

---

# 36. Event-Driven Application Layer

Avoid having the controller mutate widgets directly everywhere.

Create events:

- task_created
- task_updated
- task_status_changed
- task_progress
- task_completed
- task_failed
- info_loaded
- playlist_loaded
- diagnostics_updated

UI subscribes to application events.

This makes future GUI redesign much easier.

---

# 37. Performance

Requirements:

- UI never blocks on network/process operations
- 1,000+ playlist items must remain responsive
- thumbnail loading must be asynchronous
- queue progress updates must be throttled
- disk writes must be batched/debounced where safe
- avoid full queue redraws on every progress event

---

# 38. Download Profiles

Users can create reusable profiles.

Example: `1080p MP4`

- MP4
- max 1080p
- preferred compatible video codec
- best compatible audio
- metadata
- thumbnail
- English subtitles

Example: `Archive`

- MKV
- highest available quality
- multiple/all audio when supported
- all subtitles
- chapters
- metadata
- thumbnail
- no artificial speed limit

Profiles are applicable to:

- single URL
- multiple URLs
- playlists

When queued, snapshot the resolved profile values into the task so changing a profile later does not modify existing tasks.

---

# 39. Advanced Features — Later

Do not block the stable core release for these:

- download time ranges
- SponsorBlock controls
- built-in preview player
- audio waveform
- scheduled downloads
- automatic playlist refresh
- watch folders
- local HTTP API
- browser extension
- remote downloader mode
- checksum verification
- bandwidth schedules
- hardware-accelerated FFmpeg selection
- optional aria2 backend

---

# 40. Roadmap

## Phase 1 — Stabilize Existing Core

- [ ] characterize current behavior with tests
- [ ] introduce UUID task IDs
- [ ] typed task statuses
- [ ] harden subprocess handling
- [ ] improve progress parser
- [ ] cover current command builder with tests
- [ ] improve yt-dlp/FFmpeg diagnostics

**Exit:** current features continue to work and core command generation is covered.

## Phase 2 — Discovery

- [ ] normalized MediaInfo
- [ ] StreamInfo
- [ ] AudioTrackInfo
- [ ] SubtitleInfo
- [ ] full metadata fetch
- [ ] format matrix
- [ ] audio-track UI
- [ ] subtitle UI
- [ ] playlist metadata

**Exit:** user can inspect what is actually available before downloading.

## Phase 3 — Download Profiles

- [ ] profile model
- [ ] preset profiles
- [ ] custom builder
- [ ] metadata controls
- [ ] thumbnail controls
- [ ] subtitle controls
- [ ] audio controls
- [ ] filename templates

**Exit:** one profile works consistently for videos and playlists.

## Phase 4 — Real Queue

- [ ] persistent queue
- [ ] state machine
- [ ] worker pool
- [ ] task IDs
- [ ] per-task progress
- [ ] priority
- [ ] pause/resume
- [ ] cancel
- [ ] retry
- [ ] reorder
- [ ] startup recovery

**Exit:** queue becomes the primary execution engine.

## Phase 5 — Playlist Manager

- [ ] playlist page
- [ ] bulk selection
- [ ] search/filter
- [ ] task expansion
- [ ] saved playlists
- [ ] playlist history

## Phase 6 — Network

- [ ] proxy profiles
- [ ] proxy test
- [ ] Tor service
- [ ] routing status
- [ ] timeout/retry
- [ ] credential redaction

## Phase 7 — History + Diagnostics

- [ ] history database
- [ ] filters/search
- [ ] retry from history
- [ ] diagnostics report
- [ ] temp cleanup

## Phase 8 — UI Rewrite

- [ ] sidebar navigation
- [ ] Home
- [ ] Queue
- [ ] Playlists
- [ ] History
- [ ] Settings
- [ ] Diagnostics
- [ ] reusable UI components
- [ ] keyboard shortcuts
- [ ] light/dark theme

Do not prioritize cosmetics over queue reliability.

## Phase 9 — Packaging + QA

- [ ] Windows release
- [ ] Linux release
- [ ] macOS release
- [ ] installer/setup documentation
- [ ] release smoke tests
- [ ] upgrade tests

---

# 41. Immediate First Sprint

Implement in this exact order:

1. Inspect existing code paths.
2. Create domain enums/models.
3. Add UUID task IDs.
4. Introduce `YtDlpEngine`.
5. Introduce structured progress events.
6. Move command generation into pure `CommandBuilder`.
7. Add command-builder unit tests.
8. Define queue state machine.
9. Persist queue state.
10. Add startup recovery.
11. Refactor `AppController` toward application services.
12. Replace URL-based queue identity with task ID.
13. Normalize fetched video/audio/subtitle information.
14. Build a structured format/audio/subtitle selection UI.
15. Rebuild Queue page against the task model.

Only after this should major visual polish be done.

---

# 42. Git Workflow

Prefer small, reviewable commits:

```
refactor: introduce domain task models
feat: add yt-dlp engine abstraction
test: cover format selector builder
feat: implement persistent task queue
feat: add queue recovery
feat: add structured media inspection
feat: add download profiles
feat: expand playlist into tasks
feat: add proxy profiles
feat: add diagnostics
refactor: split monolithic GUI
feat: add history manager
build: add release packaging
```

Do not combine a massive UI rewrite, queue rewrite and engine rewrite into one uncontrolled commit.

---

# 43. Definition of Done

A feature is complete only when:

- implementation exists
- UI exists where required
- persistence works where required
- error handling exists
- logs are useful
- tests exist
- documentation is updated
- cross-platform behavior is considered
- existing functionality is not regressed

No fake buttons. No placeholder functionality presented as complete.

---

# 44. Final Product Experience

## Single video

```
Paste URL
 ↓
Fetch
 ↓
See title/description/thumbnail/duration
 ↓
Inspect video formats
Inspect audio tracks
Inspect subtitles
 ↓
Choose profile
 ↓
Choose output + metadata + subtitle + audio options
 ↓
Download Now OR Add to Queue
 ↓
Track progress
 ↓
Pause / Resume / Retry / Cancel
 ↓
Post-process
 ↓
Completed
 ↓
Open file/folder
 ↓
History
```

## Playlist

```
Paste playlist URL
 ↓
Fetch
 ↓
Select items
 ↓
Choose profile/options
 ↓
Create one task per selected item
 ↓
Queue
 ↓
Independent progress/retry/recovery
 ↓
History
```

The end result should behave like a **real desktop download manager built around modern media extraction**, not like a thin GUI wrapped around a terminal command.

The quality bar is:

**reliability + inspection + control + recovery + compatibility + maintainability + modern UX.**

---

# 45. Antigravity Rules

1. **Inspect before changing.** Read the current implementation of each subsystem before refactoring it.
2. **Preserve working behavior.** Refactor around proven code before deleting it.
3. **One subsystem at a time.** Follow the roadmap order.
4. **No fake features.** Every visible control must perform the real operation or clearly report unsupported capability.
5. **Never block the UI thread.**
6. **Use capability detection.** Show only options actually supported by the current media/engine.
7. **Prefer task IDs over URLs.**
8. **Use explicit state transitions.**
9. **Persist durable state.**
10. **Keep compatibility with existing settings/history via migration.**
11. **Do not over-hard-code individual websites when yt-dlp already provides extractor coverage.**
12. **Never promise universal website support.**
13. **Do not bypass DRM.**
14. **Never expose credentials, cookies or tokens in logs.**
15. **Write tests before deleting/rewriting major working logic.**
16. **Do not make cosmetic UI work the priority while core execution remains unreliable.**
17. **When a website rejects extraction, report the actual error instead of inventing a workaround.**
18. **Keep the README accurate to real capabilities and limitations.**
