# Custom Firefox "media dump" build — setup, build, update, package

This is a personal, never-upstreamed fork of Firefox that adds a
"right-click a playing video/audio element and save it" capability that
works regardless of how the page is streaming it (MSE, blob-backed, or a
plain URL) — see [Feature summary](#feature-summary) at the bottom for what
it actually does, and the patch commit messages (`patches/*.patch`, or
`git log` on the `media-dump` branch) for the "why" behind each change.

This doc is organized around the four things you'll actually want to do:
[Bootstrap](#1-bootstrap-new-pc-from-scratch) a new machine, [build](#2-building)
the browser, [update](#3-updating-pull-upstream-while-keeping-changes) it against
new upstream Firefox code, and [package/transport](#4-packaging--moving-to-another-pc)
a finished build elsewhere. Read [Gotchas](#gotchas-read-before-debugging) once,
before debugging anything that looks weird — most weirdness here has already
been hit and explained.

## What's actually archived, and why

`mozilla-source` is a full clone of Mozilla's entire multi-decade history
(many GB). That's never what gets archived here — it's fully reproducible
by cloning Mozilla's own public repo again, and archiving a redundant copy
of public upstream history would be wasteful. What's archived is only the
*delta*:

- The commits unique to the `media-dump` branch (as patch files — see
  `patches/` next to this file).
- This `.personal-build/` directory itself (these scripts, the poison-paths
  manifest, this doc) — tracked as part of those same commits, so it always
  travels with the code.

---

## 1. Bootstrap (new PC from scratch)

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
     .mozbuild\        <- created by `mach bootstrap`
     .cargo\ .rustup\  <- created by `mach bootstrap` / rustup
   ```
   The original build used `E:\mozilla\` as `<root>`. If you put things
   somewhere else, adjust the hardcoded `E:\mozilla\mozilla-build` fallback
   path near the top of `update-and-build.ps1` / `package-for-transport.ps1`
   (they only use it if `$env:MOZILLABUILD` isn't already set) and the
   relative paths in `run-media-dump-build.bat`.

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

   This exact procedure has been verified: a throwaway branch created from
   a plain `main` had all 12 patches applied via `git am`, and the result
   was byte-for-byte identical (`git diff --stat` empty) to the real
   `media-dump` branch. It works.

4. **Bootstrap the toolchain:**
   ```
   ./mach bootstrap
   ```
   This installs rustup, cargo, clang, nasm, etc. into `.mozbuild`/
   `.cargo`/`.rustup`. It's also what the repo's own `AGENTS.md` recommends
   for getting `searchfox-cli` if you want to explore the codebase with an
   AI agent's help again later.

   Also recreate the local git exclude entry (this doesn't travel with the
   clone or the patches — see [Gotchas](#gotchas-read-before-debugging)):
   ```
   echo /clang/ >> .git/info/exclude
   ```

Once bootstrap is done, go to [Building](#2-building) below for the first
real build.

---

## 2. Building

**First build, or after a fresh bootstrap:**
```
$env:MOZILLABUILD = "<root>\mozilla-build"
./mach.ps1 build
```
This is a full build and can take a long time (tens of minutes to a couple
hours depending on hardware).

**Day-to-day rebuilds** — same command, `mach` only redoes what's stale.
If you only touched a `.cpp` file (not a header listed in a `moz.build`'s
`EXPORTS`), `./mach build binaries` is faster (skips front-end/packaging
steps) — see the stale-header gotcha below before relying on this after
editing a header, though.

**Run the build:**
```
.\.personal-build\run-media-dump-build.bat
```
(or `./mach run` directly, once `MOZILLABUILD` is set in your shell). A
convenience copy of the `.bat` normally also sits one level up, sibling to
`mozilla-source\`/`mozilla-build\`, for a double-clickable launcher.

You do **not** need `MOZ_DISABLE_CONTENT_SANDBOX` for the media-dump feature
to work — it writes through a parent-process IPC round trip now, not a
direct file write from the sandboxed content process.

---

## 3. Updating (pull upstream, keep your changes)

```
.\.personal-build\update-and-build.ps1
```

What it does, in order:
1. Refuses to run if you're not on `media-dump` or have uncommitted changes
   (stops and tells you, doesn't guess).
2. `git fetch origin`, then hard-resets `main` to `origin/main` — safe only
   because `main` is a pure mirror with **zero** local commits, by
   convention (never commit directly to `main`).
3. Rebases `media-dump` onto the refreshed `main`. If this hits conflicts,
   it stops and prints the conflicting files — resolve by hand
   (`git rebase --continue`); the script never auto-resolves or aborts a
   rebase for you.
4. Re-purges the AI-agent breadcrumb files (see
   [Gotchas](#gotchas-read-before-debugging)) — upstream keeps re-adding
   these on every pull, so this step is what keeps them gone.
5. Runs `mach build`. If it fails with a message mentioning "clobber"
   (upstream vendored something that invalidates the object dir — has
   happened before with a libwebrtc bump), it auto-clobbers and retries
   once, since that failure mode is routine, not something to stop and ask
   about each time.

Flags: `-SkipBuild` (stop after the git/purge steps, don't build) and
`-Package` (also run `mach build package` at the end — prefer
`package-for-transport.ps1` instead when you actually want to ship the
result somewhere, since it collects and labels the artifacts properly
instead of leaving them buried under the obj dir).

---

## 4. Packaging & moving to another PC

```
.\.personal-build\package-for-transport.ps1 -Installer
```

This builds the **current** state (run `update-and-build.ps1` first if you
want latest upstream folded in) and produces `mach build package`'s
artifacts, then collects them into a timestamped, commit-labeled folder:

```
E:\mozilla\packaged-builds\<yyyy-MM-dd_HHmmss>-<short-commit>\
    firefox-*.zip                    <- portable build
    firefox-*.installer.exe          <- Windows installer (only with -Installer)
    HOW-TO-RUN.txt                   <- generated, self-contained instructions
```

`-Installer` controls whether the installer `.exe` gets copied into that
folder too — it's always built as an automatic side effect of
`mach build package` on Windows either way (there's no separate
`mach build installer` target; that name doesn't exist, confirmed by trying
it and getting "No rule to make target 'installer'").

`-OutDir <path>` changes the collection folder (default
`E:\mozilla\packaged-builds`).

**On the target PC:** copy the timestamped folder over (USB, network share,
whatever) and either extract-and-run the `.zip`'s `firefox.exe` directly, or
run the installer `.exe` like any normal Windows installer. **No special
setup, env vars, or sandbox flags are needed on the target machine** — the
media-dump feature works through the normal content sandbox, so a
packaged/installed build behaves exactly like a stock portable/installed
Firefox build from the outside.

---

## Gotchas (read before debugging)

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
  so recreate that entry — see step 4 of Bootstrap above). Always stage by
  explicit path in this repo, never a wildcard.
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

## Feature summary

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
