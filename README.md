# SpelunkrPS5

Watch [Spelunkr.tv](https://spelunkr.tv) on a jailbroken PlayStation 5, with controller support and an optional dashboard icon.

Spelunkr is a free, ad-free, always-on TV network of 30+ channels built from public Internet Archive video. This project packages a PS5 launcher around the community [Spelunkr guide client](https://github.com/buddytohelpu/spelunkr-guide): it opens the guide in the PS5 browser, adds DualSense controls, and includes an on-screen diagnostics panel for video problems.

> **Unofficial.** Not affiliated with Spelunkr.tv, the Internet Archive, or Sony. For use on your own console.

## Status

| Part | State |
|---|---|
| Built against the ps5-payload-sdk | Done |
| Embedded web server and controller layer | Tested on a PC (HTTP server, 22 controller checks in headless Chromium) |
| Browser launch on the console | Confirmed with the plain launcher; unconfirmed for this version |
| Controller input in the PS5 browser | Untested on hardware |
| Video playback on the PS5 browser | Unresolved, use the diagnostics panel to find the cause |
| Dashboard app | Untested on hardware |

## How it works

```
 ELF loader (port 9021)
        │  spelunkr.elf
        ▼
 launcher payload ──► serves the guide on 127.0.0.1:8765 (loopback only)
        │           └─ injects ps5-shim.js (controller + diagnostics)
        └──────────► opens the PS5 web browser at http://127.0.0.1:8765/?tv=1
```

- `spelunkr.elf` is a payload for a PS5 ELF loader. It embeds the guide's static files, serves them on loopback, and launches the system browser. Nothing is exposed to your LAN.
- Schedules come from Spelunkr's public feeds and video plays straight from the Internet Archive; nothing is proxied.
- The dashboard app is a tiny static `eboot` that sends the launcher to your ELF loader on `127.0.0.1:9021`, so the loader must be running first.

## Requirements

- A jailbroken PS5 running an ELF loader on port 9021 (for example [ps5-payload-elfldr](https://github.com/ps5-payload-dev/elfldr)).
- To build: [ps5-payload-sdk](https://github.com/ps5-payload-dev/sdk), `clang-18`, `lld-18`, `python3` (with Pillow only if you regenerate the icon), and a checkout of the guide client.

## Quick start

1. Download `spelunkr.elf` from [Releases](../../releases) (or build it, below).
2. Send it to your ELF loader:
   ```sh
   socat -t 99999999 - TCP:<ps5-ip>:9021 < spelunkr.elf
   ```
3. The browser opens the guide. Pick a channel and press Cross.

To open the real site instead of the local guide, put one line in `/data/spelunkr/url.txt` on the console (for example `https://spelunkr.tv`) and run the ELF again.

## Controls

| DualSense | Action |
|---|---|
| D-pad or left stick up/down | Previous / next channel |
| D-pad or left stick left/right | Previous / next program |
| Cross | Watch / return to guide |
| Circle | Back to guide |
| Square | Sound on / off |
| Triangle | Remote help |
| L1 / R1 | Picture: original / stretch |
| Options | Show / hide diagnostics panel |

The PS5 browser may not pass controller input to web pages at all. If the panel's bottom-right line says `gamepad: none visible to the browser`, use the on-screen OK / BACK buttons with the browser's own cursor.

## Video troubleshooting

When a video errors, a diagnostics panel opens by itself for 25 seconds. Press **Options** to open it any time. It shows:

- the media error code (1 aborted, 2 network, 3 decode, 4 not supported),
- network and ready state, current time, and the video URL,
- which H.264, AAC and HLS types this browser claims to support,
- the user agent and whether workers and `fetch` are available.

Please include the first six lines of the panel when opening a video issue.

## Dashboard app (optional)

`SPLK00001/` is a game-style folder (`eboot.bin`, `sce_sys/param.json`, `sce_sys/icon0.png`).

**With ShadowMount+:** place the folder where ShadowMount+ scans for game folders (see its documentation) and let it register the title. Start your ELF loader before opening the app.

**With the SDK install flow** (same steps as the SDK's `install_app` sample; needs an FTP server on the console at port 2121):

1. Upload `SPLK00001/eboot.bin` to `/system_ex/app/SPLK00001/`
2. Upload `sce_sys/param.json` and `sce_sys/icon0.png` to `/user/app/SPLK00001/sce_sys/`
3. Send `install-app.elf` to the loader
4. Upload `sce_sys/param.json.system` to `/system_ex/app/SPLK00001/sce_sys/param.json`

## Building

```sh
# 1. Install the SDK once
git clone https://github.com/ps5-payload-dev/sdk
cd sdk && sudo make DESTDIR=/opt/ps5-payload-sdk install
export PS5_PAYLOAD_SDK=/opt/ps5-payload-sdk

# 2. Get the guide client (not included in this repository)
git clone https://github.com/buddytohelpu/spelunkr-guide ../spelunkr-guide

# 3. Build
make WEB=../spelunkr-guide/web
```

Outputs land in `dist/`: `spelunkr.elf`, `SPLK00001/`, and `install-app.elf`.

To test the web server and controller layer on a normal machine:

```sh
python3 tools/embed.py ../spelunkr-guide/web shim/ps5-shim.js build/gen
gcc -DNATIVE_TEST -Ibuild/gen -o build/server-test src/launcher.c build/gen/assets.c -lpthread
./build/server-test 8765    # then open http://127.0.0.1:8765/?tv=1
```

## Repository layout

```
src/launcher.c     payload: embedded HTTP server + browser launch
src/eboot.c        dashboard app eboot: sends the launcher to the ELF loader
src/install.c      optional app installer payload (from the SDK sample)
shim/ps5-shim.js   DualSense mapping, focus styles, diagnostics panel
tools/embed.py     embeds the web files into the launcher
tools/make_fself.py, tools/eboot.x   from the SDK's install_app sample
title/             param.json, icon for the dashboard app
Makefile
```

## Licensing and distribution

- `src/eboot.c`, `src/install.c`, `tools/eboot.x` and `tools/make_fself.py` derive from the ps5-payload-sdk `install_app` sample (GPLv3+), so this repository is released under **GPL-3.0-or-later**.
- The guide client ([buddytohelpu/spelunkr-guide](https://github.com/buddytohelpu/spelunkr-guide)) has no license file. This repository therefore does **not** include its source, and you should not publish built `spelunkr.elf` or `eboot.bin` files, since they embed it. Build locally for your own console.

## Credits

- [Spelunkr.tv](https://spelunkr.tv): the original project, its broadcasts and scheduler.
- [buddytohelpu/spelunkr-guide](https://github.com/buddytohelpu/spelunkr-guide): the guide client this wraps.
- [ps5-payload-dev/sdk](https://github.com/ps5-payload-dev/sdk) by John Törnblom: SDK, install sample and signing tooling.
- [Internet Archive](https://archive.org): all video.
