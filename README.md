# FreeDisplay

**English** · [简体中文](README.zh-CN.md)

> **macOS 26 (Tahoe) compatible fork of [FreeDisplay](https://github.com/huberdf/FreeDisplay)** — free & open-source BetterDisplay alternative, rebuilt for Apple's new DCP display stack.

BetterDisplay is a great app, but its best features are locked behind a paid Pro license. FreeDisplay implements the most essential BetterDisplay features as a completely free, open-source macOS menu bar app.

This fork brings the upstream project up to date with **macOS 26 (Tahoe)**, where Apple's display subsystem has been substantially restructured around a new DCP (Display Co-Processor) driver model. Several upstream code paths silently broke under Tahoe; this fork ships the fixes.

[Download Latest Release](https://github.com/Akuwatoga/FreeDisplay/releases/latest) · [Report an Issue](https://github.com/Akuwatoga/FreeDisplay/issues) · [Upstream Project](https://github.com/huberdf/FreeDisplay)

---

## Why this fork?

Running upstream FreeDisplay on macOS 26 has several user-visible failures:

| Symptom on Tahoe | Root cause | Fix in this fork |
|---|---|---|
| Menu bar panel collapses to just the version label — displays/tools list invisible | SwiftUI `MenuBarExtra(.window)` + inner `ScrollView` intrinsic height regresses to 0 | Explicit `minHeight: 400` on the panel container |
| App crashes (`EXC_BREAKPOINT`) when dragging brightness slider | Swift 6 runtime isolation check trips because DDC write completion runs off the main actor | `DDCService.writeAsync` completion is now `@MainActor @Sendable`, hopped via `MainActor.assumeIsolated` |
| Slider for monitor A actually controls monitor B (or none) | `IODisplayConnect` tree removed in Tahoe; the old `IORegistry` parent-chain `vendor/product` matching always fails, falling back to a "sorted-index" guess that's 50/50 | New EDID-based matching — read I²C `0x50` from each `IOAVService`, parse vendor/product, match against `CGDisplayVendorNumber/ModelNumber` |
| Some monitors (e.g. LG ULTRAGEAR) silently drop to software gamma even when DDC works | Liveness probe used DDC/CI ping at `0x37/0x51`; many monitors stay silent until they receive a real VCP request | Probe via EDID read instead — every working display answers `0x50` |
| Brightness control sticks on "software" path after a single failed VCP read | Failed DDC *read* was incorrectly treated as evidence DDC isn't supported, marking the display unavailable forever | Read failure no longer pre-empts write capability |

See git history on the [`tahoe-compat`](https://github.com/Akuwatoga/FreeDisplay/tree/tahoe-compat) branch for the per-commit detail.

---

## What BetterDisplay Features Does This Replace?

| BetterDisplay Feature | FreeDisplay | Notes |
|----------------------|:-----------:|-------|
| DDC Brightness & Contrast | ✅ | Hardware control via IOKit I2C (Intel) / IOAVService (Apple Silicon) — EDID-matched on Tahoe |
| Software Brightness (Gamma) | ✅ | Per-display gamma table control with smooth transitions |
| Keyboard Brightness Keys for External Displays | ✅ | Intercepts brightness keys when cursor is on external display, shows native macOS OSD |
| Auto Brightness Sync | ✅ | Syncs external display brightness with built-in display changes |
| HiDPI Virtual Displays | ✅ | Creates HiDPI dummy displays via CGVirtualDisplay private API |
| Display Arrangement | ✅ | Position displays (external above built-in, etc.) |
| Resolution & HiDPI Switching | ✅ | Browse and switch all available display modes including HiDPI |
| ICC Color Profile Management | ✅ | Switch color profiles per display via ColorSync |
| Image Adjustment (Gamma/Temperature) | ✅ | Software contrast, color temperature, RGB channels, invert |
| Display Presets | ✅ | Save & restore full display configurations with one click |
| Virtual Display (Dummy) | ✅ | Create headless virtual displays |
| Notch Management | ✅ | Hide the MacBook notch with a black overlay |
| Launch at Login | ✅ | Via SMAppService |

### Not Included (intentionally)

- Screen streaming / PiP — rarely used, adds complexity
- EDID override — requires SIP disabled
- XDR/HDR extra brightness — requires specific hardware

### Known Tahoe limitations (system, not app)

- **High-bandwidth DCP channels reject DDC writes.** Some monitors at 5K @ 144Hz (or similar DSC-compressed modes) return `kIOReturn 0xe0114000` to any `IOAVServiceWriteI2C` call. The app silently falls back to software gamma. This appears to be a system-level restriction on the new DCP stack and affects every user-space tool, including MonitorControl and BetterDisplay. Try lowering refresh rate to 60Hz to confirm.

---

## Installation

### Option 1: Download DMG

1. Download `FreeDisplay.dmg` from [Releases](https://github.com/Akuwatoga/FreeDisplay/releases/latest)
2. Open the DMG and drag **FreeDisplay.app** to **Applications**
3. First launch: right-click → **Open** (unsigned app, one-time approval), or strip the quarantine attribute:
   ```bash
   sudo xattr -rd com.apple.quarantine /Applications/FreeDisplay.app
   ```

### Option 2: Build from Source

```bash
brew install xcodegen
git clone https://github.com/Akuwatoga/FreeDisplay.git
cd FreeDisplay
git checkout tahoe-compat
xcodegen generate
xcodebuild -scheme FreeDisplay -configuration Release CODE_SIGN_IDENTITY="-" CODE_SIGNING_REQUIRED=NO build
```

---

## Permissions

| Permission | Why |
|------------|-----|
| **Accessibility** | Required for brightness key interception on external displays |

No internet connection required (except optional update checks via GitHub Releases API).

---

## Tech Stack

- **Swift 6** + **SwiftUI** (MenuBarExtra)
- **IOKit** — DDC/CI I2C for hardware brightness/contrast
- **CoreGraphics** — Display enumeration, resolution, arrangement
- **ColorSync** — ICC color profile management
- **CGVirtualDisplay** — Virtual display creation (private API, macOS 14+)
- **CoreDisplay** — Built-in display brightness reading (private API, via dlopen)
- Zero third-party dependencies

---

## Project Structure

```
FreeDisplay/
├── App/              # AppDelegate, app entry point
├── Models/           # DisplayInfo, DisplayMode, DisplayPreset
├── Services/         # System-level services (DDC, brightness, resolution, gamma, etc.)
└── Views/            # SwiftUI views for each feature section
```

---

## How It Works

FreeDisplay sits in your menu bar and talks directly to your displays:

- **External monitors**: Uses DDC/CI protocol over I2C (Intel) or IOAVService (Apple Silicon) to control hardware brightness, contrast, and other settings
- **Built-in display**: Uses CoreGraphics gamma tables for software brightness adjustment
- **Brightness keys**: Installs a CGEventTap to intercept keyboard brightness keys and route them to the display under your mouse cursor
- **Auto brightness**: Polls the built-in display brightness via CoreDisplay private API and proportionally adjusts external displays
- **HiDPI**: Creates virtual displays via CGVirtualDisplay private API, or writes display override plists for persistent HiDPI

---

## Branches & Releases

| Branch | Purpose |
|---|---|
| `main` | Tracks upstream `huberdf/FreeDisplay` |
| `tahoe-compat` | Active development branch for macOS 26 (Tahoe) compatibility |

Releases are tagged following semantic versioning with a Tahoe channel suffix (e.g. `v1.0.1-tahoe.1`).

---

## Contributing

Issues and PRs welcome. This project uses:
- `xcodegen` for project generation (edit `project.yml`, not `.xcodeproj`)
- Swift 6 with `SWIFT_STRICT_CONCURRENCY: minimal`
- MVVM architecture (View → ViewModel → Service)
- Conventional commits (`fix(scope): ...`, `feat(scope): ...`, `chore: ...`)

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

## Acknowledgments

- Upstream: [huberdf/FreeDisplay](https://github.com/huberdf/FreeDisplay) — the original project this fork is based on
- Inspired by [BetterDisplay](https://github.com/waydabber/BetterDisplay), [MonitorControl](https://github.com/MonitorControl/MonitorControl), and [Lunar](https://lunar.fyi/)
- CGVirtualDisplay bridging header based on [Chromium's virtual_display_mac_util.mm](https://chromium.googlesource.com/chromium/src/+/main/ui/display/mac/test/virtual_display_mac_util.mm)
