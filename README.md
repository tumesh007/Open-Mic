# OpenMic Bridge

Use an Android phone as a microphone input for a Linux PC over local
WiFi — free, open source, no cloud, no paywall.

## Status

**Working:** `web/` — phone browser as the mic client, no APK required.
This is the recommended way to use the project today.

**Parked:** `native/android/` — a native Android client (Kotlin,
foreground service, `AudioRecord`) exists but is incomplete. Development
paused pending Android app dev experience. The protocol and Linux-side
receiver in `native/linux/` work standalone and were the basis for the
web variant's server.

**Not planned:** Windows support. Out of scope for this project — see
`native/` history if you want to pick it up; would need a virtual
audio driver or a wrapper around an existing one (e.g. VB-Cable), not
something built from scratch.

## Which variant to use

- **Just want it working today, phone screen on while you use it:**
  use `web/`. See `web/README.md`.
- **Want a real installed Android app, phone screen off, background
  streaming:** the pieces are in `native/android/` and `native/linux/`,
  but the Android side needs finishing — contributions welcome.

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
