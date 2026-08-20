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

- Your recordings land in `~/Documents/Screen Recordings`. That is not
  QuickTime's default — it saves to the Desktop until you change it in the
  recording dialog. Point it wherever you like and set `SRC_DIR` in the config
  file to match.
- `ffmpeg` is on your `PATH`: `brew install ffmpeg`

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

### The folder has to be somewhere macOS lets an agent read

`~/Documents`, `~/Desktop` and `~/Downloads` are protected by macOS privacy
controls. A LaunchAgent gets no prompt and no error there — directory reads just
come back empty — so the agent runs on every new recording and converts nothing.
Running the command yourself still works, because your terminal has been granted
access, which makes this an easy failure to miss.

`install-agent` refuses to install against a protected folder for that reason.
Keep recordings somewhere unprotected instead:

    mkdir -p ~/Movies/"Screen Recordings"
    echo 'SRC_DIR="$HOME/Movies/Screen Recordings"' >> ~/.config/shrink-screen-recordings.conf

then point QuickTime at it and re-run `./install-agent`. Granting Full Disk
Access to `/bin/bash` also works but is far too broad a permission for this.

If you would rather leave the recordings in `~/Documents`, skip the agent and
run the command by hand when you need it.

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

It has to *wait* rather than skip and retry later. A directory watch only fires
when an entry is created or removed — appending to an existing file leaves the
directory's mtime alone — so the run triggered by QuickTime creating the file is
the only run that happens, and it starts when the file is still empty.
