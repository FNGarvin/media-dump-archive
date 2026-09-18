# media-dump archive

This is **not** a copy of the Firefox source tree — it's just the delta:
12 commits' worth of patches implementing a custom "save any playing
video/audio" feature on top of `mozilla-central`, plus the build tooling
used to develop it.

- **`SETUP.md`** — start here. Full recovery instructions and every
  non-obvious gotcha discovered while building this.
- **`patches/`** — the actual commits, as `git format-patch` output.
  Apply with `git am patches/*.patch` onto a `media-dump` branch created
  off a fresh `mozilla-central` clone (see SETUP.md for the exact steps).

Why not just push the whole `mozilla-source` checkout? It's a full clone
of Mozilla's entire multi-decade history (many GB) — archiving a redundant
copy of public upstream history would be wasteful, and Mozilla's own
servers already preserve it forever. Only the delta above is actually
irreplaceable.
