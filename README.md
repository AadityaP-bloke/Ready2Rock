# Ready2Rock

Cross-platform tooling that turns a pile of music files into a clean, Rockbox-ready library.
It ships both a **CLI** and a **PyQt6 GUI** built on the same `Config` / `process()` core.

## Features

- **Convert** audio to MP3, Ogg Vorbis, Opus, FLAC, ALAC, M4A/AAC, WAV or AIFF at a chosen bitrate
  (or copy the original streams unchanged). Uses a **system ffmpeg** - nothing is bundled.
- **Scrape metadata from MusicBrainz** with fuzzy scoring, a 1 req/s rate limit and a local
  response cache. You choose exactly which fields are allowed to be applied.
- **Normalise artwork** to exactly 600×600 **baseline (non-progressive) JPEG**, from embedded
  art or a supplied image, written as `cover.jpg` and embedded where the format supports it.
- **Organise files** using a path template such as
  `{artist}/{album}/{track_number:02d} - {title}` (or `artist>album>track`).
- Never modifies source files. Safe to re-run.

## Install

Requires Python 3.11+ and ffmpeg on `PATH` (or `READY2ROCK_FFMPEG` pointing at it).

```bash
python -m pip install -e '.[gui,test]'
pytest
```

## Packages

Ready2Rock can be packaged as `Ready2Rock.app` (macOS), `Ready2Rock.exe`
(Windows) and `Ready2Rock-<arch>.AppImage` (Linux). The platform-specific
build scripts live in `packaging/`, and `.github/workflows/build.yml` builds
all three on their native runners. See `packaging/README.md` for details.

The packaged apps still use your system ffmpeg; ffmpeg is not bundled.

## GUI

```bash
ready2rock-gui
# or
ready2rock gui
```

The window is built around a **file-browser style review list**: add files or a folder, then
before anything is written you can:

- check/uncheck individual tracks
- edit metadata inline in the table, or use the **Track metadata** tab to edit every field
  of the selected file, see its embedded artwork, and copy edits to all checked tracks
- see the **Planned path** for every row update live as you edit tags or change the pattern
- choose conversion, MusicBrainz field selection, artwork and folder-layout options in the
  **Processing options** tab
- run with **Dry run** first, then press **Prepare files…**

Manual edits are applied *after* MusicBrainz, so they always win. Only fields you actually
changed override the scraped values. Scanning and processing run on background threads, so
the UI stays responsive. Config can be loaded/saved from the File menu.

## CLI

```bash
# Check that ffmpeg is usable
ready2rock ffmpeg

# List selectable metadata fields
ready2rock fields

# Preview a run (no writes, no ffmpeg needed)
ready2rock process "/path/to/music" -o "/path/to/out" --dry-run

# Convert to 192 kbps MP3, fix art, scrape only selected fields, and organise
ready2rock process "/path/to/music" -o "/path/to/out" \
  --format mp3 --bitrate 192k \
  --artwork --artwork-fit cover \
  --musicbrainz --mb-fields title,artist,album,album_artist,track_number,total_tracks,date,genre \
  --mb-threshold 0.8 \
  --pattern '{album_artist}/{album}/{disc_number}-{track_number:02d} - {title}'
```

Inputs may be files, directories or glob patterns (quote globs so your shell does not expand
them). Directories are scanned recursively by default; use `--no-recursive` to disable.

### Config files

Every option can live in a TOML or JSON file. Generate a starter:

```bash
ready2rock init-config ready2rock.toml
ready2rock process --config ready2rock.toml
```

CLI arguments override config-file values. See `ready2rock/config.py` for all keys.

## Architecture

```
src/ready2rock/
  fields.py        field names, aliases, parsing
  models.py        Metadata / Track / results
  config.py        Config, validation, TOML+JSON persistence
  utils.py         filename sanitisation, path helpers
  ffmpeg.py        ffmpeg detection + command construction
  artwork.py       embedded-art extraction, 600x600 baseline JPEG
  tagging.py       mutagen read/write, ID3v2.3, art embedding
  musicbrainz.py   scored lookup, rate limiting, caching
  organizer.py     path templates -> sanitised destination paths
  pipeline.py      orchestration; the entry point for the GUI
  cli.py           argparse interface
  gui/             PyQt6 interface (browser model, metadata editor, settings, workers)
```

The GUI calls `process(config, sources=..., metadata_overrides=..., progress=callback)` from
a `QThread` and feeds the returned `ProcessingResult` back into the browser model.
