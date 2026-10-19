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

- `UIRequiresFullScreen` is **not** an opt-out: per the Xcode 27.1 App Resizability skill, **iOS 27 ignores it and resizes the scene anyway** (see TN3192, "Migrating your app from the deprecated UIRequiresFullScreen key"). The companion key `UIRequiresFullScreenIgnoredStartingWithVersion` decides which releases keep the old behavior; a value of 27 or earlier makes the app fully resizable on iOS 27. Do not delete `UIRequiresFullScreen` (deletion makes the app resizable at once, before layouts are ready) and do not add the companion key without a deliberate decision - remove the key only after running and resizing every screen.

## 2. Toolchain requirements

1. **Xcode 27.1** with the **iOS 27.1 SDK** (edge-to-edge layout, reserved regions, arrangements, hinge, scene accessory, and Duo camera APIs are 27.1 APIs). **Xcode 27.1 final (27A9275) shipped Oct 15, 2026** - recompile and submit now. Note: iPhone Duo ships with iOS 27.1 while other devices already get iOS 27.2 betas; the Duo APIs live in the 27.1 SDK line.
2. Run on the **iPhone Duo simulator in Device Hub**; use the on-screen control buttons to open, close, rotate, and fold the device.
3. Xcode 27.1 ships the updated app-modernization analysis tool **"App Resizability"** (transcribed as "App Precisability" in the Tech Talk audio) - it audits fixed-size dependencies and now supports SwiftUI and iPhone Duo. Run it as an automated first pass: in Xcode, ask the coding assistant to "get my app ready for iPhone Duo"; for other coding agents, export the skill with `xcrun agent skills export`. Verified from the skill bundled with Xcode 27.1 (27A9275): a request about iPhone Duo / "the foldable iPhone" triggers **all five tasks**, run in this order: (1) `UIScreen.main` modernization, (2) interface-orientation modernization, (3) scene lifecycle migration, (4) safe area insets, (5) `userInterfaceIdiom` modernization. It covers Swift and Objective-C and runs three read-only prerequisite checks first (see item 10).
4. Note (App Store), two April 2027 deadlines: apps must be built with the latest SDKs (iOS 27 family), and **all new app/game submissions must include iPhone Duo screenshots**.
5. Timeline: Apple runs iPhone Duo **Group Labs on Sept 16-17, 2026** and **Developer Forums Q&As on Sept 23, 2026** (Photos & Camera, SwiftUI, UIKit sessions); the device **ships Oct 23, 2026** - adaptation should land before then.
6. App Store assets (verified, ASC screenshot specifications): iPhone Duo screenshots are **1398 × 2034 px** for the outer display and **2007 × 2853 px** for the inner display (swap for landscape); up to 10 per device size, .jpeg/.jpg/.png. **Uploads are live in App Store Connect as of Oct 5, 2026**, submissions for iPhone Duo-optimized apps are open now, and ASC has a new preview tool to visualize screenshots, product page headers and metadata on iPhone Duo.
7. Design assets: official **iPhone Duo design kits for Figma and Sketch** are available from Apple Design Resources (https://developer.apple.com/design/resources/) - use them for mockups before coding.
8. In-person help: Apple runs **worldwide iPhone Duo workshops** (https://developer.apple.com/events/view/upcoming-events?topics=iphone-duo&formats=inperson); the developer landing page is https://developer.apple.com/iphone-duo/ and the condensed checklist lives at https://developer.apple.com/iphone-duo/prepare/.
9. Official written guide: **"Preparing your app for iPhone Duo"** (Technology Overviews) is the canonical documentation article - https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo
10. **Prerequisite checks (verified, App Resizability skill)** - an app that fails any of these cannot resize, whatever the code says. Read every source (Info.plist, project build settings, all `.xcconfig` files; the project file wins on conflicts):
    - **Launch screen:** the target must set `UILaunchScreen`, `UILaunchScreens`, `UILaunchStoryboardName`, `UILaunchStoryboards`, or `INFOPLIST_KEY_UILaunchScreen_Generation`. Without one, **iOS 27 rejects the App Store upload with `ITMS-90870`** (TN3208, "Preparing your app's launch screen to meet App Store requirements").
    - **iPad orientations:** the iPad declaration (`UISupportedInterfaceOrientations~ipad`, else `UISupportedInterfaceOrientations`) should list all four orientations (or none); a `supportedInterfaceOrientations` override can still lock the scene. A portrait-only *iPhone* declaration is normal.
    - **Full-screen opt-out:** see the `UIRequiresFullScreen` note in section 1.
11. **Scene lifecycle is mandatory (verified):** as of iOS 27, apps built against the iOS 27 SDK that have not adopted the scene lifecycle (`UIApplicationSceneManifest` + `UIWindowSceneDelegate`) **crash at launch**. Migrate `AppDelegate`-only apps before anything else Duo-related.

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
| Deprecated layout guides *(verified)* | `topLayoutGuide`, `bottomLayoutGuide` | Replaced by `view.safeAreaLayoutGuide` / `view.safeAreaInsets` (careful: `topLayoutGuide.topAnchor` maps to `view.topAnchor`, not the guide) |
| Hardcoded bar heights *(verified)* | literals `20`, `44`, `64`, `88`, `34`, `49`, `83` used as constraint constants / `UIEdgeInsets` to clear a bar or home indicator | Bars move to the side on Duo; constrain to `safeAreaLayoutGuide` instead (confirm the literal is a bar offset, not a font size) |
| Stored / derived insets *(verified)* | safe area insets assigned to properties, `lazy var`, `static let`; insets read in `init`/`viewDidLoad`/`awakeFromNib`/`viewWillAppear`; helpers like `insets(forWidth:)`; `max(left, right)` applied to both sides; `safeAreaInsets.left > 0` threshold checks | Insets change on every pose/bar change; a pre-window read is zero. Read in layout methods only |
| Scroll / keyboard insets by hand *(verified)* | `contentInsetAdjustmentBehavior = .never`; `contentInset` set from `safeAreaInsets`; `adjustedContentInset` + inset; `keyboardWillShow`/`keyboardWillChangeFrame` observers driving constraints | Let the scroll view adjust itself; use `view.keyboardLayoutGuide` |
| Deprecated scene callback *(verified)* | `windowScene(_:didUpdate:interfaceOrientation:traitCollection:)`, `windowScene.coordinateSpace`, `UIWindowScene.interfaceOrientation` | Replace with `windowScene(_:didUpdateEffectiveGeometry:)` + `effectiveGeometry` (iOS 26+) |
| Shared-state "replacements" *(verified)* | `UITraitCollection.current`, `UIApplication.shared.connectedScenes`, `UIDevice.current` used to replace `UIScreen.main` | Same single-display assumption - just renames the bug |
| App-delegate-only lifecycle *(verified)* | no `UIApplicationSceneManifest` in Info.plist | Crashes at launch when built with the iOS 27 SDK |
| SwiftUI safe area misuse *(verified)* | bare `ignoresSafeArea()` on readable/tappable content; bar content in a `ZStack`/`overlay` with its own padding; hardcoded horizontal padding standing in for a side inset | Content ends up under the vertical bar |

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

- Verified rules for the `UIScreen.main` replacement (UIKit `UIScreen` docs + App Resizability skill):
  - Apple's `UIScreen` reference: fetch the screen from the `UIWindowScene` that manages the window, and **avoid using screen objects to make decisions about your app's interface**.
  - **Never** replace it with `UITraitCollection.current`, `UIApplication.shared` (e.g. `connectedScenes`), or `UIDevice.current` - they carry the same single-display assumption. If no local view/trait collection is in scope, add a `traitCollection:` (or `bounds:`) parameter and thread it from the caller; for public/shared API keep the old member as an `@available(*, deprecated, ...)` wrapper that forwards to the new overload.
  - SwiftUI: use `@Environment(\.displayScale)`; for `UIScreen.main.bounds` use `GeometryReader` / the container's geometry. Never add a `traitCollection:` parameter to a SwiftUI view.
  - "Available space" means `view.bounds` (view controller) or `superview.bounds` (view); in a scene delegate with no window yet use `windowScene.effectiveGeometry.coordinateSpace.bounds` (iOS 26+). Never substitute a literal.
  - If a scale/size value is **cached** (stored property, layer property, constraint constant, pre-rendered image), pair the replacement with trait observation in the same change:

```swift
registerForTraitChanges([UITraitDisplayScale.self]) { (self: Self, _) in
    self.updateBorderWidth()
}
// Prefer registerForTraitChanges over overriding traitCollectionDidChange(_:), which is deprecated since iOS 17.
// No registration needed inside layoutSubviews / draw(_:) / updateConstraints / viewIsAppearing(_:)
```

- Orientation (verified): interface orientation describes UI rotation, **not aspect ratio** (a "landscape" scene can be taller than wide), so renaming `statusBarOrientation` to `windowScene.effectiveGeometry.interfaceOrientation` keeps the bug. Replace layout decisions with size classes or a bounds comparison. Migrate the deprecated `windowScene(_:didUpdate:interfaceOrientation:traitCollection:)` to `windowScene(_:didUpdateEffectiveGeometry:)` (iOS 26+): read the **current** value from `windowScene.effectiveGeometry` (the parameter is the *previous* geometry), migrate `windowScene.coordinateSpace` to `effectiveGeometry.coordinateSpace`, and move trait-only work (Dark Mode, Dynamic Type, contrast) to `registerForTraitChanges` on the owning view, because the new callback does not fire for trait-only changes. Use `effectiveGeometry.isInteractivelyResizing` to defer expensive work until a resize drag ends.

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

- Verified LocalAuthentication details: `biometryType` only holds a meaningful value **after** `canEvaluatePolicy(_:error:)` has run at least once on that context; `LABiometryType` cases are `.none`, `.touchID`, `.faceID`, `.opticID`. `NSFaceIDUsageDescription` is required only for Face ID - **Touch ID needs no usage description**. Touch ID shows a system prompt the person can cancel before scanning (Face ID scans immediately), so Face ID-specific warning copy should be hidden on Touch ID devices. For "biometry or Apple Watch" flows there is the policy `.deviceOwnerAuthenticationWithBiometricsOrWatch`.

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
// Also correct: view.bounds.width - view.safeAreaInsets.left - view.safeAreaInsets.right
```

- Verified safe-area rules (App Resizability skill, "Safe Area Insets" task):
  - **Asymmetric horizontal insets are the normal case** where a vertical bar is present: one edge usually carries the whole (much larger than on earlier iPhones) inset, the other carries zero, and which edge changes with orientation and display. Never treat a horizontal inset as a landscape condition, never infer the edge from device/model/orientation/size class, and never gate layout on `inset > 0` - just apply the inset (zero is already correct).
  - `safeAreaInsets` is **physical** (`.left` is always physical left, never "leading"); `safeAreaLayoutGuide.leadingAnchor`/`.trailingAnchor` resolve layout direction. The same applies when writing `additionalSafeAreaInsets` (a `UIEdgeInsets`). `additionalSafeAreaInsets` holds only your *extra* chrome - adding the current safe area into it counts it twice.
  - **Never store an inset.** Read it in `layoutSubviews` / `viewWillLayoutSubviews` / `viewDidLayoutSubviews`, or prefer constraints to `safeAreaLayoutGuide`. Use `safeAreaInsetsDidChange()` / `viewSafeAreaInsetsDidChange()` (call `super`) only to invalidate (`setNeedsLayout()` / `setNeedsUpdateConstraints()`), not to recompute a cached copy.
  - **Read insets from the view you are laying out** - not the window, an ancestor, or `UIApplication.shared`. A **sheet carries its own vertical-bar inset** even where the presenting view does not; sibling split-view columns differ from each other and from the window; two scenes of the same process report different geometry at the same time.
  - Scroll views: leave `contentInsetAdjustmentBehavior` at `.automatic` (remove `.never`), never add safe area on top of `adjustedContentInset`. Keyboard: constrain to `view.keyboardLayoutGuide` instead of converting keyboard-notification frames.
  - Corners: `safeAreaInsets` / `safeAreaLayoutGuide` do **not** account for rounded corners. Where a corner can clip content use `view.layoutGuide(for: .safeArea(cornerAdaptation: .horizontal))` (constraints) or `view.directionalEdgeInsets(for: .safeArea(cornerAdaptation: .horizontal))` (manual layout) - iOS 26+.
  - Leave intentionally edge-to-edge layouts (backgrounds, media players, maps) alone; `viewRespectsSystemMinimumLayoutMargins = false` should be reported, not silently changed.
  - SwiftUI: `EdgeInsets` is already directional; a `GeometryProxy.size` is already inset (don't re-apply reported insets as padding inside the same view). Use `safeAreaBar(edge:)` for bar content (iOS 26; `safeAreaInset(edge:)` below that), `safeAreaInset(edge:)` for other side content, `safeAreaPadding` only for a real design margin. Delete hardcoded padding that only exists to clear a bar. Scope `ignoresSafeArea(edges:)` to the edges you need - a bare `ignoresSafeArea()` also extends under the vertical bar (fine only for backgrounds, artwork, gradients, scrims).

- Match the new screen corner shapes with the Concentricity APIs (introduced iOS 26, updated for Duo):

```swift
// SwiftUI
ConcentricRectangle().fill(.green).padding(8).ignoresSafeArea()
// UIKit
// UICornerConfiguration
```

- Verified (SwiftUI reference): `ConcentricRectangle` computes each corner relative to the container shape (the device corners, or any view that sets `containerShape(_:)`), so no hard-coded radii are needed. When you need the numbers instead of the shape, `GeometryProxy.concentricCornerRadii` / `concentricCornerRadii(in:)` return them (or `nil` if no container shape is set), and `GeometryProxy.containerCornerInsets` returns the container's corner insets, which may include pieces of system UI as well as window/presentation corner radii.

- Extend full-bleed hero/background images under the vertical bars with the **background extension** APIs:

```swift
// SwiftUI
Image("background")
    .ignoresSafeArea()
    .backgroundExtensionEffect()

// UIKit
let bgView = UIBackgroundExtensionView()
bgView.contentView = imageView   // contentView is a settable UIView? property
```

- Verified details: `backgroundExtensionEffect()` (and `backgroundExtensionEffect(isEnabled:)`) mirrors the view into blurred copies on **every edge that has available safe area** and **clips the view** so copies don't overlap; Apple recommends applying it to a single background instance only (visual clarity + performance) and keeping overlaid titles/buttons out of the modified view. `UIBackgroundExtensionView.contentView` is placed **within the safe area by default**; set `automaticallyPlacesContentView = false` (default `true`) to position it yourself with frames/constraints.
- Track size-class changes with trait tracking rather than polling device state: UIKit `registerForTraitChanges([UITraitHorizontalSizeClass.self]) { … }` (iOS 17+; `traitCollectionDidChange(_:)` is deprecated since iOS 17 - keep it only below an iOS 17 deployment target), SwiftUI reacts automatically via the environment.
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

- Verified (UIKit reference): `UITabBarController.Sidebar.preferredPlacement` (`.automatic` / `.sidebar` / `.tabBar`, default `.automatic`) applies where the sidebar and tab bar are **mutually exclusive** (iPhone); `.automatic` resolves to the tab bar on iOS. It is a **preference** the system may override based on space, and it has no effect on iPadOS. Observe `sidebar.isAvailable` / `UITabBarController.Sidebar.Delegate.tabBarController(_:sidebarAvailabilityDidChange:)` to offer alternative navigation when the sidebar disappears (e.g. on the outer display).

- Vertical-bar behavior for toolbar content (from "Raise the bar with iPhone Duo" + HIG "Designing for iPhone Duo"):
  - System bars opt in automatically when using standard toolbar items; top toolbar items map to the top of the vertical bar, bottom items to the bottom; tab bars stay bottom-aligned.
  - **Prefer symbol-only (SF Symbol) toolbar items.** Wide items (text buttons like "Edit", amounts, segmented controls) stay in the horizontal nav/header area instead of moving to the side.
  - Vertical bars have fixed width and flexible height. Top-to-bottom ordering in the vertical bar: (1) primary navigation, Back/Close (SwiftUI `cancellationAction`, UIKit leading-edge items); (2) prominent confirm actions like Done/Save (SwiftUI `topBarPinnedTrailing`, UIKit `pinnedTrailingGroup`); (3) flexible spacer; (4) tool/tab icons by priority; (5) overflow menu.
  - The bar region is shared with system UI (status bar, Dynamic Island expanding vertically with Live Activities). When vertical space runs out, items fold bottom-up into the overflow menu, governed by `ToolbarItemVisibilityPriority` (SwiftUI) / `UIBarButtonItemVisibilityPriority` (UIKit) - give core actions high priority so they collapse last.
  - Group related items with `ToolbarItemGroup` (SwiftUI) / `UIBarButtonItemGroup` (UIKit); the system preserves original groupings and inserts vertical space between top and bottom bar groups.
  - Keep controls near the content they affect (e.g. list controls above the list, not on the far edge), and give every non-text toolbar item both a title and a symbol: the symbol shows in the bar, the title in the overflow menu.
  - Do not override the system's default bar placement; when space is tight, navigation-focused apps should move toolbar actions into the system overflow menu, while task-oriented apps should minimize the tab bar instead.
  - Query which edge the vertical bar is on: SwiftUI `@Environment(\.toolbarVerticalEdge)`, UIKit `traitCollection.verticalBarEdge` (per the "Preparing your app for iPhone Duo" doc; both iOS 27.1). Verified details (App Resizability skill): both resolve to **`leading` / `trailing`** (layout-direction based, unlike physical `safeAreaInsets`). UIKit's `unspecified` is ambiguous (no bar possible *or* edge not resolved yet); SwiftUI's value is `nil` where the system never places a bar and needs no registration. A UIKit read inside a layout method is tracked automatically; elsewhere register in the same change:

```swift
registerForTraitChanges(UITraitCollection.systemTraitsAffectingVerticalBarEdge) { (self: Self, _) in
    self.view.setNeedsLayout()   // in a UIView subclass: self.setNeedsLayout()
}
```

  - Use the edge only to know *where* the bar is; read the safe area inset when you need *how much* space it takes.
  - Pin prominent confirm actions with `ToolbarItem(placement: .topBarPinnedTrailing)` (SwiftUI) or `navigationItem.pinnedTrailingGroup = UIBarButtonItemGroup(barButtonItems: [item], representativeItem: nil)` (UIKit); `axisBehavior` and `visibilityPriority` also exist as `UIBarButtonItem` properties.
  - Verified pinning semantics: SwiftUI `topBarPinnedTrailing` items move to the overflow menu **only when search is active and there isn't enough room**; UIKit `navigationItem.pinnedTrailingGroup` sits on the trailing side of the overflow and search items and **can't move to the overflow menu** - if it holds more than one item, give the group a `representativeItem` so it can't overflow the bar.
  - Verified visibility priorities: SwiftUI `ToolbarItemVisibilityPriority` has `.low`, `.automatic` (default, all platforms) and `.high` (`.low`/`.high` on iOS and macOS), plus relative values via `ToolbarItemVisibilityPriority(higherThan:)` / `(lowerThan:)`; apply with `.visibilityPriority(_:)` on `ToolbarContent` or `CustomizableToolbarContent`. UIKit `UIBarButtonItem.visibilityPriority` (`UIBarButtonItemVisibilityPriority`: `.low`, `.standard` (default), `.high`, plus `init(higherThan:)` / `init(lowerThan:)` / `init(_:)`); an item placed in an implicit group passes its priority to the group. Lower priority moves to the overflow menu first.
  - Verified overflow menu: `ToolbarOverflowMenu { … }` (inside `.toolbar`) and the equivalent view modifier `.toolbarOverflowMenu { … }` place actions in the overflow menu **always**, regardless of toolbar mode, platform, or customizability; on iOS that is the navigation bar's overflow menu.
  - Sheet placement can be steered with `.presentationPlacement(.trailing)` (SwiftUI) / `sheetController.preferredPlacement = .trailing` (UIKit). Verified: `PresentationPlacement` cases are `.automatic` (default), `.center`, `.leading`, `.trailing`, and **only sheet presentations respect it** (apply it to the sheet content, alongside `presentationDetents`).
  - Icon rules from the official doc: vertical presentation requires an icon; the overflow menu requires icon + title; custom toolbar views cannot present vertically (give them an `axisBehavior` or keep them in horizontal bars).
  - Spelling note: the Tech Talk page shows `.toolbarVerticalBehavior(.disabled)` while the official doc shows `.toolbarVerticalBehavior(.never)`; check the SDK for the final enum case.

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
// or, as a view modifier (verified):
ContentView()
    .toolbarOverflowMenu {
        Button("Scan") { … }
    }

// Keep critical items visible when space is tight
.toolbar {
    ToolbarItem { Button(…) }
        .visibilityPriority(.high)
    ToolbarItem { Button(…) }
        .visibilityPriority(ToolbarItemVisibilityPriority(higherThan: .automatic))
}

// UIKit
saveItem.visibilityPriority = .high          // UIBarButtonItemVisibilityPriority: .low / .standard / .high

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

- Only **active** regions are returned by default; query inactive ones (`.includeInactive`) for high-level decisions, e.g. prefer an even number of grid columns whenever a division region exists at all. The official doc also shows `kind: .all`, `options: .all`, and a `layoutDirectionBehavior:` parameter on the SwiftUI query.
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

Verified against the now-published API reference (iOS/iPadOS 27.1+ beta, Mac Catalyst supported):

- SwiftUI: `struct ArrangementView<Primary: View, Secondary: View>`, `init(primary:secondary:)`. Styles: `AutomaticArrangementViewStyle` (the default), `SplitArrangementViewStyle`, `OverlayArrangementViewStyle`, via `arrangementViewStyle(_:)`. Fine-tuning modifiers: `overlayArrangementEdge(_:)`, `splitArrangementLayoutRatio(_:)`, `splitArrangementLayoutSize(minWidth:...)`, `splitArrangementFixedLayoutSize(horizontal:vertical:)`.
- UIKit: `UIArrangementViewController` with `setViewController(_:for:animated:)`, `viewController(for:)`, `placement(for:)`, `state(for:)` returning a `ViewState`, and `updateArrangement(_:animated:)` taking `UISplitArrangement` (default) or `UIOverlayArrangement`; both support `.axes(...)`.

Choosing an arrangement:

- Existing `HStack`/`VStack`-style side-by-side layout, or main–detail content where neither view may be obscured → **split**.
- Existing `ZStack`-style layout, or clear foreground–background relationship (controls over scrollable content) → **overlay** (side-by-side when folded).
- **Do not** put navigation containers (e.g. `NavigationSplitView`) inside an `ArrangementView`, and do not put an `ArrangementView` inside `List`/`ScrollView`.

### Phase 5 - Multitasking, multiple scenes, hinge, scene accessories

- **Split View multitasking:** nothing special to implement beyond correct resizing - handle it with size classes and scene geometry. In split view, vertical bars can appear on **either** side of the app; test both sides (drag the app via the home indicator in the simulator).
- **Multiple scenes:** if the app supports multiple windows on iPad, it does on iPhone Duo too, but **new windows can only be created on the inner display**. Enable multi-scene support via `UIApplicationSceneManifest` in Info.plist (`UIApplicationSupportsMultipleScenes = YES`) if the app doesn't have it yet. Always handle scene-request errors; prefer the window scene activation action API (`UIWindowScene.ActivationAction`), which hides itself automatically when new windows aren't available. Verified (UIKit reference): `UIWindowScene.ActivationAction` is a `UIAction` subclass created with `init(title:subtitle:image:identifier:discoverabilityTitle:attributes:alternate:_:)`; its closure returns a `UIWindowScene.ActivationConfiguration` (built from an `NSUserActivity`, optional `ActivationRequestOptions` and preview), and the **`alternate:` action is what it shows on iPhone and in apps that don't support multiple windows** - so provide a meaningful alternate (e.g. "Open in place") rather than nothing. For direct requests use `UIApplication.shared.activateSceneSession(for:errorHandler:)` with an error handler; `UIWindowScene.ActivationInteraction` (pinch-out) also takes an error-handler closure.
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
// UIKit (verified API): add a UIHingeInteraction to a view
let interaction = UIHingeInteraction { [weak self] _, update in
    guard let hinge = update.hinge else { self?.handleHingeUnavailable(); return }
    self?.apply(angle: hinge.angle, status: hinge.status)
}
view.addInteraction(interaction)
// UIHingeInteraction: init(updateHandler:), isEnabled; UIHinge: angle, status
```

  - `context.hinge` / `update.hinge` is `nil` on devices without a hinge - always guard.
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
  - Verified against the SwiftUI reference in Xcode 27.1: `sceneAccessory(content:)` takes `@ContentBuilder` content conforming to `SceneAccessoryContent` (`@MainActor` protocol; `EmptyView`, `ForEach`, `Group` and `TupleContent` also conform). The **app declares the content, the system decides when and where to present it**, and the app **must remain fully functional without it**. Observe availability with `onAvailabilityChange`, and the accessory content's lifecycle with `onAppear` / `onDisappear` / `scenePhase` (e.g. show extra toolbar controls only while presented). The concrete accessory documented in the built-in reference is `ExternalNonInteractiveAccessory(isEnabled:content:)` - non-interactive content on a connected external display or AirPlay target. `CameraCaptureAccessory` is **not** listed in the built-in reference index (as of 27A9275) - confirm its signature in the SDK headers.

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
- Preview polish: set `AVCaptureVideoPreviewLayer.videoGravity` appropriately and adopt `AVCaptureDevice.dynamicAspectRatio` for correct preview aspect/positioning across poses. Verified (AVFoundation reference): `dynamicAspectRatio` is a **read-only**, key-value observable `AVCaptureDevice.AspectRatio?`; change it only with `setDynamicAspectRatio(_:completionHandler:)` after `lockForConfiguration()` (otherwise `NSGenericException`), using a value from `activeFormat.supportedDynamicAspectRatios` (otherwise `NSInvalidArgumentException`; an empty array means `nil`). The completion handler's timestamp is on the device clock - convert to the session's `synchronizationClock` before comparing with video-data-output buffers. `dynamicDimensions` (KVO) gives the resulting buffer size; `Format.videoFieldOfView(for:geometricDistortionCorrected:)` gives the FOV per aspect ratio.
- Rotation: adopt `AVCaptureDeviceRotationCoordinator` (Swift: `AVCaptureDevice.RotationCoordinator`; apply its `videoRotationAngleForHorizonLevelCapture` / `videoRotationAngleForHorizonLevelPreview`), then disable sensor orientation compensation (`AVCapturePhotoOutput.isCameraSensorOrientationCompensationEnabled = false`; check `isCameraSensorOrientationCompensationSupported` first).
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
| Vertical bars | `.axisBehavior(_)`, `.toolbarVerticalCompressionBehavior(_)`, `ToolbarOverflowMenu` / `.toolbarOverflowMenu { }`, `.visibilityPriority(_)` (`ToolbarItemVisibilityPriority` `.low/.automatic/.high`), `.topBarPinnedTrailing`, `@Environment(\.toolbarVerticalEdge)`, `.toolbarVerticalBehavior(.disabled)` | standard bar controllers adapt automatically; `UIBarButtonItem.visibilityPriority` (`.low/.standard/.high`), `navigationItem.pinnedTrailingGroup`, `traitCollection.verticalBarEdge` + `systemTraitsAffectingVerticalBarEdge`, `preferredVerticalBarBehavior = .never` |
| Safe area / corners | `safeAreaBar(edge:)`, `safeAreaInset(edge:)`, `ignoresSafeArea(edges:)`, `GeometryProxy.containerCornerInsets` / `concentricCornerRadii` | `safeAreaLayoutGuide`, `layoutGuide(for: .safeArea(cornerAdaptation: .horizontal))`, `keyboardLayoutGuide`, `registerForTraitChanges` |
| Background extension | `.backgroundExtensionEffect()` / `(isEnabled:)` | `UIBackgroundExtensionView` (`contentView`, `automaticallyPlacesContentView`) |
| Sheets | `.presentationPlacement(.automatic/.center/.leading/.trailing)` | `sheetController.preferredPlacement` |
| Scene geometry | - | `windowScene(_:didUpdateEffectiveGeometry:)`, `effectiveGeometry.coordinateSpace` / `.isInteractivelyResizing` |
| Biometrics | - | `LAContext.biometryType` (Duo = `.touchID`, side button) |
| Reserved regions | `GeometryProxy.reservedRegions(kind:options:)` | `UIView.reservedRegions(kind:)`, `UIViewReservedRegion` |
| Arrangements | `ArrangementView`, `.arrangementViewStyle(.split/.overlay)`, `\.overlayArrangementZIndex` | `UIArrangementViewController`, `updateArrangement(_)`, `state(for:)` |
| Hinge | `.onHingeChange { previous, context in }` | `UIHingeInteraction` |
| Scenes | - | `UIWindowScene` activation action (auto-hides) |
| Scene accessories | `.sceneAccessory { }` (`SceneAccessoryContent`), `ExternalNonInteractiveAccessory`, `CameraCaptureAccessory` (not in built-in reference index), `.onAvailabilityChange { }` | - |
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
- News: "Build for iPhone Duo with new resources" (Sept 18, 2026) - https://developer.apple.com/news/?id=nyuppv9r
- API reference (iOS 27.1 beta): ArrangementView - https://developer.apple.com/documentation/swiftui/arrangementview ; UIArrangementViewController - https://developer.apple.com/documentation/uikit/uiarrangementviewcontroller ; UIHingeInteraction - https://developer.apple.com/documentation/uikit/uihingeinteraction
- Apple Design Resources (iPhone Duo Figma/Sketch kits) - https://developer.apple.com/design/resources/
- Documentation: "Preparing your app for iPhone Duo" (Technology Overviews) - https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo
- News: "Prepare and submit your apps for iPhone Duo" (Oct 5, 2026) - https://developer.apple.com/news/?id=kkphp5qo
- Checklist: "Three steps to make your app shine on iPhone Duo" - https://developer.apple.com/iphone-duo/prepare/
- Videos: two iPhone Duo Group Lab recordings (~60 min each) in the Meet with Apple collection - https://developer.apple.com/videos/all-videos/
- Compiled research on iPhone Duo hardware/HIG (secondary source, Sept 2026): source of the reported point dimensions (466×678 / 626×890 pt)
- Built-in Apple Developer Documentation in Xcode 27.1 (27A9275), queried Oct 19, 2026: SwiftUI (`ToolbarItemVisibilityPriority`, `ToolbarOverflowMenu`, `toolbarOverflowMenu(content:)`, `topBarPinnedTrailing`, `PresentationPlacement`, `sceneAccessory(content:)`, `SceneAccessoryContent`, `ExternalNonInteractiveAccessory`, `backgroundExtensionEffect`, `ConcentricRectangle`, `GeometryProxy`), UIKit (`UIBarButtonItemVisibilityPriority`, `UINavigationItem.pinnedTrailingGroup`, `UIBackgroundExtensionView`, `UITabBarController.Sidebar`, `UIWindowScene.ActivationAction`, `UIScreen`), AVFoundation (`AVCaptureDevice` dynamic aspect ratio, `RotationCoordinator`), LocalAuthentication (`LABiometryType`, `LAPolicy`)
- Apple's "App Resizability" agent skill bundled with Xcode 27.1 (27A9275): `UIScreen.main`, orientation, scene lifecycle, safe area insets and `userInterfaceIdiom` task references + prerequisite checks
- TN3192: Migrating your app from the deprecated UIRequiresFullScreen key - https://developer.apple.com/documentation/technotes/tn3192-migrating-your-app-from-the-deprecated-uirequiresfullscreen-key
- TN3208: Preparing your app's launch screen to meet App Store requirements - https://developer.apple.com/documentation/technotes/tn3208-preparing-your-apps-launch-screen-to-meet-app-store-requirements

> Note: This guide was compiled from session transcripts, session pages, Apple Newsroom, the HIG, and secondary research. Some API spellings were normalized from audio transcription (e.g. `onGeometryChange`, `ArrangementView`; the Xcode tool transcribed as "App Precisability" is "App Resizability"). `ToolbarItemVisibilityPriority` / `UIBarButtonItemVisibilityPriority` and the vertical-bar ordering are confirmed by the HIG; hardware point dimensions still come from secondary reporting. As of Sept 21, 2026 the API reference is live and `ArrangementView`, `UIArrangementViewController`, and `UIHingeInteraction` are verified against it (all iOS/iPadOS 27.1+ beta); remaining unverified signatures (reserved regions, scene accessories, camera) should still be checked against the Xcode 27.1 beta SDK headers.
>
> Verification status against the documentation built into Xcode 27.1 final (27A9275), Oct 19, 2026: items marked "verified" above come from that reference or from Apple's App Resizability skill shipped in the same build. The following Duo-specific symbols were **not found** in the built-in documentation index and remain verified only via the online API reference or Tech Talks: `ArrangementView` / `UIArrangementViewController`, `UIHingeInteraction` / `onHingeChange`, `reservedRegions(kind:)` / `UIViewReservedRegion`, `CameraCaptureAccessory`, `AVCaptureDeviceDirectionCoordinator` and the inner/outer camera device types, `axisBehavior`, `toolbarVerticalBehavior`, `toolbarVerticalCompressionBehavior`, `preferredVerticalBarBehavior`. `traitCollection.verticalBarEdge` and `\.toolbarVerticalEdge` are confirmed by the App Resizability skill (iOS 27.1) but have no standalone reference page in the built-in index. Check these in the SDK headers before shipping.

## 6. Changelog

- **2026-10-19 (2):** Compared the guide against the documentation and the App Resizability skill built into Xcode 27.1 (27A9275) and added the verified facts. **Correction:** iOS 27 *ignores* `UIRequiresFullScreen` (TN3192, `UIRequiresFullScreenIgnoredStartingWithVersion`). New: scene lifecycle is mandatory with the iOS 27 SDK (otherwise crash at launch); launch screen required (`ITMS-90870`, TN3208); App Resizability's 5 tasks + prerequisite checks; extended Phase 0 audit table; `UIScreen.main` replacement rules (no `UITraitCollection.current`, `registerForTraitChanges([UITraitDisplayScale.self])`); orientation ≠ aspect ratio and the `didUpdateEffectiveGeometry` migration; detailed safe-area rules (physical vs directional insets, never store insets, sheets carry their own bar inset, corner adaptation, `keyboardLayoutGuide`, SwiftUI `safeAreaBar` / scoped `ignoresSafeArea`); `verticalBarEdge` values + `systemTraitsAffectingVerticalBarEdge`; visibility priority levels; pinned-item overflow semantics; `toolbarOverflowMenu` modifier; `PresentationPlacement` cases; sidebar placement semantics; `backgroundExtensionEffect` / `UIBackgroundExtensionView` details (fixed `contentView` usage); `GeometryProxy.containerCornerInsets` / `concentricCornerRadii`; `ActivationAction` `alternate:`; `sceneAccessory` contract and `ExternalNonInteractiveAccessory`; `setDynamicAspectRatio` requirements; LocalAuthentication details. Replaced the deprecated `traitCollectionDidChange` advice. Added a list of Duo symbols not present in the built-in index.
- **2026-10-19:** Xcode 27.1 final (27A9275) shipped Oct 15. No new iPhone Duo docs or videos since Oct 5; iOS 27.1 itself ships with the device on Oct 23.
- **2026-10-07:** Major pre-launch update. Xcode 27.1 RC (27A9275, Oct 5); App Store submissions for iPhone Duo-optimized apps are open and ASC screenshot uploads are live (10 per device size, with a new ASC preview tool); second April 2027 deadline added (iPhone Duo screenshots mandatory in submissions). Incorporated Apple's new official written guide "Preparing your app for iPhone Duo" (Technology Overviews): background extension APIs (`backgroundExtensionEffect` / `UIBackgroundExtensionView`), vertical-bar edge queries (`\.toolbarVerticalEdge` / `traitCollection.verticalBarEdge`), `topBarPinnedTrailing` / `pinnedTrailingGroup` placements, sheet `presentationPlacement(.trailing)`, icon/title rules per presentation, reserved-region `.all` queries, and trait-tracking guidance. Added the /iphone-duo/prepare/ checklist page, `xcrun agent skills export` for the App Resizability skill, and two published Group Lab recordings (~60 min each) in the video library.
- **2026-09-21:** Xcode 27.1 beta (27A9269) released Sept 18 with the iPhone Duo SDK and simulator; official Figma/Sketch design kits published in Apple Design Resources; worldwide workshops announced ("Build for iPhone Duo with new resources", Sept 18). API reference is now live: verified `ArrangementView` (init(primary:secondary:), automatic/split/overlay styles, layout-ratio and edge modifiers), `UIArrangementViewController` (updateArrangement(_:animated:), ViewPlacement, ViewState, Mac Catalyst support) and `UIHingeInteraction` (init(updateHandler:), UIHinge.angle/status). All are marked iOS/iPadOS 27.1+ beta. Also noted: iPhone Duo ships with iOS 27.1 while iOS 27.2 betas roll out for other devices.
- **2026-09-15 (2):** Added verified App Store Connect screenshot specs for iPhone Duo (outer 1398 × 2034 px, inner 2007 × 2853 px; ASC upload support later this year) and refined the Group Labs / Forums Q&A schedule (labs Sept 16-17, Q&As Sept 23).
- **2026-09-15:** Incorporated the full HIG "Designing for iPhone Duo" guidance: games adaptation rules, toolbar grouping (`ToolbarItemGroup` / `UIBarButtonItemGroup`), title+symbol requirement for bar items, control-proximity rule, and the overflow strategy for navigation-focused vs task-oriented apps. Confirmed visibility priority API names against the HIG. Status check: iOS 27.0 and Xcode 27 shipped Sept 14, 2026; iPhone Duo APIs still require the iOS 27.1 SDK (Xcode 27.1). No new iPhone Duo videos or news items since Sept 9.
- **2026-09-10:** Initial version from the six Sept 9 Tech Talks, Developer News, Apple Newsroom, and secondary research.

