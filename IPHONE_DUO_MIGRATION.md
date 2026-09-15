# iPhone Duo Compatibility Guide for Claude Code

> **Purpose:** This document instructs Claude Code how to adapt an existing iOS app (SwiftUI and/or UIKit) for **iPhone Duo**, Apple's foldable, dual-display iPhone announced on September 9, 2026. Reference this file from your project's `CLAUDE.md` (e.g. `See IPHONE_DUO_MIGRATION.md for iPhone Duo adaptation rules`) or pass it directly into context when asking Claude Code to do the migration.
>
> **Sources:** Apple Tech Talks published Sept 9, 2026 (transcripts + session pages) and Apple Developer News. Full source list at the bottom.

---

## 1. Device facts (context Claude Code must assume)

- iPhone Duo is a **foldable iPhone with two displays**: a compact **outer display** and a large **inner display**. It is still an **iPhone app target**, not a new platform or idiom.
- **Hardware (Apple Newsroom, Sept 9, 2026):** outer display 5.4", inner foldable display 7.6", both Super Retina XDR with ProMotion (up to 120 Hz), Always-On, 3000 nits peak; inner display has a nano-texture finish. A20 Pro chip (6-core CPU, 7-core GPU, dual 16-core Neural Engine). Grade 5 titanium body; precision hinge of 100+ components. Pre-orders Oct 16, 2026; **ships Oct 23, 2026**.
- **Reported logical dimensions (secondary reporting, verify in simulator):** outer ≈ 466 × 678 pt @3x; inner ≈ 626 × 890 pt portrait / 890 × 626 pt landscape. Both displays share a ≈1.42:1 aspect ratio (close to √2), so proportional layouts and two-column documents scale cleanly between displays. Verified pixel sizes from the App Store Connect screenshot specs: outer 1398 × 2034 px, inner 2007 × 2853 px (the inner value does not match the reported pt figures at @3x, one more reason to confirm in the simulator).
- **No Face ID.** Biometric authentication is **Touch ID in the side button** (works open and closed); Apple Watch unlock is also supported. The outer display has a corner-placed Dynamic Island camera; the inner display has an **under-display FaceTime camera** (surfaces in layout only as an occlusion region when active).
- **Apple Pencil (USB-C) support on the inner display** arrives later in 2026 - relevant if the app has drawing/annotation features (PencilKit).
- **Poses:** closed, fully open (flat), partially folded ("book"), tent/laptop on a table, and rotated variants. Apps must resize live as the user opens, closes, folds, or rotates the device.
- **Size classes:**
  - Outer display: like other iPhones. Portrait = compact width / regular height. Landscape = compact / compact.
  - Inner display: **regular width / regular height** (sidebars and multi-column layouts are appropriate).
- **Orientation:** The outer display honors `UISupportedInterfaceOrientations` like any iPhone. The **inner display does NOT honor supported interface orientations** (an app that restricts orientation is scaled instead). Never branch layout on interface orientation, use size classes.
- **Controls move to the side:** On iPhone Duo, navigation bars, toolbars, and tab bars lay out **vertically** along the side of the display (near the status bar/camera) to maximize vertical space. The only pose keeping horizontal bars is the inner display in portrait. Bars automatically avoid system UI (status bar, Dynamic Island, camera).
- **Safe areas and layout margins are asymmetric.** Vertical bars produce leading/trailing insets that differ per side and per pose.
- **The fold (hinge) divides the inner display** when partially folded. System components (sheets, alerts, menus, toolbar buttons) automatically nudge interactive elements away from the fold ("fold avoidance"). Scrollable content does not need to avoid the fold.
- **Split View multitasking (new to iPhone):** two apps side by side in a **50-50 split**; also a stacked video + app layout (pinned Picture in Picture). All apps participate.
- **Multiple scenes:** iPhone Duo is the first iPhone supporting multiple instances of an app's UI. New windows can only be created on the **inner** display.
- **Cameras:** separate front cameras on outer and inner displays, plus a **Virtual Front Camera** that switches automatically as the device opens/closes. The active inner FaceTime camera is exposed to layout as an **occlusion reserved region**.
- **Compatibility baseline - behavior depends on the SDK the app is built with:**

| Built with | Outer display | Inner display | Bars |
|---|---|---|---|
| Pre-iOS 27 SDK | Fixed aspect ratio, letterboxed | Boxed at legacy size/ratio, centered | Standard horizontal bars, no Duo adaptation |
| iOS 27 SDK | Fits the screen but bounded by the status bar area | Extends to the left of the status bar area, avoids camera | Horizontal bars kept; fold displacement not fully active |
| **iOS 27.1 SDK** | **Edge-to-edge full screen** | **Edge-to-edge; fold division and camera occlusion regions fully recognized** | **Vertical navigation/tab bars activate automatically** |

- `UIRequiresFullScreen` is still honored, but the app **still resizes** when the device opens/closes. Do not rely on it as an opt-out.

## 2. Toolchain requirements

1. **Xcode 27.1** with the **iOS 27.1 SDK** (edge-to-edge layout, reserved regions, arrangements, hinge, scene accessory, and Duo camera APIs are 27.1 APIs).
2. Run on the **iPhone Duo simulator in Device Hub**; use the on-screen control buttons to open, close, rotate, and fold the device.
3. Xcode 27.1 ships the updated app-modernization analysis tool **"App Resizability"** (transcribed as "App Precisability" in the Tech Talk audio) - it audits fixed-size dependencies and now supports SwiftUI and iPhone Duo. Run it as an automated first pass.
4. Note (App Store): starting April 2027, apps must be built with the latest SDKs (iOS 27 family).
5. Timeline: Apple runs iPhone Duo **Group Labs on Sept 16-17, 2026** and **Developer Forums Q&As on Sept 23, 2026** (Photos & Camera, SwiftUI, UIKit sessions); the device **ships Oct 23, 2026** - adaptation should land before then.
6. App Store assets (verified, ASC screenshot specifications): iPhone Duo screenshots are **1398 × 2034 px** for the outer display and **2007 × 2853 px** for the inner display (swap for landscape). Upload support in App Store Connect arrives later this year - prepare the assets now.

## 3. Migration workflow for Claude Code

Execute in this order. Each phase lists what to search for (audit), what to change, and the exact APIs.

### Phase 0 - Audit the codebase (read-only pass)

Grep for these anti-patterns and list every occurrence before changing code:

| Anti-pattern | Search for | Why it breaks on iPhone Duo |
|---|---|---|
| Main screen references | `UIScreen.main`, `UIScreen.main.bounds`, `UIScreen.main.scale` | Ambiguous on a two-display device; will be deprecated |
| Idiom-based layout | `userInterfaceIdiom`, `UI_USER_INTERFACE_IDIOM` | Duo is `.phone` but can have regular/regular size classes |
| Orientation-based layout | `UIDevice.current.orientation`, `interfaceOrientation`, `UIApplication.shared.statusBarOrientation` | Inner display ignores supported orientations |
| Fixed widths / breakpoints | hardcoded screen widths (`375`, `390`, `393`, `430` …), device-model checks | New screen shapes and continuous resizing |
| Symmetric inset assumptions | code doubling one inset (e.g. `safeAreaInsets.left * 2`), shared constants for both sides | Safe areas/margins are asymmetric on Duo |
| Full-screen opt-outs | `UIRequiresFullScreen` in Info.plist | Honored, but app still resizes; misleading |
| Manual frame math from screen bounds | `view.frame = UIScreen.main.bounds` etc. | Use scene bounds / view bounds |
| Orientation locks | `supportedInterfaceOrientations` restricted to portrait | App gets scaled on the inner display; consider supporting landscape (tent pose) |
| Hardcoded Face ID assumptions | literal "Face ID" strings in UI/copy, `biometryType == .faceID` assumptions, `NSFaceIDUsageDescription`-only flows | iPhone Duo has **Touch ID in the side button**, no Face ID - query `LAContext.biometryType` and label UI accordingly |

### Phase 1 - Replace screen/idiom/orientation assumptions

- Replace `UIScreen.main` with local, dynamic concepts: the SwiftUI environment, `traitCollection`, or the scene's bounds. If screen access is unavoidable, get it from the window scene:

```swift
// UIKit - Before
let screenScale = UIScreen.main.scale
// After
let screenScale = traitCollection.displayScale

// If a screen reference is truly needed:
let screen = window?.windowScene?.screen
```

- Drive all layout decisions from **size classes**:

```swift
// SwiftUI
@Environment(\.horizontalSizeClass) private var horizontalSizeClass
@Environment(\.verticalSizeClass) private var verticalSizeClass

// UIKit
traitCollection.horizontalSizeClass
traitCollection.verticalSizeClass
```

- Design for exactly two width targets: **compact width** (outer display) and **regular width** (inner display). No per-pose custom layouts, no fixed breakpoints. The app should be freely resizable (iPad-style resizing and iPhone Mirroring on the Mac are the same muscle).
- **Biometrics:** never hardcode "Face ID" in UI text or flows. Query the actual hardware:

```swift
import LocalAuthentication
let context = LAContext()
_ = context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil)
switch context.biometryType {
case .touchID: label = "Touch ID"   // iPhone Duo (side-button Touch ID)
case .faceID:  label = "Face ID"
default:       label = "Passcode"
}
```

### Phase 2 - Safe areas, margins, and corners

- Interactive/foreground content **inside the safe area**; full-bleed background content may extend beyond it.

```swift
// SwiftUI: content is inside the safe area by default; backgrounds opt out:
.ignoresSafeArea()

// UIKit: manual layout
foreground.frame = view.bounds.inset(by: view.safeAreaInsets)
backgroundView.frame = view.bounds
// Auto Layout: constrain to view.safeAreaLayoutGuide
```

- **Handle each inset side independently** - never mirror one side to the other:

```swift
// WRONG - assumes symmetric insets
let width = view.bounds.width - view.safeAreaInsets.left * 2
// RIGHT
let width = view.bounds.inset(by: view.safeAreaInsets).width
```

- Match the new screen corner shapes with the Concentricity APIs (introduced iOS 26, updated for Duo):

```swift
// SwiftUI
ConcentricRectangle().fill(.green).padding(8).ignoresSafeArea()
// UIKit
// UICornerConfiguration
```

- For custom bars or edge-to-edge custom UI, use the new **reserved region safe-positioning API (iOS 27.1)**: SwiftUI's reserved region support and UIKit's `UIViewReservedRegion` let custom UI use maximum space without colliding with system UI.

### Phase 3 - Navigation, toolbars, and tab bars (vertical bars)

- Prefer **standard system containers** - they adapt to every pose for free:
  - `NavigationSplitView` / `UISplitViewController`: columns collapse to a stack when closed, tile/overlay when open.
  - `TabView` / `UITabBarController`: tabs lay out vertically when appropriate on both displays.
- On the inner display, offer richer navigation via a tab sidebar:

```swift
// SwiftUI
TabView { … }.defaultTabBarPlacement(.sidebar)
// UIKit
tabBarController.sidebar.preferredPlacement = .sidebar
```

- Vertical-bar behavior for toolbar content (from "Raise the bar with iPhone Duo" + HIG "Designing for iPhone Duo"):
  - System bars opt in automatically when using standard toolbar items; top toolbar items map to the top of the vertical bar, bottom items to the bottom; tab bars stay bottom-aligned.
  - **Prefer symbol-only (SF Symbol) toolbar items.** Wide items (text buttons like "Edit", amounts, segmented controls) stay in the horizontal nav/header area instead of moving to the side.
  - Vertical bars have fixed width and flexible height. Top-to-bottom ordering in the vertical bar: (1) primary navigation, Back/Close (SwiftUI `cancellationAction`, UIKit leading-edge items); (2) prominent confirm actions like Done/Save (SwiftUI `topBarPinnedTrailing`, UIKit `pinnedTrailingGroup`); (3) flexible spacer; (4) tool/tab icons by priority; (5) overflow menu.
  - The bar region is shared with system UI (status bar, Dynamic Island expanding vertically with Live Activities). When vertical space runs out, items fold bottom-up into the overflow menu, governed by `ToolbarItemVisibilityPriority` (SwiftUI) / `UIBarButtonItemVisibilityPriority` (UIKit) - give core actions high priority so they collapse last.
  - Group related items with `ToolbarItemGroup` (SwiftUI) / `UIBarButtonItemGroup` (UIKit); the system preserves original groupings and inserts vertical space between top and bottom bar groups.
  - Keep controls near the content they affect (e.g. list controls above the list, not on the far edge), and give every non-text toolbar item both a title and a symbol: the symbol shows in the bar, the title in the overflow menu.
  - Do not override the system's default bar placement; when space is tight, navigation-focused apps should move toolbar actions into the system overflow menu, while task-oriented apps should minimize the tab bar instead.

```swift
// Control the axis of a custom toolbar view
.toolbar {
    ToolbarItem { ProfileView() }
        .axisBehavior(.verticalPreferred)
}

// Bar region compression preference
.toolbarVerticalCompressionBehavior(.prefersToolbarItems)

// Explicit overflow menu for lower-priority actions
.toolbar {
    ToolbarOverflowMenu {
        Button("Scan") { … }
        Button("Connect") { … }
    }
}

// Keep critical items visible when space is tight
.toolbar {
    ToolbarItem { Button(…) }
        .visibilityPriority(.high)
}

// Opt out of vertical bars (rare - e.g. single-page, bottom-heavy apps
// like Calculator, or modals with a single close button)
NavigationStack { ContentView() }
    .toolbarVerticalBehavior(.disabled)
// UIKit equivalent: preferredVerticalBarBehavior = .never
```

- The vertical bar region is **shared with system UI** (status bar, Dynamic Island / Live Activities). If space runs out, app controls collapse into an overflow menu automatically.
- Sheets: on the outer display sheet controls also move to the side (disable the sheet's vertical bar if a single toolbar button fits better); on the inner display sheets are centered with standard horizontal bars; when partially folded, sheets slide aside from the fold automatically.

### Phase 4 - Fold-aware layout: reserved regions, displacement, arrangements

**Reserved regions (iOS 27.1)** represent hardware features the layout must respect. Two kinds:

- **Division regions** - the fold/hinge. Divides an area into multiple usable regions. Active only while the device is partially folded (width 0 when flat).
- **Occlusion regions** - smaller frames occluding content: the inner under-display FaceTime camera while active and, per the HIG, the outer display's Dynamic Island area.

```swift
// SwiftUI - query from a GeometryProxy (GeometryReader or onGeometryChange)
GeometryReader { proxy in
    let folds = proxy.reservedRegions(kind: .division)
    let all   = proxy.reservedRegions(kind: .division, options: .includeInactive)
    let cams  = proxy.reservedRegions(kind: .occlusion)
    let frames = folds.map(\.frame)
}

// UIKit
let regions = view.reservedRegions(kind: .division)
let frames = regions.map(\.frame)
```

- Only **active** regions are returned by default; query inactive ones (`.includeInactive`) for high-level decisions, e.g. prefer an even number of grid columns whenever a division region exists at all.
- **Displacement pattern** (design guidance for custom UI): when the device folds, move important elements to the region that supports their purpose instead of letting them straddle the fold.
  - Book pose: alerts and key actions move to the **trailing** region (continuity with the outer display when closing).
  - Tent/laptop pose: glanceable content to the **top** region, tappable controls to the **bottom** region.
  - Move related elements together; avoid excessive movement; keep things contextual (e.g. search field stays over the content it searches).
  - **Continuous scrolling content (feeds, articles, lists) does not displace** - scrolling already adapts. Grids: keep outer margins, increase spacing around the hinge so each container stays within its region.
  - Never hide functionality in a pose: every pose must expose the same features and hierarchy.

**Arrangements (iOS 27.1)** - a layout container between navigation containers and content containers, arranging a primary and a secondary view by rules; system-provided, fold-aware:

```swift
// SwiftUI
NavigationStack {
    ArrangementView {
        PlayerView()          // primary
    } secondary: {
        UpNextView()
    }
    .arrangementViewStyle(.split.axes(.horizontal)) // default: .split
    // or: .arrangementViewStyle(.overlay)
}

// React to fold-driven z-order changes in the overlay style
@Environment(\.overlayArrangementZIndex) private var zIndex: Int
var minimization: UpNextMinimization { zIndex > 0 ? .collapsed : .expanded }
```

```swift
// UIKit
let arrangementVC = UIArrangementViewController()
let nav = UINavigationController(rootViewController: arrangementVC)
arrangementVC.setViewController(PlayerViewController(), for: .primary)
arrangementVC.setViewController(UpNextViewController(), for: .secondary)
arrangementVC.updateArrangement(.split.axes(.horizontal))
let zIndex = arrangementVC.state(for: .primary)?.zIndex ?? 0
```

Choosing an arrangement:

- Existing `HStack`/`VStack`-style side-by-side layout, or main–detail content where neither view may be obscured → **split**.
- Existing `ZStack`-style layout, or clear foreground–background relationship (controls over scrollable content) → **overlay** (side-by-side when folded).
- **Do not** put navigation containers (e.g. `NavigationSplitView`) inside an `ArrangementView`, and do not put an `ArrangementView` inside `List`/`ScrollView`.

### Phase 5 - Multitasking, multiple scenes, hinge, scene accessories

- **Split View multitasking:** nothing special to implement beyond correct resizing - handle it with size classes and scene geometry. In split view, vertical bars can appear on **either** side of the app; test both sides (drag the app via the home indicator in the simulator).
- **Multiple scenes:** if the app supports multiple windows on iPad, it does on iPhone Duo too, but **new windows can only be created on the inner display**. Enable multi-scene support via `UIApplicationSceneManifest` in Info.plist (`UIApplicationSupportsMultipleScenes = YES`) if the app doesn't have it yet. Always handle scene-request errors; prefer the window scene activation action API (`UIWindowScene.ActivationAction`), which hides itself automatically when new windows aren't available.
- **Hinge APIs** - for interactions/effects (NOT for layout; layout uses reserved regions + arrangements):

```swift
// SwiftUI
GuitarView(pitchBend: pitchBend)
    .onHingeChange { previous, context in
        if let hinge = context.hinge, hinge.status == .partiallyOpen {
            pitchBend = calculatePitchBend(angle: hinge.angle)
        } else {
            pitchBend = 0
        }
    }
// UIKit: UIHingeInteraction
```

  - `context.hinge` is `nil` on devices without a hinge - always guard.
  - Status values: `closed`, `partiallyOpen`, `fullyOpen`, plus continuous angle updates.

- **Scene accessories** - show supplementary UI on the other display simultaneously; availability is system-controlled and dynamic:

```swift
CameraView(model: model)
    .sceneAccessory {
        CameraCaptureAccessory(isEnabled: $model.isEnabled) {
            TeleprompterView(model: model)
        }
        .onAvailabilityChange { model.isAvailable = $0 }
    }
```

  - The **camera capture accessory** pairs UI on the outer display while the main app is full screen on the inner display with an active camera session (e.g. show the subject their own preview or a teleprompter). Register the accessory on the same view as the camera UI; disable related controls when unavailable.

### Phase 6 - Camera apps (AVFoundation, iOS 27.1 SDK)

- **Virtual Front Camera:** a system-provided front camera that automatically switches between the inner (open) and outer (closed) cameras - the default, zero-effort path.
- To access cameras individually, new device types exist, e.g. `.builtInOuterUltraWideCamera`, `.builtInInnerUltraWideCamera`.
- **Direction coordinator** - track which camera to use as the device opens/closes and update the session:

```swift
directionCoordinator = AVCaptureDeviceDirectionCoordinator(
    view: view,
    deviceTypes: [
        .builtInOuterUltraWideCamera,
        .builtInInnerUltraWideCamera,
        .builtInDualWideCamera,
    ],
    changeHandler: { [weak self] map in
        self?.updateCameraSession(map)
    }
)
```

- In change handlers, work with `AVCaptureDeviceDescriptor` (main-actor-safe) rather than `AVCaptureDevice`.
- Using both displays at once (preview for the subject): create a **separate direction coordinator per display** together with the SceneAccessories API.
- Preview polish: set `AVCaptureVideoPreviewLayer.videoGravity` appropriately and adopt `AVCaptureDevice.dynamicAspectRatio` for correct preview aspect/positioning across poses.
- Rotation: adopt `AVCaptureDeviceRotationCoordinator`, then disable sensor orientation compensation (`AVCapturePhotoOutput.isCameraSensorOrientationCompensationEnabled = false`).
- If a viewfinder is central to the UX, keep important content and controls clear of the FaceTime camera occlusion region.

### Phase 7 - Design rules checklist (apply while touching UI code)

1. One app, one experience: hierarchy and functionality identical across outer/inner displays and every pose. A pose may get an optimized layout (e.g. media top / controls bottom in laptop pose) but never exclusive functionality.
2. Audit **centered layouts**: on the outer display most content needs a horizontal offset so it isn't behind the side controls - aligning to horizontal safe area insets does this automatically. Full-display centering is acceptable only for immersive, non-scrolling UIs whose interactive elements can't be covered; mixed approach (full-bleed background + inset scrollable foreground) is fine if all interactive elements live in the inset area.
3. Inner display should not be a stretched-out iPhone app: use split views, two-column rearrangements of stacked layouts, or a tab sidebar (information-dense apps).
4. Keep interactive elements out of the fold's curved region; rely on system components for automatic fold avoidance; scrollable content may pass through the fold.
5. Use standard system components and bars wherever possible - displacement, fold avoidance, vertical bars, and overflow come free.
6. Support accessibility across all poses.
7. Consider supporting landscape if the app is portrait-only (tent pose makes landscape common).
8. Games (HIG): a game may lock to portrait or landscape, but it must keep filling the screen as the pose changes; prefer adapting the aspect ratio over letterboxing or pillarboxing, and if padding is unavoidable, fill it with artwork rather than black bars.

### Phase 8 - Verification matrix (must pass before done)

Test in the iPhone Duo simulator (Device Hub, Xcode 27.1), in each of:

| # | Configuration | What to check |
|---|---|---|
| 1 | Outer display, portrait | Content offset from side controls; nothing under vertical bar |
| 2 | Outer display, landscape | Compact/compact layout; asymmetric insets |
| 3 | Inner display, flat, portrait | Regular/regular layout; horizontal bars OK; no stretched-phone look |
| 4 | Inner display, flat, landscape | Sidebar/split behavior |
| 5 | Partially folded (book) | Fold avoidance; displacement; no interactive elements in the fold |
| 6 | Tent / laptop pose | Controls reachable at bottom; content visible at top |
| 7 | Open ↔ close transitions | State preserved; smooth resize; hinge callbacks reset |
| 8 | Split View multitasking, app on LEFT | Vertical bar side switches; layout correct |
| 9 | Split View multitasking, app on RIGHT | Same |
| 10 | Video + app stacked layout | Vertical resizing in real time |
| 11 | Rotation in every pose | Size-class-driven layout only |
| 12 | Sheets/alerts/menus in every pose | Correct placement, fold avoidance |
| 13 | (Camera apps) open/close during session | Virtual front camera / direction coordinator switch |
| 14 | (Multi-scene apps) request window on outer display | Graceful failure / hidden activation action |

## 4. Quick reference - new/updated APIs by framework

| Area | SwiftUI | UIKit / AVFoundation |
|---|---|---|
| Size classes | `@Environment(\.horizontalSizeClass)` | `traitCollection.horizontalSizeClass` |
| Corners | `ConcentricRectangle` | `UICornerConfiguration` |
| Tab sidebar | `.defaultTabBarPlacement(.sidebar)` | `tabBarController.sidebar.preferredPlacement = .sidebar` |
| Vertical bars | `.axisBehavior(_)`, `.toolbarVerticalCompressionBehavior(_)`, `ToolbarOverflowMenu`, `.visibilityPriority(_)` (`ToolbarItemVisibilityPriority`), `.toolbarVerticalBehavior(.disabled)` | standard bar controllers adapt automatically; `UIBarButtonItemVisibilityPriority`, `preferredVerticalBarBehavior = .never` |
| Biometrics | - | `LAContext.biometryType` (Duo = `.touchID`, side button) |
| Reserved regions | `GeometryProxy.reservedRegions(kind:options:)` | `UIView.reservedRegions(kind:)`, `UIViewReservedRegion` |
| Arrangements | `ArrangementView`, `.arrangementViewStyle(.split/.overlay)`, `\.overlayArrangementZIndex` | `UIArrangementViewController`, `updateArrangement(_)`, `state(for:)` |
| Hinge | `.onHingeChange { previous, context in }` | `UIHingeInteraction` |
| Scenes | - | `UIWindowScene` activation action (auto-hides) |
| Scene accessories | `.sceneAccessory { }`, `CameraCaptureAccessory`, `.onAvailabilityChange { }` | - |
| Camera | - | Virtual Front Camera, `AVCaptureDeviceDirectionCoordinator`, `AVCaptureDeviceDescriptor`, `.builtInOuterUltraWideCamera` / `.builtInInnerUltraWideCamera`, `dynamicAspectRatio`, `AVCaptureDeviceRotationCoordinator`, `isCameraSensorOrientationCompensationEnabled` |
| Screen access | environment / scene bounds | `window?.windowScene?.screen`, `traitCollection.displayScale` |

## 5. Sources

All published by Apple on September 9, 2026 (iPhone Duo announcement day) unless noted:

- News: "Get ready for iPhone Duo" - https://developer.apple.com/news/?id=vn8abkxx
- News: "App Store submissions now open for the latest OS releases" - https://developer.apple.com/news/?id=k1mtkt1k
- Tech Talk: Prepare your app for iPhone Duo - https://developer.apple.com/videos/play/tech-talks/111461/
- Tech Talk: Design for iPhone Duo - https://developer.apple.com/videos/play/tech-talks/111466/
- Tech Talk: Strike a pose with adaptive layouts on iPhone Duo - https://developer.apple.com/videos/play/tech-talks/111463/
- Tech Talk: Leverage multiple displays and scenes on iPhone Duo - https://developer.apple.com/videos/play/tech-talks/111464/
- Tech Talk: Raise the bar with iPhone Duo - https://developer.apple.com/videos/play/tech-talks/111462/
- Tech Talk: Build a great camera experience for iPhone Duo - https://developer.apple.com/videos/play/tech-talks/111465/
- Related: Modernize your UIKit app (WWDC26) - https://developer.apple.com/videos/play/wwdc2026/278
- HIG: Designing for iPhone Duo - https://developer.apple.com/design/human-interface-guidelines/designing-for-iphone-duo
- Apple Newsroom: "Apple unveils iPhone Duo" (Sept 9, 2026) - https://www.apple.com/newsroom/2026/09/apple-unveils-iphone-duo/
- App Store Connect: Screenshot specifications - https://developer.apple.com/help/app-store-connect/reference/screenshot-specifications/
- Compiled research on iPhone Duo hardware/HIG (secondary source, Sept 2026): source of the reported point dimensions (466×678 / 626×890 pt)

> Note: This guide was compiled from session transcripts, session pages, Apple Newsroom, the HIG, and secondary research. Some API spellings were normalized from audio transcription (e.g. `onGeometryChange`, `ArrangementView`; the Xcode tool transcribed as "App Precisability" is "App Resizability"). `ToolbarItemVisibilityPriority` / `UIBarButtonItemVisibilityPriority` and the vertical-bar ordering are confirmed by the HIG; hardware point dimensions still come from secondary reporting. Verify exact API signatures against the iOS 27.1 SDK headers in Xcode 27.1 before relying on them in code review.

## 6. Changelog

- **2026-09-15 (2):** Added verified App Store Connect screenshot specs for iPhone Duo (outer 1398 × 2034 px, inner 2007 × 2853 px; ASC upload support later this year) and refined the Group Labs / Forums Q&A schedule (labs Sept 16-17, Q&As Sept 23).
- **2026-09-15:** Incorporated the full HIG "Designing for iPhone Duo" guidance: games adaptation rules, toolbar grouping (`ToolbarItemGroup` / `UIBarButtonItemGroup`), title+symbol requirement for bar items, control-proximity rule, and the overflow strategy for navigation-focused vs task-oriented apps. Confirmed visibility priority API names against the HIG. Status check: iOS 27.0 and Xcode 27 shipped Sept 14, 2026; iPhone Duo APIs still require the iOS 27.1 SDK (Xcode 27.1). No new iPhone Duo videos or news items since Sept 9.
- **2026-09-10:** Initial version from the six Sept 9 Tech Talks, Developer News, Apple Newsroom, and secondary research.

