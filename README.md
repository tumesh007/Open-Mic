# OpenMic Bridge — Web Variant

Phone acts as a mic input for Ubuntu with **no Android app** — just a
browser tab. Trade-off vs the native client: the browser tab must stay
in the foreground with the screen on. If you background the tab or lock
the phone, streaming will likely stop or glitch (behavior varies by
Android/Chrome version — this is a real limitation, not a bug to fix).

## Requirements

- Ubuntu with PipeWire (default on 24.04+)
- Python 3 with `aiohttp`: `pip install aiohttp --break-system-packages`
- Phone and PC on the same WiFi network
- Any modern mobile browser (Chrome recommended)

## The one gotcha: secure context

Browsers block microphone access (`getUserMedia`) on plain `http://`
unless the origin is `localhost`. Your PC's LAN IP doesn't qualify, so
you need ONE of the following, done once:

### Option A — Chrome flag (recommended, no certificates needed)

1. On your **phone**, open Chrome and go to:
   `chrome://flags/#unsafely-treat-insecure-origin-as-secure`
2. Set it to **Enabled**.
3. In the text box that appears, add your PC's address, e.g.:
   `http://192.168.1.100:8080`
4. Tap **Relaunch** at the bottom.

This only affects that one address on that one phone/browser — it does
not weaken security for any other site.

**Note:** the flag matches the exact address you type, including
whether it's an IP or hostname. If you later switch to the `.local`
hostname (see "Making it permanent" below), add that as a *second*
entry in the same flag rather than replacing the IP one — otherwise
the hostname version won't have mic permission even though the IP
version did.

### Option B — self-signed HTTPS certificate

If you'd rather not touch browser flags (e.g. you want other people's
phones to connect too), generate a self-signed cert and run the server
with SSL. You'll get a one-time "connection not private" warning to
click through on each phone. Ask if you want this variant of
`server.py` — it's a small change (wrap the aiohttp app with an
`ssl.SSLContext`).

## Running it

```bash
cd openmic-bridge-web
python3 server.py --port 8080
```

The script will:
1. Create the PipeWire virtual mic (`openmic_bridge`), same as the
   native-client version — if you already ran `receiver.py` before,
   this reuses the same sink/source names, so don't run both at once.
2. Print the URL to open on your phone.

On your phone: open `http://<this-PC's-LAN-IP>:8080`, tap **Start
streaming**, grant mic permission. In your desktop app (Zoom, Discord,
OBS, browser), select **OpenMicBridge** as the microphone input.

## Making it permanent (auto-start + no IP typing)

### 1. Auto-start the server on boot

```bash
cd openmic-bridge-web
./install_autostart.sh
```

This checks for and installs everything needed — audio (`pactl`/`paplay`
via `apt`, `aiohttp` via `pip`/`apt`) and video (`ffmpeg`, `v4l2loopback`).
For video specifically, it:
- Tries Ubuntu's packaged `v4l2loopback-dkms` first.
- Falls back to building the current upstream source if the packaged
  version fails on your kernel (this happened on a newer kernel during
  development — the packaged source predated a kernel API change).
- Writes `/etc/modprobe.d/` and `/etc/modules-load.d/` config so the
  virtual camera exists automatically on every boot, with no `sudo`
  needed at runtime (required — systemd user services can't prompt for
  a password, so without this, video would silently fail on every
  reboot even though it worked when tested manually).
- Adds your user to the `video` group if needed (tells you to log out
  and back in — group membership doesn't apply to your current
  session).

If video setup fails for any reason, the script continues and installs
audio-only — nothing about the working mic path is put at risk by a
video failure. It then installs a systemd **user** service, copies the
project to `~/openmic-bridge-web`, enables `linger` so it starts at
boot even before you log in, and verifies the service actually started.
Safe to re-run on a machine you've already set up, or a brand new one.

Useful commands afterward:
```bash
systemctl --user status openmic-bridge-web.service     # is it running?
journalctl --user -u openmic-bridge-web.service -f      # live logs
systemctl --user stop openmic-bridge-web.service        # stop for now
systemctl --user disable openmic-bridge-web.service     # stop autostarting
```

### 2. Skip typing the IP — use your PC's hostname instead

Ubuntu ships `avahi-daemon` by default, which broadcasts your PC's
hostname over mDNS. Instead of looking up the LAN IP every time, try:

```
http://<your-pc-hostname>.local:8080
```

(Find your hostname with `hostname` in a terminal.) If `.local`
doesn't resolve on your phone, your router/WiFi network is blocking
mDNS — fall back to the numeric IP, or set a static IP for the PC in
your router settings so at least the IP itself never changes.

### 3. The phone side — one tap instead of typing a URL

Nothing runs automatically here; a website can't be pushed open on
your phone. But once you've done the Chrome flag step and visited the
page once, use Chrome's menu → **"Add to Home Screen"**. That gives
you an icon that opens straight to the mic page — closest you'll get
to "just tap and go" without building a native app.

## Optional: noise suppression (background noise, TV, etc.)

```bash
python3 server.py --port 8080 --denoise
```
or combine with video:
```bash
python3 server.py --port 8080 --video --denoise
```

Runs incoming audio through RNNoise (via ffmpeg's `arnndn` filter)
before it reaches the virtual mic — trained to suppress steady
background noise (TV, fans, traffic) while preserving speech. The
model (`models/std.rnnn`, ~300KB, BSD-licensed) is bundled in this
repo, nothing to download separately.

**Trade-offs, real ones:**
- Adds real CPU cost — this is a neural-network filter running on
  every audio frame, not a cheap EQ tweak. Fine on any reasonably
  modern desktop; worth checking CPU usage (`top`) on an older or
  low-power PC before leaving it on permanently.
- Adds a small amount of latency (the filter needs the audio
  resampled to 48kHz and back, plus its own processing).
- It suppresses steady background noise well; it will not remove
  another person's voice, and very loud or sudden sounds (a door
  slam, a dog barking) may still get through.

Verified independently before shipping: the exact filter chain used
here (`aresample=48000,arnndn=m=...,aresample=<rate>`) was tested
directly against a synthetic noisy signal at 16kHz mono — the same
format this project actually uses — and produced clean output at
exit code 0. What hasn't been tested end-to-end is the `-f pulse`
output stage against a real PipeWire session — if `--denoise` fails
on first use, that's the first thing to check
(`journalctl --user -u openmic-bridge-web.service -f` while
connecting, same as always).

## Finding your PC's LAN IP

```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

## How it works

- `public/index.html` — UI, requests mic access, opens a WebSocket to
  `/ws`, sends a JSON `{type: "start", sampleRate, channels}` handshake
  then streams raw 16-bit PCM as binary WebSocket frames.
- `public/audio-processor.js` — an `AudioWorkletProcessor` running on
  the browser's dedicated audio thread; converts Float32 samples from
  the mic to Int16 PCM and hands them to the main thread.
- `server.py` — aiohttp server. Serves the page, and on the `/ws`
  route spawns `paplay` writing into the `openmic_sink` PipeWire sink
  at whatever sample rate the phone's browser reports (no fixed rate
  — avoids downsampling artifacts).

No custom protocol header is needed here (unlike the UDP native-client
version) because WebSocket runs over TCP: ordered, reliable delivery,
so there's no packet loss/reordering to handle in v1.

## Video (webcam) support

Video support is powered by WebRTC for low latency and high quality. **Independent of the working audio path** — enabling this cannot break
your existing mic setup; it's a separate WebSocket route and a
separate opt-in button on the page.

### One-time setup

```bash
sudo apt install v4l2loopback-dkms ffmpeg
```

`v4l2loopback-dkms` rebuilds itself against your kernel on every kernel
update — this is the same driver-fragility trade-off flagged when this
idea first came up, unavoidable on Linux for virtual cameras (unlike
audio, which PipeWire handles in userspace).

### Running it

```bash
python3 server.py --port 8080 --video
```

Without `--video`, the server behaves exactly as before — nothing about
your current setup changes unless you explicitly add this flag.

On the phone page, a second section appears with its own "Start video"
button, independent of the audio "Start streaming" button — use both,
either, or neither.

### Status

Confirmed working end-to-end (v2). Known rough edges, still real:
- Video quality and framerate are adaptively managed by WebRTC.
- Resolution targets 720p HD and preserves aspect ratios correctly via PyAV frame conversion.
- Automatic reconnect logic relies on native WebRTC ICE restarts where supported.
- Requires the `video` group and a persistent v4l2loopback module load
  at boot — both handled by `install_autostart.sh`, but if you set this
  up manually instead, see that script for the exact steps.


## Desktop GUI

The installation script installs a system tray application and GUI that allows you to easily view connection status and control settings:
- **Denoise (RNNoise)**: Toggle AI noise suppression on the fly.
- **Video**: Enable or disable video track processing.
- **Loudspeaker Output**: Toggle whether the audio plays directly to your computer speakers (Loudspeaker) or to the virtual microphone (default, for use in Zoom/Discord).

You can launch it from your application menu as **OpenMic Bridge GUI**. It will automatically start minimized in your system tray on boot.

## Known limitations (be upfront with yourself about these)


- Screen must stay on; tab must stay foregrounded. No foreground-service
  equivalent exists for a website.
- No mDNS auto-discovery — you type the IP once, same as the native
  client's v1.
- No authentication — anyone on your WiFi who knows the port can
  connect. Fine for a home network; don't expose this port outside
  your LAN.

## Reconnect behavior

If the WebSocket drops mid-stream (WiFi blip, server restart, phone
briefly switching networks), the page detects the close event and
retries automatically with exponential backoff (1s, 2s, 4s... capped
at 10s). The mic and audio pipeline stay alive throughout — audio
frames are simply dropped while disconnected, and streaming resumes
once the socket reconnects, with no need to tap Start again. The
status line shows "Connection lost — reconnecting..." during this.
Tapping **Stop** manually always wins — it cancels any pending
reconnect attempt.

## Repo layout

```
openmic-bridge/
├── web/              working — browser-based client + Linux server
├── native/
│   ├── android/       parked — Kotlin foreground-service mic client
│   └── linux/         working standalone — UDP receiver + PipeWire setup
├── protocol/           wire format used by the native UDP client
├── LICENSE
└── README.md            (this file)
```

## Requirements

- Ubuntu 24.04+ (PipeWire) for the receiver/server side
- Phone and PC on the same local WiFi network
- Python 3 + `aiohttp` for the web variant server

## License

MIT — see `LICENSE`.

# Release v3.0.0

## Major Updates

*   **WebRTC Video Migration:** The experimental JPEG-over-WebSocket video transport has been entirely replaced with a robust WebRTC implementation. This provides adaptive bitrate, improved framerates, reduced latency, and native resolution negotiation, providing a vastly smoother video streaming experience from the phone to your Linux virtual camera.
*   **Android App Update:** The Android native app has been updated to integrate the new WebRTC client. The Android build has been bumped to support this functionality, delivering high-performance background audio and foreground video cleanly.
*   **Desktop GUI & System Tray:** We have introduced a brand new desktop GUI powered by Tkinter and PyStray. The `install_autostart.sh` script now seamlessly installs this GUI, setting it up to launch minimized to your system tray on boot. You can quickly view connection status, toggle the AI noise denoiser (RNNoise), and now toggle a Loudspeaker output mode without needing to use terminal commands!
*   **Loudspeaker Routing:** Added a user-requested feature to easily toggle audio playback. By default, audio routes to your virtual microphone (`openmic_sink`) for seamless use in Zoom, Discord, and OBS. If you toggle "Loudspeaker Output" in the new GUI, audio routes straight to your physical speakers/headphones so you can monitor or use the phone as a megaphone.
*   **Stability Fixes:** Corrected multiple background service execution issues and indentation edge-cases to ensure `systemd` user services execute cleanly without dropping `aiohttp` API routes (resolving the previous `405 Method Not Allowed` API errors).

## Installation / Update Instructions

To install or update to v3.0.0, pull the latest codebase and run the unified installer:

```bash
cd openmic-bridge-web
./install_autostart.sh
```

*(Note: The installer will now automatically install `python3-tk`, `python3-pil`, `python3-pystray`, `python3-aiortc`, and other dependencies, and may prompt for your `sudo` password to persist v4l2loopback and install packages.)*

After the installation is complete, the background service will automatically restart, and the new GUI application will be added to your system applications menu (as `OpenMic Bridge GUI`). You can find it in your system tray on your next desktop login, or launch it immediately from your app drawer!
