# Custom Firefox "media dump" build — recovery notes

This is a personal, never-upstreamed fork of Firefox that adds a
"right-click a playing video/audio element and save it" capability that
works regardless of how the page is streaming it (MSE, blob-backed, or a
plain URL) — see the design notes and full history in this branch's commit
log for the "why". This file is what you need to get the environment
working again from scratch, e.g. on a new PC. It captures hard-won,
non-obvious gotchas from the original build session, not just the happy
path.

## What's actually archived, and why

`mozilla-source` is a full clone of Mozilla's entire multi-decade history
(many GB). That's never what gets archived here — it's fully reproducible
by cloning Mozilla's own public repo again, and archiving a redundant copy
of public upstream history would be wasteful. What's archived is only the
*delta*:

- The commits unique to the `media-dump` branch (as patch files — see
  `patches/` next to this file, or wherever this SETUP.md ended up).
- This `.personal-build/` directory itself (these scripts, the poison-paths
  manifest, this doc) — tracked as part of those same commits.

## Recreating the environment on a new PC

1. **Clone mozilla-central:**
   ```
   git clone https://github.com/mozilla-firefox/firefox mozilla-source
   ```
   This will take a while and use significant disk space (tens of GB).

2. **Recreate the sibling directory layout.** The scripts here assume a
   layout like:
   ```
   <root>\
     mozilla-source\   <- the clone from step 1
     mozilla-build\    <- MozillaBuild (https://ftp.mozilla.org/pub/mozilla/libraries/win32/MozillaBuildSetup-latest.exe)
     .mozbuild\         <- created by `mach bootstrap`
     .cargo\ .rustup\   <- created by `mach bootstrap` / rustup
   ```
   If you put things somewhere other than `E:\mozilla\`, you'll need to
   adjust the hardcoded `E:\mozilla\mozilla-build` fallback path near the
   top of `update-and-build.ps1` / `package-for-transport.ps1` (they only
   use it if `$env:MOZILLABUILD` isn't already set) and the relative paths
   in `run-media-dump-build.bat`.

3. **Create the `media-dump` branch and apply the archived patches:**
   ```
   cd mozilla-source
   git checkout -b media-dump
   git am /path/to/archive/patches/*.patch
   ```
   `git am` applies them as real commits, preserving the original commit
   messages (which have a lot of the "why" behind non-obvious code, e.g.
   why several things needed real bug fixes discovered only through actual
   `ffprobe`-verified testing, not just compiling).

4. **Bootstrap the toolchain:**
   ```
   ./mach bootstrap
   ```
   This installs rustup, cargo, clang, nasm, etc. into `.mozbuild`/
   `.cargo`/`.rustup`. It's also what the repo's own `AGENTS.md` recommends
   for getting `searchfox-cli` if you want to explore the codebase with an
   AI agent's help again later.

5. **Set `MOZILLABUILD` and build:**
   ```
   $env:MOZILLABUILD = "<root>\mozilla-build"
   ./mach.ps1 build
   ```
   Or just run `.\.personal-build\update-and-build.ps1` — it sets this
   itself, and additionally pulls latest upstream + rebases first (safe to
   run even on a brand new checkout; it'll just be a no-op pull).

6. **Run it:** `.\.personal-build\run-media-dump-build.bat` (or copy it up
   one level next to `mozilla-build`/`mozilla-source` for convenience, like
   the original layout had it).

## Ongoing workflows

- **Pulling upstream updates while keeping this branch's changes:**
  `.\.personal-build\update-and-build.ps1`. Rebases `media-dump` onto the
  freshly-pulled `main`, re-purges the AI-agent breadcrumb files (see
  below), and rebuilds. Auto-detects and recovers from the "upstream
  changed low-level build files, need to clobber" failure mode.
- **Packaging a build to move to another PC:**
  `.\.personal-build\package-for-transport.ps1 -Installer`. Collects a
  portable zip (and the installer .exe) into a timestamped folder under
  `E:\mozilla\packaged-builds\` (adjust `-OutDir` if needed), with its own
  `HOW-TO-RUN.txt`. **No special setup is needed on the target PC** for
  this to work — capture writes go through the parent process now, so a
  packaged/installed build just runs like a normal Firefox build.

## Non-obvious gotchas discovered building this (read before debugging)

- **`MOZILLABUILD` isn't auto-detected.** `mach` defaults to expecting
  MozillaBuild at `C:\mozilla-build`; if it's anywhere else, `mach` fails
  with `AssertionError: MozillaBuild was not found`. Both scripts here set
  this env var themselves.
- **`mach build binaries` can compile against a *stale* exported header**
  if you (or an agent) edits an `EXPORTS`-listed header while a previous
  background build is still running — it skips the export/header-copy
  step that a plain `mach build` (or `mach build export`) does. If you get
  a confusing "declaration doesn't match" error right after editing a
  header, this is almost certainly why. Non-exported `.cpp`-only edits are
  safe with `build binaries`.
- **Upstream occasionally vendors something that requires a clobber**
  (this happened with a libwebrtc vendoring bump partway through the
  original session) — `mach build` fails with a message mentioning
  "clobber". `update-and-build.ps1` detects this and runs `mach clobber`
  + retries automatically; if you're running `mach build` by hand and hit
  this, just run `./mach clobber` then build again (it'll be a full
  rebuild, ~20-30+ min depending on hardware).
- **`NS_WARNING` is a complete no-op outside `DEBUG` builds** (see
  `xpcom/base/nsDebug.h`) — this build isn't a DEBUG build, so `NS_WARNING`
  calls anywhere in this feature's code are silently swallowed. On top of
  that, **content-process `stderr` is not visible via `mach run`'s
  `-attach-console`** the way the parent process's is — so neither
  mechanism reaches you for diagnostics in `dom/media` code (which runs in
  the content process for a normal tab). The working pattern used here:
  encode diagnostic info into something that crosses the process boundary
  on its own, like the dump's own filename (see `MediaDumpMuxer.cpp`'s
  `ShortCodecTag`) — not console logging.
- **`mach build installer` is not a real target** (confirmed: fails with
  "No rule to make target 'installer'"). The installer `.exe` is produced
  automatically as a side effect of `mach build package` on Windows.
- **`git add -A`/`git add .` in this repo will stage the untracked
  `clang/` toolchain download** (hundreds of MB) since it isn't covered by
  a committed `.gitignore` — it's excluded via this checkout's local
  `.git/info/exclude` instead (which doesn't travel with clones/patches,
  so recreate that entry: add a `/clang/` line to the new checkout's own
  `.git/info/exclude`). Always stage by explicit path in this repo, never
  a wildcard.
- **The AI-agent breadcrumb files are deliberately deleted, not missing by
  accident.** `.agents/skills/`, `.claude/skills/`, `.claude/settings.json`,
  `.codex/config.toml`, and top-level `README.md`/`CODE_OF_CONDUCT.md`/
  `SECURITY.md` are intentionally removed on this branch (considered
  untrustworthy/undesirable content to have influencing any agent working
  in this tree) — see `poison-paths.txt` for the exact list. Upstream keeps
  re-adding these on every pull; `update-and-build.ps1`'s purge step is
  what keeps them gone. If you ever expand this list, update
  `poison-paths.txt` to match.
- **`main` should always stay a pure, untouched mirror of `origin/main`.**
  All feature work lives on `media-dump`; the update workflow relies on
  being able to safely `git reset --hard origin/main` on `main` without
  losing anything, specifically because nothing is ever committed there
  directly.

## Feature summary (as of this archive)

- Right-click "Save Video As"/"Save Audio As" on a playing `<video>`/
  `<audio>` element: works even when there's no reusable network URL
  (MSE/blob-backed content), by capturing demuxed samples directly instead
  of re-fetching. Falls back to the normal instant download when a real
  URL *is* available, so this is strictly additive.
- Hooks `MediaFormatReader::OnVideoDemuxCompleted`/`OnAudioDemuxCompleted`
  (`dom/media/MediaFormatReader.cpp`) — the point every playback pipeline
  converges at before decode — so it doesn't matter whether the page used
  MSE, a blob URL, or a plain URL.
- Muxes into a single Matroska (`.mkv`) file per element via
  `dom/media/MediaDumpMuxer.{h,cpp}`, reusing Gecko's existing low-level
  EBML primitives (`media/libmkv`, the same ones `MediaRecorder`'s
  `WebMWriter` uses) but reading each track's actual codec instead of
  assuming one.
- Supported codecs: video VP8/VP9/H.264/AV1, audio Opus/Vorbis/AAC. Not
  yet supported: HEVC and anything more exotic (falls back to audio-only
  output, now clearly labeled in the filename e.g.
  `-unsupported-video-hevc-...`, rather than silently producing a broken
  file).
- Content-process sandbox stays enabled: file writes go through a new
  `PContent` IPC round trip (`RequestMediaDumpFile`, `dom/ipc/PContent.ipdl`
  + `ContentParent.cpp`) modeled on Gecko's existing anonymous-temp-file
  mechanism, except it opens a real, user-chosen, persistent path.
- DRM/EME content is excluded: encrypted samples are dropped at the sample
  level (`MediaRawData::mCrypto.IsEncrypted()`), never written.
- **Known limitation:** capture only starts from the *next* keyframe after
  arming, not retroactively — if you want the whole video, seek/restart
  playback to 0:00 right after picking the save folder, since a decoder
  restart always delivers a keyframe next. No pre-roll buffer exists (yet)
  for "capture what I've already been watching."
