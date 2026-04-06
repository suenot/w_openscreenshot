# OpenScreenshot — Design Spec
_Date: 2026-04-06_

## Context

MacBook Retina displays produce 2000–4000px screenshots. When sending these as context to LLMs, costs are disproportionately high. This app solves that by capturing a selected screen region and automatically downscaling it before copying to clipboard — replicating the macOS native Cmd+Shift+5 experience with a compression step added.

---

## Requirements

- **Trigger:** Global hotkey Cmd+Shift+6
- **Mode:** Select region only (rectangle drag)
- **Output:** PNG copied to clipboard (no file saved)
- **Compression:** Configurable scale preset (1x, 1/2x, 1/4x, 1/8x), persisted in UserDefaults
- **Lifecycle:** Menu bar app (no Dock icon), Launch at Login
- **Platform:** macOS 12.3+ (ScreenCaptureKit), Swift/SwiftUI

---

## Architecture

### Components

| Component | Type | Responsibility |
|---|---|---|
| `AppDelegate` | NSApplicationDelegate | App entry point, menu bar item, hotkey registration, Launch at Login |
| `CaptureOverlayWindow` | NSWindow (fullscreen, transparent) | Crosshair cursor, drag-to-select rectangle, size tooltip |
| `ToolbarPanel` | NSPanel (floating) | Mode buttons, Options popover, Capture button |
| `ScreenCaptureManager` | Service class | ScreenCaptureKit capture, Core Image resize, clipboard write |

### Data Flow

```
Cmd+Shift+6
  → CaptureOverlayWindow appears (fullscreen, all spaces)
  → ToolbarPanel appears (bottom-center, above Dock)
  → user drags selection rect
  → user clicks Capture
  → ScreenCaptureManager.capture(rect: CGRect, scale: ScalePreset)
      → SCScreenshotManager.captureImage(contentFilter:configuration:)
      → CIImage → scale filter → CGImage
      → NSImage → NSPasteboard.general
  → windows hide
```

---

## UI Components

### ToolbarPanel
- Floating `NSPanel`, `NSWindowStyleMask.borderless`, `NSWindowLevel.floating`
- Positioned bottom-center of screen (mimics native Cmd+Shift+5 panel)
- Buttons (left to right):
  - ✕ close button
  - Region select mode button (active/only mode)
  - **Options ▾** — opens popover
  - **Capture** — triggers capture

### Options Popover
- Label: "Scale"
- Segmented control or radio buttons: `1x` · `1/2x` · `1/4x` · `1/8x`
- Selection persisted to `UserDefaults` key `captureScale`
- Default: `1/2x`

### CaptureOverlayWindow
- `NSWindow` covering all screens, `NSWindowLevel.screenSaver`
- Background: clear, `isOpaque = false`
- On appear: cursor → crosshair
- On drag: draws selection rect — semi-transparent blue fill + white 1pt border
- Tooltip showing pixel dimensions of selection (physical pixels at chosen scale)
- On Escape: dismiss both windows

### Menu Bar
- SF Symbol icon: `viewfinder`
- Menu items:
  - "Take Screenshot" (triggers same flow as hotkey)
  - Separator
  - "Scale: 1/2x" (shows current preset, opens preferences)
  - Separator
  - "Launch at Login" (checkmark toggle)
  - "Quit OpenScreenshot"

---

## Capture & Compression

### ScreenCaptureManager

1. **Permission check:** `SCShareableContent.getExcludingDesktopWindows` — prompts system permission dialog on first run
2. **Display selection:** find `SCDisplay` containing the cursor's current position
3. **Capture:** `SCScreenshotManager.captureImage(contentFilter: SCContentFilter(display:), configuration: SCStreamConfiguration)` — returns `CGImage` at native Retina resolution
4. **Scale:** 
   - `1x` → no processing
   - `1/2x`, `1/4x`, `1/8x` → `CIImage` → `CIFilter(name: "CILanczosScaleTransform")` with appropriate scale factor
5. **Clipboard:** wrap in `NSImage`, write to `NSPasteboard.general` as PNG

### Scale math
Selection drag is in **points**. On Retina (2x), `500pt × 300pt` = `1000px × 600px` physical.  
Scale preset applies to physical pixels:
- `1x` → `1000 × 600` in clipboard
- `1/2x` → `500 × 300`
- `1/4x` → `250 × 150`
- `1/8x` → `125 × 75`

### Clipboard format
PNG (`NSBitmapImageFileType.png`) — lossless, optimal for LLM vision APIs.

---

## System Integration

### Global Hotkey
`CGEventTap` with `CGEventMaskBit(.keyDown)` — listens for Cmd+Shift+6 system-wide.  
Requires Accessibility permission (prompted on first use).

### Launch at Login
`ServiceManagement.SMAppService.mainApp.register()` / `.unregister()` (macOS 13+).  
Fallback for macOS 12: `SMLoginItemSetEnabled`.

---

## File Structure

```
OpenScreenshot/
├── OpenScreenshot.xcodeproj
├── OpenScreenshot/
│   ├── App/
│   │   ├── OpenScreenshotApp.swift       # @main, NSApplicationDelegateAdaptor
│   │   └── AppDelegate.swift             # menu bar, hotkey, launch at login
│   ├── Capture/
│   │   ├── ScreenCaptureManager.swift    # SCKit capture + CIImage resize
│   │   └── ScalePreset.swift             # enum: 1x, half, quarter, eighth
│   ├── UI/
│   │   ├── CaptureOverlayWindow.swift    # fullscreen transparent NSWindow
│   │   ├── SelectionView.swift           # SwiftUI view drawing the drag rect
│   │   ├── ToolbarPanel.swift            # NSPanel with SwiftUI content
│   │   └── ToolbarView.swift             # SwiftUI toolbar buttons + Options
│   └── Resources/
│       └── Assets.xcassets
```

---

## Permissions Required

| Permission | Why |
|---|---|
| Screen Recording | ScreenCaptureKit capture |
| Accessibility | Global hotkey via CGEventTap |

Both are requested at first launch with explanatory alert before system prompt.

---

## Verification

1. Build and run — app appears in menu bar only (no Dock icon)
2. Press Cmd+Shift+6 — overlay + toolbar appear
3. Drag a region — blue rectangle drawn, pixel size shown
4. Click Options → change scale to 1/4x
5. Click Capture — windows disappear
6. Paste in any app — image is present at ~1/4 original pixel size
7. Relaunch app — scale preset remembered
8. Toggle "Launch at Login" in menu — verify in System Settings > General > Login Items
9. Press Escape — overlay dismisses without capturing
