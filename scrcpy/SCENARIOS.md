# scrcpy — command scenarios for deep research

Installed build: scrcpy 4.1 at `/usr/local/bin/scrcpy` (locally built, not a
dpkg package). Every option below appears in [`help.txt`](help.txt) or
`man scrcpy`; behaviour was observed on this host with no Android device
attached.

> **Scope:** mirroring, input injection, recording, and OTG mode all touch a
> real device. Only run them against hardware you own or are authorised to
> test. Recording a device may capture personal data; handle the output as
> sensitive evidence.

> **Two host quirks shape every scenario.** The binary embeds
> `/usr/local/bin/adb`, which does not exist here, so set `ADB=/usr/bin/adb`.
> And with no device attached, every mode exits 1 with
> `Could not find any ADB device`; option validation happens first.

## How to invoke

```text
@deep-research For an authorised Android assessment with scrcpy 4.1, explain
how the client pushes and starts scrcpy-server, how the adb tunnel carries
video/audio/control sockets, how `--video-source`, `--audio-source`, and
`--keyboard`/`--mouse`/`--gamepad` change the pipeline, and how recording,
v4l2, virtual displays, and OTG mode differ. Show which option values are
validated before a device is needed, which modes exit 1 with no device, and
how the embedded adb path and `ADB` environment variable interact. Cite the
installed help text, `man scrcpy`, and observed output. Separate confirmed
behaviour from inference and state confidence.
```

Create a working directory and record the environment:

```bash
mkdir -p ~/scrcpy-demo/evidence
cd ~/scrcpy-demo
export ADB=/usr/bin/adb
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
scrcpy --version > evidence/version.txt 2>&1
```

## Scenario 1 — version, help, manual, binary, dependencies

| # | Command | Purpose |
|---|---------|---------|
| 1 | `scrcpy --version` | Version and linked libraries |
| 2 | `scrcpy --version \| head -3` | Short version line |
| 3 | `scrcpy -v` | Short version flag |
| 4 | `scrcpy --help \| head -20` | Option header |
| 5 | `scrcpy --help \| grep -c '^    --'` | Count long options |
| 6 | `scrcpy --help \| grep -A20 'Shortcuts:'` | Shortcut list |
| 7 | `scrcpy --help \| grep -A10 'Environment variables'` | Environment contract |
| 8 | `scrcpy --help \| grep -A6 'Exit status'` | Documented exit codes |
| 9 | `man scrcpy \| col -b \| head -20` | Manual page |
| 10 | `man scrcpy \| col -b \| grep -c '^       --'` | Options in the manual |
| 11 | `command -v scrcpy` | Resolve the binary |
| 12 | `readlink -f "$(command -v scrcpy)"` | Real path |
| 13 | `file "$(command -v scrcpy)"` | Confirm a native ELF |
| 14 | `ls -l "$(command -v scrcpy)"` | Size and timestamp |
| 15 | `dpkg -S /usr/local/bin/scrcpy 2>&1 \| head -2` | Not a dpkg file |
| 16 | `ldd "$(command -v scrcpy)" \| head -12` | Linked libraries |
| 17 | `scrcpy --version \| grep -E 'SDL\|libavcodec\|libusb'` | Dependency versions |
| 18 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Is scrcpy packaged by Debian here, or locally built?
- Which SDL and FFmpeg versions is it linked against?
- Does the manual document options the help text omits?
- Which exit codes does the help text promise?

**Sources:** `scrcpy --version`; `scrcpy --help`; `man scrcpy`; `dpkg -S`.
**Output:** a build-provenance record and a help/manual comparison.

## Scenario 2 — adb resolution and device selection

| # | Command | Purpose |
|---|---------|---------|
| 19 | `timeout 5 scrcpy --list-displays 2>&1 \| head -4` | Default adb path fails |
| 20 | `timeout 5 scrcpy --list-displays 2>&1 \| grep -c 'usr/local/bin/adb'` | Confirm embedded path |
| 21 | `ADB=/usr/bin/adb timeout 5 scrcpy --list-displays 2>&1 \| head -3` | Correct adb path |
| 22 | `ADB=/usr/bin/adb timeout 5 scrcpy -s bogus 2>&1 \| head -3` | Bad serial |
| 23 | `ADB=/usr/bin/adb timeout 5 scrcpy -d 2>&1 \| head -3` | USB selector |
| 24 | `ADB=/usr/bin/adb timeout 5 scrcpy -e 2>&1 \| head -3` | TCP selector |
| 25 | `ANDROID_SERIAL=bogus ADB=/usr/bin/adb timeout 5 scrcpy 2>&1 \| head -3` | Environment selector |
| 26 | `ADB=/usr/bin/adb timeout 5 scrcpy --tcpip=192.0.2.10:5555 2>&1 \| head -3` | Unreachable TCP target |
| 27 | `timeout 5 scrcpy --tcpip=192.0.2.10:5555 >/dev/null 2>&1; echo "tcpip=$?"` | Blocks or fails |
| 28 | `ADB=/usr/bin/adb timeout 5 scrcpy --list-apps 2>&1 \| head -3` | App list needs a device |
| 29 | `ADB=/usr/bin/adb timeout 5 scrcpy --list-cameras 2>&1 \| head -3` | Camera list needs a device |
| 30 | `ADB=/usr/bin/adb timeout 5 scrcpy --list-encoders 2>&1 \| head -3` | Encoder list needs a device |
| 31 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-video --no-audio --no-window 2>&1 \| head -3` | Headless mode still needs a device |
| 32 | `pgrep -af 'adb.*server' \| head -2` | adb server side effect |
| 33 | `ADB=/usr/bin/adb adb kill-server 2>&1 \| head -2` | Clean up the server |
| 34 | `ADB=/nonexistent timeout 5 scrcpy 2>&1 \| head -3` | Bad ADB value |
| 35 | `SCRCPY_SERVER_PATH=/nonexistent timeout 5 scrcpy 2>&1 \| head -3` | Bad server path |

**Research questions**
- Which adb path is compiled into this build?
- Does `ADB` override it for every subcommand?
- Which selectors are checked before the server starts?
- Does `--tcpip` time out or fail fast when the host is unreachable?

**Sources:** observed output; `scrcpy --help`; `man scrcpy`.
**Output:** an adb-resolution and selector matrix.

## Scenario 3 — mirroring and control modes

| # | Command | Purpose |
|---|---------|---------|
| 36 | `ADB=/usr/bin/adb timeout 5 scrcpy` | Default mirror and control |
| 37 | `ADB=/usr/bin/adb timeout 5 scrcpy -n` | Read-only mirror |
| 38 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-control` | Long form |
| 39 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-video` | Audio only |
| 40 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-audio` | Video only |
| 41 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-playback` | No host playback |
| 42 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-video-playback` | Decode without display |
| 43 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-audio-playback` | Do not play audio |
| 44 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-window` | Headless |
| 45 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-window --no-audio` | Headless video only |
| 46 | `ADB=/usr/bin/adb timeout 5 scrcpy --require-audio` | Fail if audio breaks |
| 47 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-dup 2>&1 \| head -3` | Source switches to playback |
| 48 | `ADB=/usr/bin/adb timeout 5 scrcpy -S` | Screen off at start |
| 49 | `ADB=/usr/bin/adb timeout 5 scrcpy -w` | Stay awake |
| 50 | `ADB=/usr/bin/adb timeout 5 scrcpy -t` | Show touches |
| 51 | `ADB=/usr/bin/adb timeout 5 scrcpy --keep-active` | Simulate activity |
| 52 | `ADB=/usr/bin/adb timeout 5 scrcpy --power-off-on-close` | Screen off on exit |
| 53 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-power-on` | Do not wake at start |
| 54 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-cleanup` | Keep server and state |
| 55 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-terminal-title` | Leave terminal title alone |

**Research questions**
- Which flags imply others (`--no-window` implies no video playback)?
- Does `--no-playback` disable streams when nothing records them?
- Does `--no-cleanup` leave the server jar on the device?
- Which state does scrcpy restore on exit?

**Sources:** `scrcpy --help`; `man scrcpy`; observed output.
**Output:** a stream/control matrix with implications.

## Scenario 4 — video pipeline and codecs

| # | Command | Purpose |
|---|---------|---------|
| 56 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-codec=h264` | Default codec |
| 57 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-codec=h265` | H.265 |
| 58 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-codec=av1` | AV1 |
| 59 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-codec=vp8` | VP8 |
| 60 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-codec=vp9` | VP9 |
| 61 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-codec=bogus 2>&1 \| head -2` | Invalid codec |
| 62 | `ADB=/usr/bin/adb timeout 5 scrcpy -b 8M` | Bit rate |
| 63 | `ADB=/usr/bin/adb timeout 5 scrcpy -b 2M` | Lower bit rate |
| 64 | `ADB=/usr/bin/adb timeout 5 scrcpy -m 1024` | Max size |
| 65 | `ADB=/usr/bin/adb timeout 5 scrcpy --max-size=800` | Long form |
| 66 | `ADB=/usr/bin/adb timeout 5 scrcpy --max-size=bogus 2>&1 \| head -2` | Invalid size |
| 67 | `ADB=/usr/bin/adb timeout 5 scrcpy --max-fps=30` | Frame cap |
| 68 | `ADB=/usr/bin/adb timeout 5 scrcpy --max-fps=bogus 2>&1 \| head -2` | Invalid fps |
| 69 | `ADB=/usr/bin/adb timeout 5 scrcpy --min-size-alignment=8` | Size alignment |
| 70 | `ADB=/usr/bin/adb timeout 5 scrcpy --min-size-alignment=3 2>&1 \| head -2` | Non-power-of-2 |
| 71 | `ADB=/usr/bin/adb timeout 5 scrcpy --ignore-video-encoder-constraints` | Ignore limits |
| 72 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-buffer=50` | Playback buffer |
| 73 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-buffer=bogus 2>&1 \| head -2` | Invalid buffer |
| 74 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-downsize-on-error` | No retry |
| 75 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-mipmaps` | No mipmaps |
| 76 | `ADB=/usr/bin/adb timeout 5 scrcpy --angle=90` | Rotate content |
| 77 | `ADB=/usr/bin/adb timeout 5 scrcpy --crop=1080:1920:0:0` | Crop |
| 78 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-encoder=bogus` | Encoder selection |

**Research questions**
- Which codecs are validated on the host before a device is queried?
- Does `--max-size` preserve aspect ratio?
- Does `--angle` rotate before or after encoding?
- What happens when `--crop` exceeds the display size?

**Sources:** `scrcpy --help`; observed output.
**Output:** a codec and sizing matrix.

## Scenario 5 — audio pipeline and sources

| # | Command | Purpose |
|---|---------|---------|
| 79 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-codec=opus` | Default codec |
| 80 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-codec=aac` | AAC |
| 81 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-codec=flac` | FLAC |
| 82 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-codec=raw` | Raw |
| 83 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-codec=bogus 2>&1 \| head -2` | Invalid codec |
| 84 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=output` | Output capture |
| 85 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=mic` | Microphone |
| 86 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=playback` | Playback capture |
| 87 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=mic-unprocessed` | Unprocessed mic |
| 88 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=mic-camcorder` | Camcorder mic |
| 89 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=mic-voice-recognition` | Voice-recognition mic |
| 90 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=mic-voice-communication` | Voice-communication mic |
| 91 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=voice-call` | Call audio |
| 92 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=voice-call-uplink` | Call uplink |
| 93 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=voice-call-downlink` | Call downlink |
| 94 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=voice-performance` | Performance audio |
| 95 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-source=bogus 2>&1 \| head -2` | Invalid source |
| 96 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-bit-rate=96K` | Bit rate |
| 97 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-buffer=100` | Buffering |
| 98 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-buffer=bogus 2>&1 \| head -2` | Invalid buffer |
| 99 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-output-buffer=20` | Output buffer |
| 100 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-encoder=bogus` | Encoder selection |
| 101 | `ADB=/usr/bin/adb timeout 5 scrcpy --audio-dup 2>&1 \| head -3` | Duplication warning |
| 102 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-audio` | Disable audio |

**Research questions**
- Which audio sources require Android 11+ or a specific device build?
- Does `--audio-dup` change the source before or after validation?
- Which containers support each codec?
- Does `--require-audio` change the exit code when capture fails?

**Sources:** `scrcpy --help`; observed output.
**Output:** an audio source/codec matrix.

## Scenario 6 — recording and external output

| # | Command | Purpose |
|---|---------|---------|
| 103 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.mp4` | Record MP4 |
| 104 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.mkv` | Record MKV |
| 105 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.m4a` | Audio container rejects video |
| 106 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-video -r out.opus` | Audio-only recording |
| 107 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-video -r out.flac` | FLAC recording |
| 108 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-video -r out.wav` | WAV recording |
| 109 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.bogus 2>&1 \| head -2` | Unknown extension |
| 110 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.bin --record-format=mkv` | Format override |
| 111 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.mp4 --record-format=bogus 2>&1 \| head -2` | Invalid format |
| 112 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.mp4 --record-orientation=90` | Record rotation |
| 113 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.mp4 --record-orientation=45 2>&1 \| head -2` | Invalid rotation |
| 114 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.mp4 --no-playback` | Record without display |
| 115 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.mp4 --time-limit=30` | Time limit |
| 116 | `ADB=/usr/bin/adb timeout 5 scrcpy -r out.mp4 --time-limit=bogus 2>&1 \| head -2` | Invalid time |
| 117 | `ADB=/usr/bin/adb timeout 5 scrcpy --v4l2-sink=/dev/video10` | V4L2 output |
| 118 | `ADB=/usr/bin/adb timeout 5 scrcpy --v4l2-sink=/dev/video10 --v4l2-buffer=100` | V4L2 buffer |
| 119 | `ADB=/usr/bin/adb timeout 5 scrcpy --v4l2-buffer=100 2>&1 \| head -2` | Buffer without sink |
| 120 | `ADB=/usr/bin/adb timeout 5 scrcpy --print-fps` | FPS log |
| 121 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-playback --no-video --no-audio` | All output disabled |

**Research questions**
- Does `--record` infer the format before or after checking the device?
- Which containers reject video, and which reject audio?
- Does `--time-limit` stop cleanly at the limit?
- Does `--v4l2-buffer` require `--v4l2-sink` at parse time?

**Sources:** `scrcpy --help`; observed output.
**Output:** a recording/container matrix with exit codes.

## Scenario 7 — window, display, camera, virtual displays

| # | Command | Purpose |
|---|---------|---------|
| 122 | `ADB=/usr/bin/adb timeout 5 scrcpy --window-title=audit` | Window title |
| 123 | `ADB=/usr/bin/adb timeout 5 scrcpy --window-x=100 --window-y=100` | Window position |
| 124 | `ADB=/usr/bin/adb timeout 5 scrcpy --window-width=640 --window-height=480` | Window size |
| 125 | `ADB=/usr/bin/adb timeout 5 scrcpy --window-x=bogus 2>&1 \| head -2` | Invalid position |
| 126 | `ADB=/usr/bin/adb timeout 5 scrcpy --window-width=bogus 2>&1 \| head -2` | Invalid dimension |
| 127 | `ADB=/usr/bin/adb timeout 5 scrcpy --window-borderless` | Borderless |
| 128 | `ADB=/usr/bin/adb timeout 5 scrcpy --always-on-top` | Always on top |
| 129 | `ADB=/usr/bin/adb timeout 5 scrcpy -f` | Fullscreen |
| 130 | `ADB=/usr/bin/adb timeout 5 scrcpy --render-fit=letterbox` | Letterbox |
| 131 | `ADB=/usr/bin/adb timeout 5 scrcpy --render-fit=stretched` | Stretched |
| 132 | `ADB=/usr/bin/adb timeout 5 scrcpy --render-fit=unscaled` | Unscaled |
| 133 | `ADB=/usr/bin/adb timeout 5 scrcpy --render-fit=bogus 2>&1 \| head -2` | Invalid fit |
| 134 | `ADB=/usr/bin/adb timeout 5 scrcpy --render-driver=software` | Render driver |
| 135 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-window-aspect-ratio-lock` | Free resize |
| 136 | `ADB=/usr/bin/adb timeout 5 scrcpy --disable-screensaver` | Screensaver off |
| 137 | `ADB=/usr/bin/adb timeout 5 scrcpy --display-id=0` | Specific display |
| 138 | `ADB=/usr/bin/adb timeout 5 scrcpy --display-id=bogus 2>&1 \| head -2` | Invalid display |
| 139 | `ADB=/usr/bin/adb timeout 5 scrcpy --new-display` | Default virtual display |
| 140 | `ADB=/usr/bin/adb timeout 5 scrcpy --new-display=1920x1080` | Virtual display size |
| 141 | `ADB=/usr/bin/adb timeout 5 scrcpy --new-display=1920x1080/420` | Size and DPI |
| 142 | `ADB=/usr/bin/adb timeout 5 scrcpy --new-display=/240` | DPI only |
| 143 | `ADB=/usr/bin/adb timeout 5 scrcpy -x 2>&1 \| head -2` | Flex without new display |
| 144 | `ADB=/usr/bin/adb timeout 5 scrcpy --new-display -x` | Flex with new display |
| 145 | `ADB=/usr/bin/adb timeout 5 scrcpy --start-app=?firefox` | Prefix app match |
| 146 | `ADB=/usr/bin/adb timeout 5 scrcpy --start-app=+org.mozilla.firefox` | Force-stop then start |
| 147 | `ADB=/usr/bin/adb timeout 5 scrcpy --start-app=+?firefox` | Both prefixes |
| 148 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-facing=front` | Front camera |
| 149 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-facing=back` | Back camera |
| 150 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-size=1920x1080` | Capture size |
| 151 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-ar=16:9` | Aspect ratio |
| 152 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-ar=16:9 --camera-size=1920x1080 2>&1 \| head -2` | Mutually exclusive |
| 153 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-fps=30` | Camera fps |
| 154 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-high-speed 2>&1 \| head -2` | Needs explicit fps |
| 155 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-fps=120 --camera-high-speed` | High-speed capture |
| 156 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-torch` | Torch on |
| 157 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-zoom=2` | Zoom |
| 158 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=camera --camera-id=0` | Camera by id |
| 159 | `ADB=/usr/bin/adb timeout 5 scrcpy --camera-facing=front 2>&1 \| head -2` | Camera option without source |
| 160 | `ADB=/usr/bin/adb timeout 5 scrcpy --video-source=bogus 2>&1 \| head -2` | Invalid source |

**Research questions**
- Which camera options are validated only after `--video-source=camera`?
- Does `--new-display` work without Android 10+?
- Does `--start-app` accept a package name or an activity?
- Is `-x` rejected before the device wait?

**Sources:** `scrcpy --help`; observed output.
**Output:** a display/camera/virtual-display matrix.

## Scenario 8 — input modes, OTG, validation, and cleanup

| # | Command | Purpose |
|---|---------|---------|
| 161 | `ADB=/usr/bin/adb timeout 5 scrcpy --keyboard=disabled` | No keyboard |
| 162 | `ADB=/usr/bin/adb timeout 5 scrcpy --keyboard=sdk` | SDK keyboard |
| 163 | `ADB=/usr/bin/adb timeout 5 scrcpy --keyboard=uhid` | UHID keyboard |
| 164 | `ADB=/usr/bin/adb timeout 5 scrcpy --keyboard=aoa` | AOA keyboard |
| 165 | `ADB=/usr/bin/adb timeout 5 scrcpy --keyboard=bogus 2>&1 \| head -2` | Invalid keyboard |
| 166 | `ADB=/usr/bin/adb timeout 5 scrcpy --mouse=disabled` | No mouse |
| 167 | `ADB=/usr/bin/adb timeout 5 scrcpy --mouse=sdk` | SDK mouse |
| 168 | `ADB=/usr/bin/adb timeout 5 scrcpy --mouse=uhid` | UHID mouse |
| 169 | `ADB=/usr/bin/adb timeout 5 scrcpy --mouse=aoa` | AOA mouse |
| 170 | `ADB=/usr/bin/adb timeout 5 scrcpy --mouse=bogus 2>&1 \| head -2` | Invalid mouse |
| 171 | `ADB=/usr/bin/adb timeout 5 scrcpy --gamepad=disabled` | No gamepad |
| 172 | `ADB=/usr/bin/adb timeout 5 scrcpy --gamepad=aoa` | AOA gamepad |
| 173 | `ADB=/usr/bin/adb timeout 5 scrcpy --gamepad=bogus 2>&1 \| head -2` | Invalid gamepad |
| 174 | `ADB=/usr/bin/adb timeout 5 scrcpy -K` | Shortcut for uhid/aoa keyboard |
| 175 | `ADB=/usr/bin/adb timeout 5 scrcpy -M` | Shortcut for uhid/aoa mouse |
| 176 | `ADB=/usr/bin/adb timeout 5 scrcpy --mouse-bind=bhsn:++++` | Valid bindings |
| 177 | `ADB=/usr/bin/adb timeout 5 scrcpy --mouse-bind=zzzz 2>&1 \| head -2` | Invalid binding |
| 178 | `ADB=/usr/bin/adb timeout 5 scrcpy --shortcut-mod=lctrl,lsuper` | Shortcut modifiers |
| 179 | `ADB=/usr/bin/adb timeout 5 scrcpy --shortcut-mod=bogus 2>&1 \| head -2` | Invalid modifier |
| 180 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-mouse-hover` | No hover |
| 181 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-key-repeat` | No key repeat |
| 182 | `ADB=/usr/bin/adb timeout 5 scrcpy --prefer-text` | Text events |
| 183 | `ADB=/usr/bin/adb timeout 5 scrcpy --raw-key-events` | Raw key events |
| 184 | `ADB=/usr/bin/adb timeout 5 scrcpy --legacy-paste` | Legacy paste |
| 185 | `ADB=/usr/bin/adb timeout 5 scrcpy --no-clipboard-autosync` | No clipboard sync |
| 186 | `ADB=/usr/bin/adb timeout 5 scrcpy --display-ime-policy=local` | IME policy |
| 187 | `ADB=/usr/bin/adb timeout 5 scrcpy --display-ime-policy=bogus 2>&1 \| head -2` | Invalid IME policy |
| 188 | `ADB=/usr/bin/adb timeout 5 scrcpy --pause-on-exit=if-error` | Pause mode |
| 189 | `ADB=/usr/bin/adb timeout 5 scrcpy --pause-on-exit=bogus 2>&1 \| head -2` | Invalid pause mode |
| 190 | `ADB=/usr/bin/adb timeout 5 scrcpy -V debug` | Verbosity |
| 191 | `ADB=/usr/bin/adb timeout 5 scrcpy -V bogus 2>&1 \| head -2` | Invalid verbosity |
| 192 | `ADB=/usr/bin/adb timeout 5 scrcpy --nonsense 2>&1 \| head -2` | Unknown option |
| 193 | `ADB=/usr/bin/adb timeout 5 scrcpy --nonsense >/dev/null 2>&1; echo "badopt=$?"` | Exit code |
| 194 | `timeout 5 scrcpy --otg 2>&1 \| head -6` | OTG enumerates USB hosts |
| 195 | `timeout 5 scrcpy --otg --keyboard=disabled --mouse=disabled 2>&1 \| head -2` | All inputs disabled |
| 196 | `ADB=/usr/bin/adb timeout 5 scrcpy --kill-adb-on-close` | Kill adb on exit |
| 197 | `ADB=/usr/bin/adb timeout 5 scrcpy --force-adb-forward` | Force forward |
| 198 | `ADB=/usr/bin/adb timeout 5 scrcpy --tunnel-host=127.0.0.1 --tunnel-port=27183 2>&1 \| head -2` | Tunnel override |
| 199 | `ADB=/usr/bin/adb timeout 5 scrcpy -p 27183:27199` | Port range |
| 200 | `ADB=/usr/bin/adb timeout 5 scrcpy -p bogus 2>&1 \| head -2` | Invalid port |
| 201 | `ADB=/usr/bin/adb timeout 5 scrcpy --push-target=/sdcard/Download/evidence` | Push target |
| 202 | `ADB=/usr/bin/adb timeout 5 scrcpy --screen-off-timeout=30` | Screen-off timeout |
| 203 | `rm -rf ~/scrcpy-demo` | Remove the demo tree |
| 204 | `test ! -e ~/scrcpy-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which input mode needs device-side keyboard layout configuration?
- Does `-K` choose `uhid` or `aoa` depending on `--otg`?
- Does `--otg` really work without adb?
- Which errors are parse-time and which need a device?

**Sources:** `scrcpy --help`; `man scrcpy`; observed output.
**Output:** an input-mode matrix, a validation table, and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `--version`, `-v`, `--help`, `man scrcpy`, binary and library provenance | 1 |
| `ADB`, `ANDROID_SERIAL`, `-s`, `-d`, `-e`, `--tcpip`, `--list-*` | 2 |
| `-n`, `--no-video`, `--no-audio`, `--no-playback`, `--no-window`, `--no-cleanup`, `-S`, `-w`, `-t`, `--keep-active` | 3 |
| `--video-codec`, `--video-encoder`, `-b`, `-m`, `--max-fps`, `--min-size-alignment`, `--video-buffer`, `--angle`, `--crop`, `--ignore-video-encoder-constraints` | 4 |
| `--audio-codec`, `--audio-source`, `--audio-bit-rate`, `--audio-buffer`, `--audio-output-buffer`, `--audio-dup`, `--require-audio` | 5 |
| `-r`, `--record-format`, `--record-orientation`, `--time-limit`, `--v4l2-sink`, `--v4l2-buffer`, `--print-fps` | 6 |
| window options, `--render-fit`, `--display-id`, `--new-display`, `-x`, `--start-app`, camera options | 7 |
| `--keyboard`, `--mouse`, `--gamepad`, `-K`, `-M`, `--mouse-bind`, `--shortcut-mod`, `--otg`, `--pause-on-exit`, `-V`, `-p`, `--tunnel-*` | 8 |

Full text: [`help.txt`](help.txt); authoritative description in `man scrcpy`.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, manual, binary, dependencies | 18 |
| 2 | adb resolution and device selection | 17 |
| 3 | Mirroring and control modes | 20 |
| 4 | Video pipeline and codecs | 23 |
| 5 | Audio pipeline and sources | 24 |
| 6 | Recording and external output | 19 |
| 7 | Window, display, camera, virtual displays | 39 |
| 8 | Input modes, OTG, validation, and cleanup | 44 |
| **Total** | | **204** |

## Legal and safety

`scrcpy` can read the screen, capture audio and camera, inject input, install
APKs by drag-and-drop, and record everything it sees. Only mirror devices you
own or are explicitly authorised to test. Recordings may contain personal data,
credentials, or notification content — store them as sensitive evidence and
delete them when the engagement ends. `--no-cleanup` leaves the server jar on
the device; remove it deliberately. OTG mode bypasses adb but still controls
the device, so treat it as an interactive session. Do not use `--tcpip` on an
untrusted network without a tunnel.
