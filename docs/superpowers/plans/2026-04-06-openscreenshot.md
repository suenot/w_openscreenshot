# OpenScreenshot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a macOS menu bar app that captures a selected screen region, downscales it by a configurable factor, and copies the PNG to clipboard via Cmd+Shift+6.

**Architecture:** SwiftUI + AppKit hybrid — `NSApplicationDelegate` manages menu bar and hotkey, two windows (`NSWindow` overlay + `NSPanel` toolbar) handle UI, `ScreenCaptureKit` does capture, `Core Image` does resize.

**Tech Stack:** Swift 5.9+, SwiftUI, AppKit, ScreenCaptureKit, Core Image, ServiceManagement, CGEventTap — macOS 13+ target (covers SMAppService API).

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `OpenScreenshot/App/OpenScreenshotApp.swift` | Create | `@main` entry, `NSApplicationDelegateAdaptor`, hide Dock icon |
| `OpenScreenshot/App/AppDelegate.swift` | Create | Menu bar item, hotkey registration, Launch at Login toggle |
| `OpenScreenshot/Capture/ScalePreset.swift` | Create | `enum ScalePreset` with raw values and display strings |
| `OpenScreenshot/Capture/ScreenCaptureManager.swift` | Create | Permission check, SCKit capture, CIImage resize, clipboard write |
| `OpenScreenshot/UI/SelectionView.swift` | Create | SwiftUI view: drag rect drawing, size tooltip |
| `OpenScreenshot/UI/CaptureOverlayWindow.swift` | Create | Fullscreen transparent `NSWindow`, hosts `SelectionView` |
| `OpenScreenshot/UI/ToolbarView.swift` | Create | SwiftUI toolbar: close, mode, Options popover, Capture button |
| `OpenScreenshot/UI/ToolbarPanel.swift` | Create | Floating `NSPanel`, hosts `ToolbarView`, positioned bottom-center |
| `OpenScreenshot/Resources/Assets.xcassets` | Create | App icon + menu bar icon template image |
| `OpenScreenshot.xcodeproj` | Create | Xcode project with entitlements for Screen Recording + Accessibility |

---

## Task 1: Xcode Project Scaffold

**Files:**
- Create: `OpenScreenshot.xcodeproj` (via Xcode or `xcodegen`)
- Create: `OpenScreenshot/OpenScreenshot.entitlements`
- Create: `OpenScreenshot/Info.plist`

- [ ] **Step 1: Create Xcode project**

Open Xcode → New Project → macOS → App. Settings:
- Product Name: `OpenScreenshot`
- Bundle Identifier: `com.yourname.openscreenshot`
- Interface: SwiftUI
- Language: Swift
- Minimum Deployment: macOS 13.0
- Uncheck "Include Tests" for now

- [ ] **Step 2: Configure Info.plist — hide Dock icon**

In `Info.plist`, add key:
```xml
<key>LSUIElement</key>
<true/>
```
This makes the app a "agent" app — no Dock icon, no main menu bar.

- [ ] **Step 3: Add entitlements**

In `OpenScreenshot.entitlements`, ensure these keys exist (Xcode may add them via Signing & Capabilities):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.app-sandbox</key>
    <false/>
</dict>
</plist>
```
Note: sandbox must be OFF for `CGEventTap` (global hotkey) to work.

- [ ] **Step 4: Add Privacy descriptions to Info.plist**

```xml
<key>NSScreenCaptureDescription</key>
<string>OpenScreenshot needs screen recording permission to capture screenshots.</string>
<key>NSAppleEventsUsageDescription</key>
<string>OpenScreenshot needs accessibility permission to register the global hotkey.</string>
```

- [ ] **Step 5: Build to verify empty project compiles**

In Xcode: Cmd+B. Expected: Build Succeeded with 0 errors.

- [ ] **Step 6: Commit**

```bash
git init
git add .
git commit -m "chore: scaffold Xcode project for OpenScreenshot"
```

---

## Task 2: ScalePreset enum

**Files:**
- Create: `OpenScreenshot/Capture/ScalePreset.swift`

- [ ] **Step 1: Create the file**

```swift
// OpenScreenshot/Capture/ScalePreset.swift
import Foundation

enum ScalePreset: String, CaseIterable {
    case full    = "1x"
    case half    = "1/2x"
    case quarter = "1/4x"
    case eighth  = "1/8x"

    var factor: Double {
        switch self {
        case .full:    return 1.0
        case .half:    return 0.5
        case .quarter: return 0.25
        case .eighth:  return 0.125
        }
    }

    var displayName: String { rawValue }

    static var `default`: ScalePreset { .half }

    static func load() -> ScalePreset {
        let raw = UserDefaults.standard.string(forKey: "captureScale") ?? ""
        return ScalePreset(rawValue: raw) ?? .default
    }

    func save() {
        UserDefaults.standard.set(self.rawValue, forKey: "captureScale")
    }
}
```

- [ ] **Step 2: Build to verify it compiles**

Cmd+B. Expected: Build Succeeded.

- [ ] **Step 3: Commit**

```bash
git add OpenScreenshot/Capture/ScalePreset.swift
git commit -m "feat: add ScalePreset enum with UserDefaults persistence"
```

---

## Task 3: ScreenCaptureManager

**Files:**
- Create: `OpenScreenshot/Capture/ScreenCaptureManager.swift`

- [ ] **Step 1: Create the file**

```swift
// OpenScreenshot/Capture/ScreenCaptureManager.swift
import Foundation
import ScreenCaptureKit
import CoreImage
import AppKit

@MainActor
class ScreenCaptureManager {

    static let shared = ScreenCaptureManager()
    private init() {}

    // MARK: - Permission

    func requestPermission() async -> Bool {
        do {
            // Triggers the system permission dialog on first call
            try await SCShareableContent.excludingDesktopWindows(false, onScreenWindowsOnly: true)
            return true
        } catch {
            return false
        }
    }

    // MARK: - Capture

    /// Captures `rect` (in screen points, flipped coordinates) and scales by `preset.factor`.
    /// Returns PNG data or nil on failure.
    func capture(rect: CGRect, preset: ScalePreset) async -> Data? {
        guard let display = await primaryDisplay() else { return nil }

        let filter = SCContentFilter(display: display, excludingApplications: [], exceptingWindows: [])
        let config = SCStreamConfiguration()

        // Convert point rect to pixel rect (Retina: multiply by backingScaleFactor)
        let scale = NSScreen.main?.backingScaleFactor ?? 2.0
        let pixelRect = CGRect(
            x: rect.origin.x * scale,
            y: rect.origin.y * scale,
            width: rect.width * scale,
            height: rect.height * scale
        )

        config.width = Int(pixelRect.width)
        config.height = Int(pixelRect.height)
        config.sourceRect = pixelRect
        config.scalesToFit = false
        config.pixelFormat = kCVPixelFormatType_32BGRA

        do {
            let cgImage = try await SCScreenshotManager.captureImage(
                contentFilter: filter,
                configuration: config
            )
            let scaled = applyScale(cgImage: cgImage, factor: preset.factor)
            return pngData(from: scaled)
        } catch {
            print("Capture error: \(error)")
            return nil
        }
    }

    // MARK: - Private

    private func primaryDisplay() async -> SCDisplay? {
        do {
            let content = try await SCShareableContent.excludingDesktopWindows(false, onScreenWindowsOnly: true)
            return content.displays.first
        } catch {
            return nil
        }
    }

    private func applyScale(cgImage: CGImage, factor: Double) -> CGImage {
        guard factor != 1.0 else { return cgImage }
        let ciImage = CIImage(cgImage: cgImage)
        guard let filter = CIFilter(name: "CILanczosScaleTransform") else { return cgImage }
        filter.setValue(ciImage, forKey: kCIInputImageKey)
        filter.setValue(factor, forKey: kCIInputScaleKey)
        filter.setValue(1.0, forKey: kCIInputAspectRatioKey)
        guard let outputCI = filter.outputImage else { return cgImage }
        let ctx = CIContext()
        let outputRect = CGRect(
            origin: .zero,
            size: CGSize(width: Double(cgImage.width) * factor,
                         height: Double(cgImage.height) * factor)
        )
        guard let result = ctx.createCGImage(outputCI, from: outputRect) else { return cgImage }
        return result
    }

    private func pngData(from cgImage: CGImage) -> Data? {
        let nsImage = NSImage(cgImage: cgImage, size: .zero)
        guard let tiff = nsImage.tiffRepresentation,
              let bitmap = NSBitmapImageRep(data: tiff) else { return nil }
        return bitmap.representation(using: .png, properties: [:])
    }

    // MARK: - Clipboard

    func copyToClipboard(pngData: Data) {
        let pasteboard = NSPasteboard.general
        pasteboard.clearContents()
        pasteboard.setData(pngData, forType: .png)
    }
}
```

- [ ] **Step 2: Build to verify it compiles**

Cmd+B. Expected: Build Succeeded. If ScreenCaptureKit import fails, ensure target is macOS 13+.

- [ ] **Step 3: Commit**

```bash
git add OpenScreenshot/Capture/ScreenCaptureManager.swift
git commit -m "feat: add ScreenCaptureManager with SCKit capture and CIImage scaling"
```

---

## Task 4: SelectionView (SwiftUI drag-to-select)

**Files:**
- Create: `OpenScreenshot/UI/SelectionView.swift`

- [ ] **Step 1: Create the file**

```swift
// OpenScreenshot/UI/SelectionView.swift
import SwiftUI

struct SelectionView: View {
    @Binding var selectionRect: CGRect   // in view coordinates (points)
    @Binding var isDragging: Bool

    var body: some View {
        GeometryReader { geo in
            ZStack {
                // Dim everything outside selection while dragging
                if isDragging {
                    Color.black.opacity(0.3)
                        .allowsHitTesting(false)
                }

                if isDragging && selectionRect != .zero {
                    // Clear hole punched by blending mode
                    Rectangle()
                        .fill(Color.clear)
                        .frame(width: selectionRect.width, height: selectionRect.height)
                        .position(x: selectionRect.midX, y: selectionRect.midY)
                        .blendMode(.destinationOut)

                    // Selection border
                    Rectangle()
                        .stroke(Color.white, lineWidth: 1)
                        .frame(width: selectionRect.width, height: selectionRect.height)
                        .position(x: selectionRect.midX, y: selectionRect.midY)
                        .allowsHitTesting(false)

                    // Size label
                    let scale = NSScreen.main?.backingScaleFactor ?? 2.0
                    let pw = Int(selectionRect.width * scale)
                    let ph = Int(selectionRect.height * scale)
                    Text("\(pw) × \(ph)")
                        .font(.system(size: 11, weight: .medium))
                        .foregroundColor(.white)
                        .padding(.horizontal, 6)
                        .padding(.vertical, 3)
                        .background(Color.black.opacity(0.6))
                        .cornerRadius(4)
                        .position(
                            x: min(selectionRect.maxX, geo.size.width - 60),
                            y: max(selectionRect.minY - 20, 16)
                        )
                        .allowsHitTesting(false)
                }
            }
            .compositingGroup()
        }
    }
}
```

- [ ] **Step 2: Build to verify it compiles**

Cmd+B. Expected: Build Succeeded.

- [ ] **Step 3: Commit**

```bash
git add OpenScreenshot/UI/SelectionView.swift
git commit -m "feat: add SelectionView with drag rect and pixel size label"
```

---

## Task 5: CaptureOverlayWindow

**Files:**
- Create: `OpenScreenshot/UI/CaptureOverlayWindow.swift`

- [ ] **Step 1: Create the file**

```swift
// OpenScreenshot/UI/CaptureOverlayWindow.swift
import AppKit
import SwiftUI

class CaptureOverlayWindow: NSWindow {

    var onCapture: ((CGRect) -> Void)?
    var onDismiss: (() -> Void)?

    private var selectionRect: CGRect = .zero
    private var dragStartPoint: NSPoint = .zero
    private var isDragging: Bool = false

    private var hostingView: NSHostingView<AnyView>?

    // Observable state passed into SwiftUI
    private let state = OverlayState()

    init() {
        let screen = NSScreen.main ?? NSScreen.screens[0]
        super.init(
            contentRect: screen.frame,
            styleMask: [.borderless],
            backing: .buffered,
            defer: false
        )
        self.level = NSWindow.Level(rawValue: Int(CGWindowLevelForKey(.screenSaverWindow)))
        self.isOpaque = false
        self.backgroundColor = .clear
        self.ignoresMouseEvents = false
        self.collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]
        self.acceptsMouseMovedEvents = true

        let view = SelectionStateView(state: state)
        let hosting = NSHostingView(rootView: AnyView(view))
        hosting.frame = self.contentView?.bounds ?? .zero
        hosting.autoresizingMask = [.width, .height]
        self.contentView = hosting
        self.hostingView = hosting
    }

    override var acceptsFirstResponder: Bool { true }
    override func becomeKey() -> Bool { true }

    // MARK: - Mouse

    override func mouseDown(with event: NSEvent) {
        dragStartPoint = event.locationInWindow
        isDragging = false
        selectionRect = .zero
        state.selectionRect = .zero
        state.isDragging = false
    }

    override func mouseDragged(with event: NSEvent) {
        let current = event.locationInWindow
        isDragging = true
        let x = min(dragStartPoint.x, current.x)
        let y = min(dragStartPoint.y, current.y)
        let w = abs(current.x - dragStartPoint.x)
        let h = abs(current.y - dragStartPoint.y)
        selectionRect = CGRect(x: x, y: y, width: w, height: h)
        state.selectionRect = selectionRect
        state.isDragging = true
    }

    override func mouseUp(with event: NSEvent) {
        // Don't fire capture on click — user uses Capture button
        state.isDragging = selectionRect != .zero
    }

    // MARK: - Keyboard

    override func keyDown(with event: NSEvent) {
        if event.keyCode == 53 { // Escape
            onDismiss?()
        }
    }

    // MARK: - Cursor

    func showWithCrosshair() {
        NSCursor.crosshair.set()
        self.makeKeyAndOrderFront(nil)
    }

    func hide() {
        self.orderOut(nil)
        NSCursor.arrow.set()
    }

    /// Returns the current selection rect in screen coordinates (top-left origin, points)
    func currentSelectionInScreenCoords() -> CGRect? {
        guard selectionRect != .zero,
              selectionRect.width > 5,
              selectionRect.height > 5 else { return nil }
        let screen = NSScreen.main ?? NSScreen.screens[0]
        // NSWindow uses flipped y; convert to screen coords
        let screenY = screen.frame.height - selectionRect.maxY
        return CGRect(
            x: selectionRect.minX + screen.frame.minX,
            y: screenY + screen.frame.minY,
            width: selectionRect.width,
            height: selectionRect.height
        )
    }
}

// MARK: - Observable state bridge

class OverlayState: ObservableObject {
    @Published var selectionRect: CGRect = .zero
    @Published var isDragging: Bool = false
}

struct SelectionStateView: View {
    @ObservedObject var state: OverlayState
    var body: some View {
        SelectionView(
            selectionRect: Binding(get: { state.selectionRect }, set: { state.selectionRect = $0 }),
            isDragging: Binding(get: { state.isDragging }, set: { state.isDragging = $0 })
        )
    }
}
```

- [ ] **Step 2: Build to verify it compiles**

Cmd+B. Expected: Build Succeeded.

- [ ] **Step 3: Commit**

```bash
git add OpenScreenshot/UI/CaptureOverlayWindow.swift
git commit -m "feat: add CaptureOverlayWindow with drag-to-select and crosshair"
```

---

## Task 6: ToolbarView (SwiftUI)

**Files:**
- Create: `OpenScreenshot/UI/ToolbarView.swift`

- [ ] **Step 1: Create the file**

```swift
// OpenScreenshot/UI/ToolbarView.swift
import SwiftUI

struct ToolbarView: View {
    @Binding var selectedScale: ScalePreset
    @State private var showOptions = false
    var onClose: () -> Void
    var onCapture: () -> Void

    var body: some View {
        HStack(spacing: 4) {
            // Close
            Button(action: onClose) {
                Image(systemName: "xmark.circle.fill")
                    .resizable()
                    .frame(width: 16, height: 16)
                    .foregroundColor(.secondary)
            }
            .buttonStyle(.plain)
            .padding(.leading, 8)

            Divider().frame(height: 20)

            // Mode: region select (only mode, always active)
            Button(action: {}) {
                Image(systemName: "selection.pin.in.out")
                    .frame(width: 28, height: 28)
                    .background(Color.accentColor.opacity(0.2))
                    .cornerRadius(6)
            }
            .buttonStyle(.plain)
            .disabled(true)  // always selected

            Divider().frame(height: 20)

            // Options
            Button(action: { showOptions.toggle() }) {
                HStack(spacing: 2) {
                    Text("Options")
                        .font(.system(size: 13))
                    Image(systemName: "chevron.down")
                        .font(.system(size: 10))
                }
                .padding(.horizontal, 8)
                .padding(.vertical, 4)
            }
            .buttonStyle(.plain)
            .popover(isPresented: $showOptions, arrowEdge: .top) {
                OptionsPopover(selectedScale: $selectedScale)
            }

            // Capture
            Button(action: onCapture) {
                Text("Capture")
                    .font(.system(size: 13, weight: .medium))
                    .foregroundColor(.white)
                    .padding(.horizontal, 12)
                    .padding(.vertical, 5)
                    .background(Color.accentColor)
                    .cornerRadius(6)
            }
            .buttonStyle(.plain)
            .padding(.trailing, 8)
        }
        .frame(height: 44)
        .background(.regularMaterial)
        .cornerRadius(10)
        .shadow(color: .black.opacity(0.3), radius: 8, y: 2)
    }
}

struct OptionsPopover: View {
    @Binding var selectedScale: ScalePreset

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            Text("Scale")
                .font(.system(size: 11))
                .foregroundColor(.secondary)
                .padding(.top, 8)
                .padding(.horizontal, 12)

            ForEach(ScalePreset.allCases, id: \.self) { preset in
                Button(action: {
                    selectedScale = preset
                    preset.save()
                }) {
                    HStack {
                        if selectedScale == preset {
                            Image(systemName: "checkmark")
                                .frame(width: 14)
                        } else {
                            Spacer().frame(width: 14)
                        }
                        Text(preset.displayName)
                            .font(.system(size: 13))
                        Spacer()
                    }
                    .padding(.horizontal, 12)
                    .padding(.vertical, 4)
                    .contentShape(Rectangle())
                }
                .buttonStyle(.plain)
            }
        }
        .padding(.bottom, 8)
        .frame(minWidth: 120)
    }
}
```

- [ ] **Step 2: Build to verify it compiles**

Cmd+B. Expected: Build Succeeded.

- [ ] **Step 3: Commit**

```bash
git add OpenScreenshot/UI/ToolbarView.swift
git commit -m "feat: add ToolbarView with Options popover and scale picker"
```

---

## Task 7: ToolbarPanel (NSPanel)

**Files:**
- Create: `OpenScreenshot/UI/ToolbarPanel.swift`

- [ ] **Step 1: Create the file**

```swift
// OpenScreenshot/UI/ToolbarPanel.swift
import AppKit
import SwiftUI

class ToolbarPanel: NSPanel {

    var onClose: (() -> Void)?
    var onCapture: (() -> Void)?

    private let scaleState = ToolbarScaleState()

    var selectedScale: ScalePreset {
        get { scaleState.scale }
        set { scaleState.scale = newValue }
    }

    init() {
        super.init(
            contentRect: NSRect(x: 0, y: 0, width: 320, height: 52),
            styleMask: [.borderless, .nonactivatingPanel],
            backing: .buffered,
            defer: false
        )
        self.level = NSWindow.Level(rawValue: Int(CGWindowLevelForKey(.floatingWindow)))
        self.isOpaque = false
        self.backgroundColor = .clear
        self.hasShadow = false
        self.collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]
        self.isMovableByWindowBackground = false

        setupContent()
    }

    private func setupContent() {
        let view = ToolbarPanelView(state: scaleState, onClose: { [weak self] in
            self?.onClose?()
        }, onCapture: { [weak self] in
            self?.onCapture?()
        })
        let hosting = NSHostingView(rootView: view)
        hosting.frame = self.contentView?.bounds ?? .zero
        hosting.autoresizingMask = [.width, .height]
        self.contentView = hosting
    }

    func showCentered(on screen: NSScreen) {
        let sw = screen.visibleFrame.width
        let sh = screen.visibleFrame
        let panelW: CGFloat = 320
        let panelH: CGFloat = 52
        let x = screen.frame.minX + (sw - panelW) / 2
        let y = screen.frame.minY + sh.minY + 20  // 20pt above bottom safe area
        self.setFrameOrigin(NSPoint(x: x, y: y))
        self.orderFront(nil)
    }

    func hide() {
        self.orderOut(nil)
    }
}

class ToolbarScaleState: ObservableObject {
    @Published var scale: ScalePreset = ScalePreset.load()
}

struct ToolbarPanelView: View {
    @ObservedObject var state: ToolbarScaleState
    var onClose: () -> Void
    var onCapture: () -> Void

    var body: some View {
        ToolbarView(
            selectedScale: Binding(get: { state.scale }, set: { state.scale = $0 }),
            onClose: onClose,
            onCapture: onCapture
        )
        .padding(4)
    }
}
```

- [ ] **Step 2: Build to verify it compiles**

Cmd+B. Expected: Build Succeeded.

- [ ] **Step 3: Commit**

```bash
git add OpenScreenshot/UI/ToolbarPanel.swift
git commit -m "feat: add ToolbarPanel as floating NSPanel with SwiftUI content"
```

---

## Task 8: AppDelegate — menu bar + hotkey + launch at login

**Files:**
- Create: `OpenScreenshot/App/AppDelegate.swift`

- [ ] **Step 1: Create the file**

```swift
// OpenScreenshot/App/AppDelegate.swift
import AppKit
import ServiceManagement
import Carbon

class AppDelegate: NSObject, NSApplicationDelegate {

    private var statusItem: NSStatusItem!
    private var overlayWindow: CaptureOverlayWindow?
    private var toolbarPanel: ToolbarPanel?
    private var eventTap: CFMachPort?

    func applicationDidFinishLaunching(_ notification: Notification) {
        NSApp.setActivationPolicy(.accessory)  // no Dock icon
        setupMenuBar()
        setupHotkey()
        requestPermissionsIfNeeded()
    }

    // MARK: - Menu Bar

    private func setupMenuBar() {
        statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.squareLength)
        if let button = statusItem.button {
            button.image = NSImage(systemSymbolName: "viewfinder", accessibilityDescription: "OpenScreenshot")
            button.image?.isTemplate = true
        }

        let menu = NSMenu()
        menu.addItem(withTitle: "Take Screenshot", action: #selector(openScreenshotTool), keyEquivalent: "")
        menu.addItem(.separator())

        let scaleItem = NSMenuItem(title: "Scale: \(ScalePreset.load().displayName)", action: nil, keyEquivalent: "")
        scaleItem.tag = 100
        menu.addItem(scaleItem)

        menu.addItem(.separator())

        let loginItem = NSMenuItem(title: "Launch at Login", action: #selector(toggleLaunchAtLogin), keyEquivalent: "")
        loginItem.state = launchAtLoginEnabled ? .on : .off
        loginItem.tag = 101
        menu.addItem(loginItem)

        menu.addItem(.separator())
        menu.addItem(withTitle: "Quit OpenScreenshot", action: #selector(NSApp.terminate(_:)), keyEquivalent: "q")

        statusItem.menu = menu
    }

    @objc func openScreenshotTool() {
        showCaptureUI()
    }

    // MARK: - Capture UI

    private func showCaptureUI() {
        let overlay = CaptureOverlayWindow()
        let toolbar = ToolbarPanel()

        overlay.onDismiss = { [weak self] in self?.hideCaptureUI() }
        toolbar.onClose   = { [weak self] in self?.hideCaptureUI() }
        toolbar.onCapture = { [weak self] in self?.performCapture(overlay: overlay, toolbar: toolbar) }

        let screen = NSScreen.main ?? NSScreen.screens[0]
        overlay.showWithCrosshair()
        toolbar.showCentered(on: screen)

        self.overlayWindow = overlay
        self.toolbarPanel = toolbar
    }

    private func hideCaptureUI() {
        overlayWindow?.hide()
        toolbarPanel?.hide()
        overlayWindow = nil
        toolbarPanel = nil
    }

    private func performCapture(overlay: CaptureOverlayWindow, toolbar: ToolbarPanel) {
        guard let rect = overlay.currentSelectionInScreenCoords() else {
            // No selection — flash the selection area or just ignore
            return
        }
        let preset = toolbar.selectedScale
        hideCaptureUI()

        Task { @MainActor in
            guard let pngData = await ScreenCaptureManager.shared.capture(rect: rect, preset: preset) else {
                return
            }
            ScreenCaptureManager.shared.copyToClipboard(pngData: pngData)
            // Update menu scale label
            updateScaleMenuItem(preset: preset)
        }
    }

    private func updateScaleMenuItem(preset: ScalePreset) {
        guard let menu = statusItem.menu,
              let item = menu.item(withTag: 100) else { return }
        item.title = "Scale: \(preset.displayName)"
    }

    // MARK: - Global Hotkey (Cmd+Shift+6)

    private func setupHotkey() {
        // Request Accessibility permission for CGEventTap
        let options = [kAXTrustedCheckOptionPrompt.takeUnretainedValue() as String: true] as CFDictionary
        guard AXIsProcessTrustedWithOptions(options) else {
            print("Accessibility not yet granted — hotkey will not work until granted")
            return
        }
        installEventTap()
    }

    private func installEventTap() {
        let mask = CGEventMask(1 << CGEventType.keyDown.rawValue)
        let tap = CGEventTap.create(
            tap: .cgSessionEventTap,
            place: .headInsertEventTap,
            options: .defaultTap,
            eventsOfInterest: mask,
            callback: { proxy, type, event, refcon -> Unmanaged<CGEvent>? in
                guard type == .keyDown else { return Unmanaged.passRetained(event) }
                let flags = event.flags
                let keycode = event.getIntegerValueField(.keyboardEventKeycode)
                // keycode 22 = 6, Cmd+Shift = .maskCommand | .maskShift
                let isCmd6 = keycode == 22
                    && flags.contains(.maskCommand)
                    && flags.contains(.maskShift)
                    && !flags.contains(.maskAlternate)
                    && !flags.contains(.maskControl)
                if isCmd6 {
                    let delegate = Unmanaged<AppDelegate>.fromOpaque(refcon!).takeUnretainedValue()
                    DispatchQueue.main.async { delegate.showCaptureUI() }
                    return nil  // consume event
                }
                return Unmanaged.passRetained(event)
            },
            userInfo: Unmanaged.passUnretained(self).toOpaque()
        )

        guard let tap else {
            print("Failed to create CGEventTap — check Accessibility permission")
            return
        }

        eventTap = tap
        let runLoopSource = CFMachPortCreateRunLoopSource(kCFAllocatorDefault, tap, 0)
        CFRunLoopAddSource(CFRunLoopGetCurrent(), runLoopSource, .commonModes)
        CGEvent.tapEnable(tap: tap, enable: true)
    }

    // MARK: - Launch at Login

    private var launchAtLoginEnabled: Bool {
        if #available(macOS 13.0, *) {
            return SMAppService.mainApp.status == .enabled
        }
        return false
    }

    @objc func toggleLaunchAtLogin() {
        if #available(macOS 13.0, *) {
            let service = SMAppService.mainApp
            do {
                if service.status == .enabled {
                    try service.unregister()
                } else {
                    try service.register()
                }
            } catch {
                print("Launch at Login error: \(error)")
            }
        }
        // Update checkmark
        if let item = statusItem.menu?.item(withTag: 101) {
            item.state = launchAtLoginEnabled ? .on : .off
        }
    }

    // MARK: - Permissions

    private func requestPermissionsIfNeeded() {
        Task {
            _ = await ScreenCaptureManager.shared.requestPermission()
        }
    }
}
```

- [ ] **Step 2: Build to verify it compiles**

Cmd+B. Expected: Build Succeeded.

- [ ] **Step 3: Commit**

```bash
git add OpenScreenshot/App/AppDelegate.swift
git commit -m "feat: add AppDelegate with menu bar, global hotkey, and launch at login"
```

---

## Task 9: App Entry Point

**Files:**
- Create: `OpenScreenshot/App/OpenScreenshotApp.swift`

- [ ] **Step 1: Create the file**

```swift
// OpenScreenshot/App/OpenScreenshotApp.swift
import SwiftUI

@main
struct OpenScreenshotApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    var body: some Scene {
        // No WindowGroup — this is a menu bar only app
        Settings { EmptyView() }
    }
}
```

- [ ] **Step 2: Delete or replace generated ContentView.swift**

Remove the default `ContentView.swift` Xcode creates if present.

- [ ] **Step 3: Build and run**

Cmd+R. Expected:
- No Dock icon appears
- Menu bar shows viewfinder icon
- Clicking menu shows "Take Screenshot", scale, launch at login, quit

- [ ] **Step 4: Commit**

```bash
git add OpenScreenshot/App/OpenScreenshotApp.swift
git commit -m "feat: wire up app entry point with NSApplicationDelegateAdaptor"
```

---

## Task 10: End-to-End Verification

- [ ] **Step 1: Test hotkey**

Press Cmd+Shift+6. Expected: crosshair overlay appears + toolbar panel appears at bottom-center.

- [ ] **Step 2: Test drag selection**

Drag a region. Expected: blue semi-transparent rect drawn, pixel size shown in top-right corner of selection.

- [ ] **Step 3: Test Escape**

Press Escape. Expected: overlay and toolbar disappear, cursor returns to arrow.

- [ ] **Step 4: Test Options**

Press Cmd+Shift+6, click Options. Expected: popover with 4 scale presets, 1/2x checked by default.

- [ ] **Step 5: Test capture**

Drag a region, select 1/4x in Options, click Capture. Open any app and paste (Cmd+V). Expected: image pasted is ~1/4 the pixel dimensions of the selected area.

- [ ] **Step 6: Test persistence**

Quit and relaunch. Open Options — 1/4x should still be selected.

- [ ] **Step 7: Test Launch at Login**

Toggle "Launch at Login" in menu. Open System Settings → General → Login Items. Expected: OpenScreenshot appears in the list.

- [ ] **Step 8: Final commit**

```bash
git add -A
git commit -m "feat: OpenScreenshot v1.0 — region capture with configurable scale to clipboard"
```

---

## Troubleshooting

| Issue | Fix |
|---|---|
| `CGEventTap` returns nil | Grant Accessibility in System Settings → Privacy & Security → Accessibility |
| `SCScreenshotManager` throws | Grant Screen Recording in System Settings → Privacy & Security → Screen Recording |
| Scale menu shows wrong value after capture | `updateScaleMenuItem` is called — check `statusItem.menu?.item(withTag: 100)` |
| Toolbar not visible | Check `showCentered(on:)` — `visibleFrame` vs `frame` on multi-monitor setups |
| Popover appears behind overlay | Overlay `windowLevel` is `.screenSaver`; panel is `.floating` — both should appear above normal windows but popover may need `NSApp.activate(ignoringOtherApps: true)` before showing |
