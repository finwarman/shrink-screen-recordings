# shrink-screen-recordings

QuickTime records at your display's native resolution and refresh rate, and
offers no setting for either. On a Retina + ProMotion Mac that means 4096x2480
at 120fps, roughly 1MB per second — a one minute clip is too big to attach to a
pull request.

This fits recordings inside 1080p at 24fps. Measured on a real 23 second clip:

| | size |
| --- | --- |
| original (4096px, 120fps) | 21M |
| 1080p, 24fps, crf 23 | 836K |
| 720p, 30fps, crf 23 | 544K |

Originals are never modified. Output goes to a `Compressed/` subfolder as
`.mp4`, which is what GitHub wants anyway.

## Assumes

- Your recordings land in `~/Movies/Screen Recordings`. That is not QuickTime's
  default — it saves to the Desktop until you change it in the recording dialog,
  and QuickTime has no setting for it outside that dialog. Point it wherever you
  like and set `SRC_DIR` in the config file to match.
- `~/Movies` rather than `~/Documents` on purpose: see
  [the folder has to be readable](#the-folder-has-to-be-somewhere-macos-lets-an-agent-read).
- `ffmpeg` is on your `PATH`: `brew install ffmpeg`

## Use

    shrink-screen-recordings

Converts every `.mov` in `~/Movies/Screen Recordings` that doesn't already have
an up-to-date `.mp4` in `Compressed/`. Safe to re-run.

    shrink-screen-recordings --help

## Automatic on save

    ./install-agent

Installs a LaunchAgent that watches the folder and converts each new recording
as it lands. Logs to `~/Library/Logs/shrink-screen-recordings.log`.

    ./install-agent --uninstall

### The folder has to be somewhere macOS lets an agent read

`~/Documents`, `~/Desktop` and `~/Downloads` are protected by macOS privacy
controls. A LaunchAgent gets no prompt and no error there — directory reads just
come back empty — so the agent runs on every new recording and converts nothing.
Running the command yourself still works, because your terminal has been granted
access, which makes this an easy failure to miss.

`install-agent` refuses to install against a protected folder for that reason,
which is why the default is `~/Movies/Screen Recordings`. If you keep recordings
in `~/Documents` anyway, skip the agent and run the command by hand — granting
Full Disk Access to `/bin/bash` also works, but is far too broad a permission
for this.

## Configuration

Flags win over the config file, which wins over the defaults.

    ~/.config/shrink-screen-recordings.conf

Shell assignments:

    SRC_DIR="$HOME/Movies/Screen Recordings"
    MAX_WIDTH=1920
    MAX_HEIGHT=1080
    FPS=24
    CRF=23          # lower is better quality, 18-28 sensible
    KEEP_AUDIO=yes

The frame is fitted inside `MAX_WIDTH x MAX_HEIGHT` keeping aspect ratio, and is
never upscaled. Halving a Retina capture (so 2048 wide from 4096) keeps text
crisper than an arbitrary width, because it lands on exact logical pixels.

## Notes

Recordings are written progressively, so the script waits for a file to be
closed and its byte count to settle before touching it. Without that, a
recording caught mid-save transcodes truncated.

Byte count alone is not enough. A still screen encodes to almost nothing, so a
recording sitting on static content can hold the same size for seconds and look
finished — and if that partial transcode lands after the recording ends, the
truncated output looks up to date and never gets redone. Whoever is recording
still holds the file open, so that is the primary signal.

It has to *wait* rather than skip and retry later. A directory watch only fires
when an entry is created or removed — appending to an existing file leaves the
directory's mtime alone — so the run triggered by QuickTime creating the file is
the only run that happens, and it starts when the file is still empty.
