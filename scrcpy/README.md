# scrcpy — Android screen mirroring and control

- **Version:** 4.1
- **Binary:** `/usr/local/bin/scrcpy` (locally built; not a dpkg package)
- **Help snapshot:** [`help.txt`](help.txt) — `scrcpy --help`
- **Manual:** `man scrcpy`
- **Upstream:** https://github.com/Genymobile/scrcpy
- **Licence:** Apache-2.0

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`scrcpy` mirrors an Android display and audio to the desktop and injects
keyboard, mouse, and gamepad input back into the device. It needs only adb and
USB debugging — no root, no app installed on the device. Video is encoded by
the device's MediaCodec, streamed over an adb tunnel, and decoded on the host.

For mobile assessments it is the interactive companion to `adb`: watch the app
while you drive it, record proof-of-concept video, drag-and-drop an APK, or use
OTG mode to test a device that has USB debugging disabled.

## How it connects

```text
scrcpy ──spawns──▶ adb server ──USB/TCP──▶ adbd
   │                                        │
   └── pushes scrcpy-server.jar ────────────┘
       opens a local port, then tunnels the
       video/audio/control sockets through adb
```

- A `scrcpy-server` jar is pushed to `/data/local/tmp` and started with
  `app_process`.
- The host listens on `27183:27199` by default (`-p`).
- On exit, the server jar is removed and device state (show touches, stay
  awake, power mode) is restored unless `--no-cleanup` is given.

## Device selection

| Option | Meaning |
|--------|---------|
| `-s SERIAL` | Select by serial; required when several devices are attached |
| `-d` | Select the single USB device |
| `-e` | Select the single TCP/IP device |
| `--tcpip[=[+]IP[:PORT]]` | Enable and connect over TCP/IP (default port 5555) |
| `--serial` | Long form of `-s` |
| `$ANDROID_SERIAL` | Default serial when no selector is given |

`--tcpip` without an address discovers the current device's IP, enables TCP
mode, then reconnects. Prefix the address with `+` to force a reconnection.

## Display and video

| Option | Meaning |
|--------|---------|
| `--video-source=display\|camera` | Mirror the display or a camera |
| `--video-codec=h264\|h265\|av1\|vp8\|vp9` | Codec (default h264) |
| `--video-encoder=NAME` | Force a MediaCodec encoder |
| `--video-codec-options=K[:T]=V[,...]` | Encoder options (`int`, `long`, `float`, `string`) |
| `-b, --video-bit-rate=RATE` | Bit rate, e.g. `8M` (default 8M) |
| `-m, --max-size=N` | Cap width and height (default 0 = unlimited) |
| `--max-fps=N` | Frame-rate cap (Android 10+) |
| `--min-size-alignment=N` | Power-of-2 size alignment (1, 2, 4, 8, 16) |
| `--ignore-video-encoder-constraints` | Ignore reported encoder limits |
| `--no-video` | Disable video forwarding |
| `--no-video-playback` | Decode but do not display |
| `--video-buffer=MS` | Playback buffering delay |
| `--no-downsize-on-error` | Do not retry at lower resolution on codec error |
| `--no-mipmaps` | Disable mipmap generation |
| `--angle=DEGREES` | Rotate content clockwise |
| `--crop=WIDTH:HEIGHT:X:Y` | Crop the video |
| `--display-id=ID` | Mirror a specific display |
| `--display-ime-policy=local\|fallback\|hide` | IME handling on virtual displays |

## Audio

| Option | Meaning |
|--------|---------|
| `--audio-source=output\|mic\|playback\|mic-*\|voice-call*` | Audio capture source |
| `--audio-codec=opus\|aac\|flac\|raw` | Codec (default opus) |
| `--audio-encoder=NAME` | Force a MediaCodec encoder |
| `--audio-codec-options=K[:T]=V[,...]` | Encoder options |
| `--audio-bit-rate=RATE` | Bit rate (default 128K) |
| `--audio-buffer=MS` | Buffering delay (default 50) |
| `--audio-output-buffer=MS` | Output buffering delay |
| `--audio-dup` | Duplicate audio; switches source to `playback` |
| `--no-audio` | Disable audio forwarding |
| `--no-audio-playback` | Do not play audio on the host |
| `--require-audio` | Fail if audio was requested but does not work |

Android 11+ is required for audio forwarding. `--audio-dup` is for devices
that cannot capture and play simultaneously.

## Input

| Option | Meaning |
|--------|---------|
| `-n, --no-control` | Read-only mirroring |
| `--keyboard=disabled\|sdk\|uhid\|aoa` | Keyboard injection mode |
| `--mouse=disabled\|sdk\|uhid\|aoa` | Mouse injection mode |
| `--gamepad=disabled\|aoa` | Gamepad injection mode |
| `-K` | `--keyboard=uhid` (or `aoa` with `--otg`) |
| `-M` | `--mouse=uhid` (or `aoa` with `--otg`) |
| `--mouse-bind=XXXX[:XXXX]` | Secondary-click bindings |
| `--no-mouse-hover` | Do not forward hover motion |
| `--no-key-repeat` | Do not forward key repeat |
| `--prefer-text` | Send alpha/space as text events |
| `--raw-key-events` | Send all keys as key events |
| `--legacy-paste` | Paste via key events instead of clipboard API |
| `--no-clipboard-autosync` | Disable clipboard synchronisation |
| `--shortcut-mod=KEY[,...]` | Shortcut modifiers (`lctrl`, `rctrl`, `lalt`, `ralt`, `lsuper`, `rsuper`) |

`--mouse-bind` takes one or two 4-character sequences for right, middle, 4th,
and 5th click. Characters: `+` forward, `-` ignore, `b` back, `h` home,
`s` app-switch, `n` notification panel. Default is `bhsn:++++` for SDK and
`++++:bhsn` for AOA/UHID.

## Recording and output

| Option | Meaning |
|--------|---------|
| `-r, --record=FILE` | Record to file; format from extension or `--record-format` |
| `--record-format=mp4\|mkv\|m4a\|mka\|opus\|aac\|flac\|wav` | Force container |
| `--record-orientation=0\|90\|180\|270\|flip*` | Record rotation |
| `--no-playback` | Disable host video and audio playback |
| `--v4l2-sink=/dev/videoN` | Output to a v4l2loopback device (Linux) |
| `--v4l2-buffer=MS` | V4L2 buffering delay |
| `--time-limit=SECONDS` | Stop after a fixed time |
| `--print-fps` | Print frames per second to the log |

Audio-only containers (`m4a`, `mka`, `opus`, `aac`, `flac`, `wav`) reject a
video stream. Combine them with `--no-video`.

## Window and device state

| Option | Meaning |
|--------|---------|
| `--window-title=TEXT` | Window title |
| `--window-x`, `--window-y` | Initial position (`auto` or pixels) |
| `--window-width`, `--window-height` | Initial size (0 = automatic) |
| `--window-borderless` | No decorations |
| `--always-on-top` | Keep above other windows |
| `-f, --fullscreen` | Start fullscreen |
| `--render-fit=letterbox\|stretched\|unscaled` | Fit mode |
| `--render-driver=NAME` | SDL render driver hint |
| `--no-window` | No window (implies `--no-video-playback`) |
| `--no-window-aspect-ratio-lock` | Allow free resize |
| `-x, --flex-display` | Resize a `--new-display` virtual display with the window |
| `--disable-screensaver` | Keep the host screensaver away |
| `-S, --turn-screen-off` | Turn the device screen off at start |
| `-w, --stay-awake` | Keep the device awake while plugged in |
| `-t, --show-touches` | Show physical touches |
| `--keep-active` | Simulate activity to keep the screen on |
| `--power-off-on-close` | Turn the device screen off on exit |
| `--no-power-on` | Do not power the device on at start |
| `--screen-off-timeout=SECONDS` | Screen-off timeout while running |
| `--no-terminal-title` | Do not change the terminal title |

## Camera

Camera mirroring requires Android 12+ and `--video-source=camera`.

| Option | Meaning |
|--------|---------|
| `--camera-id=ID` | Select a camera by id |
| `--camera-facing=front\|back\|external` | Select by facing |
| `--camera-size=WxH` | Capture size |
| `--camera-ar=W:H` | Aspect ratio (mutually exclusive with `--camera-size`) |
| `--camera-fps=N` | Capture frame rate |
| `--camera-high-speed` | High-speed capture; requires explicit `--camera-fps` |
| `--camera-zoom=FACTOR` | Zoom |
| `--camera-torch` | Turn the torch on |
| `--list-cameras` | List cameras |
| `--list-camera-sizes` | List valid capture sizes |

## Virtual displays and apps

| Option | Meaning |
|--------|---------|
| `--new-display[=[WxH][/DPI]]` | Create a virtual display |
| `--start-app=NAME` | Start a package; `?` prefix = prefix match, `+` = force-stop first |
| `--list-apps` | List installed apps |
| `--list-displays` | List displays |
| `--list-encoders` | List video and audio encoders |
| `--no-vd-destroy-content` | Move apps to the main display instead of destroying them |
| `--no-vd-system-decorations` | Disable virtual-display decorations |

## OTG mode

`--otg` simulates a physical keyboard and mouse over the USB OTG protocol. It
does not use adb, so it works when USB debugging is off, but it requires USB
and disables mirroring. At least one input must stay enabled.

## Shortcuts

`MOD` is left-Alt or left-Super by default; change it with `--shortcut-mod`.

| Shortcut | Action |
|----------|--------|
| `MOD+q` | Quit |
| `MOD+f`, `F11` | Fullscreen |
| `MOD+Left` / `MOD+Right` | Rotate display |
| `MOD+Shift+Left` / `Right` | Flip horizontally |
| `MOD+Shift+Up` / `Down` | Flip vertically |
| `MOD+z` / `MOD+Shift+z` | Pause / unpause display |
| `MOD+Shift+r` | Reset capture/encoding |
| `MOD+g` | Resize to 1:1 |
| `MOD+w` | Remove black borders |
| `MOD+h` / middle-click | HOME |
| `MOD+b` / `MOD+Backspace` / right-click | BACK |
| `MOD+s` / 4th-click | APP_SWITCH |
| `MOD+m` | MENU |
| `MOD+Up` / `MOD+Down` | Volume up / down |
| `MOD+p` | Power |
| `MOD+o` / `MOD+Shift+o` | Screen off / on |
| `MOD+r` | Rotate device |
| `MOD+n` / `MOD+Shift+n` | Expand / collapse notifications |
| `MOD+c` / `MOD+x` / `MOD+v` | Copy / cut / paste |
| `MOD+Shift+v` | Paste as key events |
| `MOD+k` | Open physical-keyboard settings |
| `MOD+i` | Toggle FPS counter |
| `Ctrl+click-drag` | Pinch-zoom and rotate |
| `Shift+click-drag` | Tilt vertically |
| `Ctrl+Shift+click-drag` | Tilt horizontally |
| Drag-and-drop APK | Install the APK |
| Drag-and-drop file | Push to `--push-target` |

## Environment variables

| Variable | Meaning |
|----------|---------|
| `ADB` | Path to the adb executable |
| `ANDROID_SERIAL` | Default device serial |
| `SCRCPY_ICON_DIR` | Icon directory |
| `SCRCPY_SERVER_PATH` | Path to the server binary |

**This build embeds `/usr/local/bin/adb`, which does not exist on this host.**
Set `ADB=/usr/bin/adb` or every command fails with
`Failed to execute: [/usr/local/bin/adb]`.

## Recipes

**1. Mirror a USB device.**

```bash
ADB=/usr/bin/adb scrcpy -s SERIAL
```

**2. Read-only mirroring for a demo.**

```bash
ADB=/usr/bin/adb scrcpy --no-control
```

**3. Record a proof-of-concept clip with audio.**

```bash
ADB=/usr/bin/adb scrcpy -r poc.mp4 --audio-source=mic
```

**4. Mirror and record without opening a window.**

```bash
ADB=/usr/bin/adb scrcpy --no-window -r capture.mkv
```

**5. List apps, displays, cameras, and encoders.**

```bash
ADB=/usr/bin/adb scrcpy --list-apps
ADB=/usr/bin/adb scrcpy --list-displays
ADB=/usr/bin/adb scrcpy --list-cameras
ADB=/usr/bin/adb scrcpy --list-encoders
```

**6. Start an app on a fresh virtual display.**

```bash
ADB=/usr/bin/adb scrcpy --new-display=1920x1080/420 --start-app=+?firefox
```

**7. Push a file by drag-and-drop target.**

```bash
ADB=/usr/bin/adb scrcpy --push-target=/sdcard/Download/evidence
```

**8. Turn the device screen off while mirroring.**

```bash
ADB=/usr/bin/adb scrcpy -S --stay-awake
```

**9. Stream to a v4l2loopback device.**

```bash
sudo modprobe v4l2loopback
ADB=/usr/bin/adb scrcpy --v4l2-sink=/dev/video10 --no-playback
```

**10. Use OTG when USB debugging is disabled.**

```bash
scrcpy --otg -s SERIAL
```

**11. Connect over TCP/IP.**

```bash
ADB=/usr/bin/adb scrcpy --tcpip=192.0.2.10:5555
```

**12. Keep the server and device state for repeat runs.**

```bash
ADB=/usr/bin/adb scrcpy --no-cleanup --no-terminal-title
```

## Exit status

| Code | Meaning |
|------|---------|
| 0 | Normal termination |
| 1 | Start failure (bad option, no device, no display, codec error) |
| 2 | Device disconnected while running |

The help text documents these three values. In practice every pre-connection
error observed here returned 1, including invalid option values.

## Quirks and limits

- **The embedded adb path is wrong on this host.** Set `ADB=/usr/bin/adb`.
- **`--list-*` needs a device.** With none attached they exit 1 with
  `Could not find any ADB device`, not an empty list.
- **Option values are validated before the device wait.** Bad codecs,
  orientations, and bindings fail immediately; valid ones then fail on the
  missing device.
- **`--camera-size` and `--camera-ar` are mutually exclusive.**
- **`--camera-high-speed` needs an explicit `--camera-fps`.**
- **`--flex-display` only works with `--new-display`.**
- **`--v4l2-buffer` without `--v4l2-sink` is an error.**
- **Audio-only containers reject video.** Use `--no-video` with `m4a`, `opus`,
  and friends.
- **`--record` needs a known extension.** `.bin` fails unless
  `--record-format` is set.
- **`--audio-dup` changes the source.** It switches to `playback` and warns.
- **`--otg` does not use adb.** With no Android device it may list USB host
  controllers and ask for `-s`.
- **`--tcpip=ADDR` can block.** An unreachable address waits for a connection.
- **`--no-playback` can disable streams.** With no recording and no V4L2 sink,
  video and audio are turned off entirely.
- **Virtual displays need Android 10+.** `--new-display` and `--flex-display`
  depend on device support.
- **The server jar is pushed to `/data/local/tmp`.** `--no-cleanup` leaves it
  there; remove it manually.
- **This is a locally built binary.** `dpkg -S` does not know it, so package
  integrity checks will not cover it.

## Related tools

- [`adb`](../adb/) — transport, shell, install, and port forwarding.
- [`fastboot`](../fastboot/) — bootloader flashing and unlocking.
- [`apktool`](../apktool/) — decode an APK pulled from the device.
- [`ghidra`](../ghidra/) — analyse native libraries.
- [`exiftool`](../exiftool/) — inspect screenshots and recorded video.
- `ffmpeg` — transcode or trim a recording (not installed here).
- `frida`, `objection`, `jadx` — dynamic and static app analysis (not
  installed here).
