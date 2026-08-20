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

## Requirements

    brew install ffmpeg

## Use

    shrink-screen-recordings

Converts every `.mov` in `~/Documents/Screen Recordings` that doesn't already
have an up-to-date `.mp4` in `Compressed/`. Safe to re-run.

    shrink-screen-recordings --help

## Automatic on save

    ./install-agent

Installs a LaunchAgent that watches the folder and converts each new recording
as it lands. Logs to `~/Library/Logs/shrink-screen-recordings.log`.

    ./install-agent --uninstall

## Configuration

Flags win over the config file, which wins over the defaults.

    ~/.config/shrink-screen-recordings.conf

Shell assignments:

    SRC_DIR="$HOME/Documents/Screen Recordings"
    MAX_WIDTH=1920
    MAX_HEIGHT=1080
    FPS=24
    CRF=23          # lower is better quality, 18-28 sensible
    KEEP_AUDIO=yes

The frame is fitted inside `MAX_WIDTH x MAX_HEIGHT` keeping aspect ratio, and is
never upscaled. Halving a Retina capture (so 2048 wide from 4096) keeps text
crisper than an arbitrary width, because it lands on exact logical pixels.

## Notes

Recordings are written progressively, so the script waits for a file's byte
count to stop changing before touching it. Without that, a recording caught
mid-save transcodes truncated.
