<p align="center">
  <img alt="eDEX-UI" src="media/logo.png">
  <br><br>
  <strong>eDEX-UI — native Apple Silicon (arm64) build</strong>
  <br>
  <em>A maintained fork of the archived <a href="https://github.com/GitSquared/edex-ui">GitSquared/eDEX-UI</a>, rebuilt to run natively on Apple Silicon Macs.</em>
</p>

---

## Why this fork exists

[eDEX-UI](https://github.com/GitSquared/edex-ui) is a fullscreen, sci-fi terminal emulator + system monitor. The original project was **archived in October 2021** and only ever shipped an **x86-only** macOS build (Electron 12 / Chromium 89).

On modern Apple Silicon (M-series) Macs that build:

- runs under **Rosetta 2** translation, and
- its old Chromium **cannot use the GPU** (Metal) on a recent macOS, so it falls back to **100% software rendering**.

The result on an M-series Mac is unusable: ~6.5 CPU cores pinned at idle, severe stutter/freeze on keyboard input, and a meltdown of the whole system. No native arm64 build of eDEX-UI existed anywhere — the upstream arm64 CI was disabled right before the project was archived.

**This fork is a native arm64 build with the app modernised enough to run well on Apple Silicon.**

### Measured difference (Apple M5, macOS 26)

| Build | Idle CPU | Input |
|---|---|---|
| Upstream x86 (Rosetta, Electron 12) | ~653% (6.5 cores) | freezes on key-hold |
| **This fork (native arm64, Electron 31)** | **~30% (0.3 cores)** | smooth |

## What changed vs upstream

- **Electron 12 → 31.7.7** — modern Chromium uses **Metal/GPU** on Apple Silicon, which fixes both the idle-CPU meltdown and the input freeze (the original problem was software rasterisation).
- **`@electron/remote`** wired up properly — Electron 14+ removed the built-in `remote` module; renderer code migrated from `electron.remote.*` to the `@electron/remote` module + `remote.enable()` on the window.
- **node-pty 0.10 → 1.1** — rebuilt for the Electron 31 Node ABI (arm64).
- **xterm 4.14 → 5.3** + addons.
- **Terminal pane no longer renders white/inverted** — augmented-ui 1.1.2's `::after` border layer mis-clips on Chromium 126 and floods the terminal with the light border colour; disabled it for `#main_shell` and restored a thin CSS border.
- **Network status no longer stuck OFFLINE behind a VPN** — the connectivity check no longer force-binds the ping socket to a specific interface IP (which fails when the default route is a VPN `utun` tunnel with no MAC); it now lets the OS route normally.
- Minor input-path tidy-ups (skip the on-screen-keyboard repaint on key auto-repeat; the ligatures addon is not loaded).

## Download

A prebuilt, **native arm64** `.dmg` is attached to the [latest release](../../releases/latest).

The build is **unsigned** (no Apple Developer certificate), so on first launch macOS Gatekeeper will block it. To open it:

```sh
xattr -cr "/Applications/eDEX-UI.app"
```

…or right-click the app → **Open** → **Open**.

## Build from source

Tested on an Apple Silicon Mac, macOS 26. The non-obvious bits are the toolchain versions — the project's build tooling is from 2021 and needs era-appropriate Node + a Python that still ships `distutils`.

```sh
# Tooling (via Homebrew + fnm)
brew install fnm python@3.11
fnm install 20.18.1
export PYTHON="$(brew --prefix python@3.11)/libexec/bin/python"   # node-gyp needs distutils (gone in Python 3.12+)

git clone --recurse-submodules https://github.com/pydantick/edex-ui-arm64.git
cd edex-ui-arm64

# Install deps under Node 20
fnm exec --using=20.18.1 -- npm install
fnm exec --using=20.18.1 -- bash -c 'cd src && npm install'

# Rebuild node-pty for the Electron ABI (arm64)
fnm exec --using=20.18.1 -- npx @electron/rebuild -v 31.7.7 -a arm64 -m ./src -w node-pty -f

# Run
fnm exec --using=20.18.1 -- npm start

# Package a native arm64 .app
fnm exec --using=20.18.1 -- npx @electron/packager ./src "eDEX-UI" \
  --platform=darwin --arch=arm64 --electron-version=31.7.7 \
  --out=./dist --overwrite --prune=false --icon=./media/icon.icns
```

> Submodules use SSH URLs; if you don't have SSH keys set up, prefix git with
> `-c url."https://github.com/".insteadOf="git@github.com:"`.

## Credits & license

All credit for eDEX-UI goes to **[Gabriel "Squared" SAILLARD](https://github.com/GitSquared)** and the original [eDEX-UI contributors](https://github.com/GitSquared/edex-ui/graphs/contributors). This fork only modernises the build for Apple Silicon.

Licensed under **GPL-3.0**, same as upstream — see [LICENSE](LICENSE).
