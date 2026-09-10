# Preparing Your App for iPhone Duo

A developer guide to bringing an iOS app to **iPhone Duo**, Apple's first multi-display, folding iPhone.

This guide is compiled from Apple's six iPhone Duo tech talks (September 2026). Their English transcripts are in [`subtitles/`](subtitles/). API names come from the official code snippets on each talk's page. Names marked **†** appear only in the narration, so check their exact spelling in Xcode 27.1 documentation before relying on them.

| # | Talk | Focus | Transcript |
|---|------|-------|------------|
| 111466 | [Design for iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111466/) | Design principles, poses, control placement | [vtt](subtitles/111466-design-for-iphone-duo.en.vtt) |
| 111461 | [Prepare your app for iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111461/) | SDK, Xcode, size classes, safe areas | [vtt](subtitles/111461-prepare-your-app-for-iphone-duo.en.vtt) |
| 111462 | [Raise the bar with iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111462/) | Vertical navigation/tool/tab bars | [vtt](subtitles/111462-raise-the-bar-with-iphone-duo.en.vtt) |
| 111463 | [Strike a pose with adaptive layouts on iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111463/) | The fold, reserved regions, arrangements | [vtt](subtitles/111463-strike-a-pose-with-adaptive-layouts-on-iphone-duo.en.vtt) |
| 111464 | [Leverage multiple displays and scenes on iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111464/) | Hinge input, multitasking, scene accessories | [vtt](subtitles/111464-leverage-multiple-displays-and-scenes-on-iphone-duo.en.vtt) |
| 111465 | [Build a great camera experience for iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111465/) | Front cameras, direction coordinator, preview | [vtt](subtitles/111465-build-a-great-camera-experience-for-iphone-duo.en.vtt) |

---

## Contents

1. [TL;DR](#1-tldr)
2. [The device](#2-the-device)
3. [What your app gets, by SDK](#3-what-your-app-gets-by-sdk)
4. [Tooling: Xcode 27.1, Device Hub, App Resizability](#4-tooling-xcode-271-device-hub-app-resizability)
5. [Flexible layout fundamentals](#5-flexible-layout-fundamentals)
6. [Safe areas and layout margins](#6-safe-areas-and-layout-margins)
7. [Navigation, tabs, sheets and presentations](#7-navigation-tabs-sheets-and-presentations)
8. [Vertical bars](#8-vertical-bars)
9. [Designing around the fold](#9-designing-around-the-fold)
10. [Reserved regions API](#10-reserved-regions-api)
11. [Arrangements](#11-arrangements)
12. [The hinge as an input](#12-the-hinge-as-an-input)
13. [Multitasking and multiple scenes](#13-multitasking-and-multiple-scenes)
14. [Scene accessories](#14-scene-accessories)
15. [Camera apps](#15-camera-apps)
16. [Release checklist](#16-release-checklist)
17. [API quick reference](#17-api-quick-reference)
18. [Glossary](#18-glossary)
19. [Further reading](#19-further-reading)

---

## 1. TL;DR

1. **Build with Xcode 27.1 and the iOS 27.1 SDK.** Your app only goes edge to edge and gets vertical bars once it's rebuilt.
2. **Test in the iPhone Duo simulator in Device Hub.** Open, close, rotate and fold it, and try Split View on both sides.
3. **Make the app freely resizable.** Base layout on size classes, never on idiom, interface orientation, fixed widths or `UIScreen.main`.
4. **Treat safe areas and layout margins as asymmetric.** Controls now sit along the side of the screen.
5. **Use the system containers and bars:** `NavigationStack`/`NavigationSplitView`/`TabView`, or `UINavigationController`/`UISplitViewController`/`UITabBarController`. They adapt to every pose, move bars to the side and avoid the fold automatically.
6. **Audit your toolbars.** Give every item a title and a symbol, order items correctly, set `axisBehavior` on custom views, set `visibilityPriority`, and decide the compression behavior.
7. **Handle the fold.** Rely on system components. Use `reservedRegions(kind:)` for custom controls, and `ArrangementView` / `UIArrangementViewController` for two-pane layouts.
8. **Support Split View multitasking and multiple windows.** New windows can't be created on the outer display.
9. **Camera apps:** use the Virtual Front Camera, or adopt `AVCaptureDeviceDirectionCoordinator` for full control. Consider a `CameraCaptureAccessory` on the outer display.
10. **Run the App Resizability agent skill** in Xcode 27.1.

---

## 2. The device

### Hardware

- **Outer display.** It's wider and shorter than a traditional iPhone. The status bar (redesigned) and the front camera sit on the **right side**, and so do app controls, where the right thumb reaches them. Content gets an uninterrupted area to their left, about the size of the content area on a traditional iPhone. The Dynamic Island grows **vertically** as Live Activities arrive.
- **Inner display.** It's the largest display ever on an iPhone. It has a **hinge/fold** through the middle and an **under-display FaceTime camera**, the first on iPhone.
- **Cameras.** There are **two front cameras**: an *outer ultra wide* and an *inner ultra wide* (under-display). Both have square sensors and an ultra-wide field of view. The rear cameras can also face the user when the open device is flipped. See [§15](#15-camera-apps).

### Poses

People hold, fold and set iPhone Duo down in many ways, and your app has to work in every pose. That doesn't mean building a separate layout for each one (see [§5](#5-flexible-layout-fundamentals)).

| Pose | Display | Bars | Fold |
|------|---------|------|------|
| Closed, portrait | Outer | Vertical, along the right side | — |
| Closed, landscape ("tent", standing on its edges) | Outer | Vertical | — |
| Open, landscape | Inner | Vertical, along the side | Flat: fold region is *inactive* |
| Open, portrait | Inner | **Horizontal** (familiar top/bottom bars). This is the only pose that keeps horizontal bars. | Flat: inactive |
| Partially folded "like a book" | Inner | Vertical | *Active*: splits the screen into left/right regions |
| Propped on a table "like a laptop", inner display facing you | Inner | Horizontal | *Active*: splits the screen into top/bottom regions |
| Split View (two apps, 50/50) | Inner | Vertical, on each app's **outer** edge | — |
| Picture in Picture pinned to the top | Inner | — | Folding expands the video to half the screen |

### Size classes

| Display | Horizontal | Vertical |
|---------|-----------|----------|
| Outer, portrait | Compact | Regular |
| Outer, landscape | Compact | Compact |
| Inner | Regular | Regular |

Even with regular/regular size classes, **it's still an iPhone app**, so don't branch on the interface idiom. Split View and a pinned PiP change your app's size too, so always read the current traits instead of assuming a value per display.

**Sources:** [Design › Design principles](https://developer.apple.com/videos/play/tech-talks/111466/?time=28) · [Prepare › Use size classes](https://developer.apple.com/videos/play/tech-talks/111461/?time=166)

---

## 3. What your app gets, by SDK

Your app runs on iPhone Duo **without being rebuilt**, but it only uses the whole screen once it's built with the latest SDK.

| Built with | Behavior on iPhone Duo |
|------------|------------------------|
| An SDK older than iOS 27 | Works. **Closed:** uses the area to the left of the status bar and camera. **Open:** runs at a familiar iPhone size and aspect ratio. |
| iOS 27 SDK (app-resizing support) | Your resizing work pays off: on the inner display the app extends up to the left edge of the status bar area. |
| **iOS 27.1 SDK** (Xcode 27.1) | The app extends **to the edges of the screen**. Standard navigation and toolbar buttons lay out **vertically under the status bar**. You need this SDK for vertical bars and the new iPhone Duo APIs. |

If you already support iPhone app resizing (iOS 27 lets people freely resize iPhone apps through **iPhone Mirroring on the Mac**), most of the groundwork is done. Opening and closing iPhone Duo is just another resize.

**Sources:** [Prepare › Build with the latest SDK](https://developer.apple.com/videos/play/tech-talks/111461/?time=30)

---

## 4. Tooling: Xcode 27.1, Device Hub, App Resizability

- **Install Xcode 27.1.**
- **Simulate in Device Hub.** Choose the **iPhone Duo simulator**. Buttons at the bottom of the window **open, close, rotate and fold** the device.
- **Test Split View multitasking.** Show the inner display, drag your app by the **home indicator** to one side of the screen until a drop area appears, then drag it to the other side. Vertical controls can appear on either edge of your app, so check both.
- **Run the App Resizability agent skill.** Xcode 27.1 renames the "app modernization" skill introduced in *Modernize your UIKit app* (WWDC26) to **App Resizability** and extends it to **SwiftUI and iPhone Duo**. It's the quickest way to check your app against the layout practices in this guide.

**Sources:** [Prepare › Get started in Xcode](https://developer.apple.com/videos/play/tech-talks/111461/?time=77) · [Prepare › Next steps](https://developer.apple.com/videos/play/tech-talks/111461/?time=552)

---

## 5. Flexible layout fundamentals

iPhone apps have kept adapting to new screen sizes, screen shapes and hardware such as the Dynamic Island. iPhone Duo adds another screen shape, a new camera location and live resizing whenever the device opens, closes, rotates or folds.

### Design for two size classes, not for poses

- Target **compact width** (outer display) and **regular width** (inner display). Avoid **fixed widths, breakpoints and any metric tied to a specific screen**.
- Build layouts from **layout margins** and **horizontal safe-area insets**. If you already do, your app mostly adapts on its own.
- In short: make the app **freely resizable**.

```swift
// SwiftUI
@Environment(\.horizontalSizeClass) private var horizontalSizeClass
@Environment(\.verticalSizeClass) private var verticalSizeClass

// UIKit
traitCollection.horizontalSizeClass
traitCollection.verticalSizeClass
```

### Don't use idiom or interface orientation for layout

- **Outer display:** interface orientation behaves like any other iPhone. Consider **supporting landscape**, because people may stand the phone up like a tent.
- **Inner display:** it **ignores your supported interface orientations**. As with the idiom, never make layout decisions from the interface orientation. Use size classes.

### Stop referencing the main screen

On a device with two displays, "the main screen" is ambiguous, and `UIScreen.main` **will be deprecated in a future release**. Prefer local context: the SwiftUI environment, the trait collection, or the scene's bounds. If you really need the screen, get it from the window scene:

```swift
// Before
let screenScale = UIScreen.main.scale
// After
let screenScale = traitCollection.displayScale

// When you truly need the screen object
let screen = window?.windowScene?.screen
```

### Match the screen corners

The iOS 26 **concentricity APIs** are updated for iPhone Duo's screen shapes. Use `ConcentricRectangle` in SwiftUI and `UICornerConfiguration` in UIKit.

```swift
ConcentricRectangle()
    .fill(Color.green)
    .padding(8.0)
    .ignoresSafeArea()
```

### `UIRequiresFullScreen`

iPhone Duo still honors `UIRequiresFullScreen` and respects your supported orientations. **Your app still resizes** when the device opens or closes, and it is **scaled** on the inner display, including in Split View.

**Sources:** [Prepare › Adopt flexible layouts](https://developer.apple.com/videos/play/tech-talks/111461/?time=93) · [Prepare › Avoid screen assumptions](https://developer.apple.com/videos/play/tech-talks/111461/?time=237) · [Prepare › Concentricity](https://developer.apple.com/videos/play/tech-talks/111461/?time=270) · [Design › Adapting your design](https://developer.apple.com/videos/play/tech-talks/111466/?time=222)

---

## 6. Safe areas and layout margins

### The rules

- **System bars lay out outside the safe area.** They automatically avoid the status bar and the camera. Horizontal bars add **top/bottom** insets and vertical bars add **leading/trailing** insets.
- **Foreground content (anything interactive) goes inside the safe area.** SwiftUI does this by default. In UIKit, use `safeAreaInsets` or constrain to `safeAreaLayoutGuide`.
- **Background content (full-bleed art) may extend past it,** behind toolbars and sidebars. Use `.ignoresSafeArea()` in SwiftUI or `view.bounds` in UIKit.
- **Insets are often asymmetric,** especially on iPhone Duo. Vertical buttons can sit on the **left** in landscape and in Split View. **Layout margins are asymmetric too**, so foreground content can come closer to the vertical controls and status bar while keeping the margin on the other side.

```swift
// UIKit: foreground in the safe area, background full bleed
foreground.frame = view.bounds.inset(by: view.safeAreaInsets)
backgroundView.frame = view.bounds

// ❌ Assumes opposite insets are equal
let width = view.bounds.width - view.safeAreaInsets.left * 2
// ✅ Handles each side independently
let width = view.bounds.inset(by: view.safeAreaInsets).width
```

### Three ways to place content on the outer display

With controls on the right, most layouts become asymmetric.

1. **Offset (the default for most content).** Content shifts so the controls don't cover it. Aligning to horizontal safe-area insets does this for you.
2. **Center on the full display.** This suits immersive, highly visual screens that **don't scroll**, provided no interactive element can end up under the controls.
3. **Mix the two.** Use a full-width background or header with inset, scrollable foreground content. Keep every interactive element inside the inset scrolling area.

### Custom UI outside the safe area

iOS 27.1 adds **reserved regions**, which let custom UI use as much space as possible without colliding with system UI. They're the right tool for **custom bars** and **edge-to-edge UI**, and for adapting to the fold. See [§10](#10-reserved-regions-api).

**Sources:** [Prepare › Respect safe areas](https://developer.apple.com/videos/play/tech-talks/111461/?time=366) · [Prepare › Asymmetric insets](https://developer.apple.com/videos/play/tech-talks/111461/?time=450) · [Design › Designing for the outer display](https://developer.apple.com/videos/play/tech-talks/111466/?time=393)

---

## 7. Navigation, tabs, sheets and presentations

Standard navigation patterns adapt to every pose for free.

### Split views

`NavigationSplitView` / `UISplitViewController`:

- **Closed:** collapses to single-stack navigation.
- **Open:** columns appear **tiled** or as **overlays**.
- **Partially folded:** the system keeps both columns visible in an even **50/50 split** on either side of the fold.

### Tab bars and sidebars

`TabView` / `UITabBarController` adapt to all poses. By default, tabs appear on both displays and lay out vertically where appropriate. On the inner display you can opt into a **sidebar**, which works best for **information-dense apps**, as in the Health example.

```swift
// SwiftUI
TabView { … }
    .defaultTabBarPlacement(.sidebar)

// UIKit
tabBarController.sidebar.preferredPlacement = .sidebar
```

### Making the inner display more than a stretched iPhone app

- **Split views** show several levels of your hierarchy at once.
- **Reflowing content:** a vertical stack that becomes **two columns** when there's more width, as in the Music example.
- **Tab bar as a sidebar** (above).

Keep the **hierarchy identical inside and out**, and never limit functionality to one pose. People open and close the device often, so the app has to feel predictable and consistent.

### Sheets

| Situation | Behavior |
|-----------|----------|
| Outer display (default) | If the sheet has a toolbar, its controls move to the **side** (vertical bar). |
| Outer display, vertical bar disabled | The sheet stops just short of the front camera and the status bar repositions itself. Good for sheets with **a single toolbar button**. |
| Inner display, landscape or portrait | **Centered**, with standard **horizontal** bars. |
| Inner display, moved with the `preferredPlacement` API† | Placed **left**: no vertical bar. Placed **right**: gets a vertical bar. |
| Partially folded | Sheets **slide over** so they don't rest in the fold. |

### Other presentations

Popovers, context menus, alerts and action sheets adapt to each pose. The system repositions these lightweight contextual views around reserved regions such as the fold so they stay fully visible.

**Sources:** [Prepare › Adopt standard navigation](https://developer.apple.com/videos/play/tech-talks/111461/?time=301) · [Prepare › Sidebar](https://developer.apple.com/videos/play/tech-talks/111461/?time=344) · [Design › Designing for the inner display](https://developer.apple.com/videos/play/tech-talks/111466/?time=454) · [Design › Sheet behavior](https://developer.apple.com/videos/play/tech-talks/111466/?time=516)

---

## 8. Vertical bars

The wider aspect ratio gives apps more horizontal room, so controls normally at the **top and bottom move to the side**. That frees vertical space for content and puts controls within reach. They're the **same components**, just laid out on another axis. Vertical bars appear on the outer display and on the inner display in landscape. The inner display in portrait keeps horizontal bars.

### 8.1 Opting in

1. **Rebuild with the latest SDK** (iOS 27.1).
2. **Use bars owned by navigation containers.**
   - SwiftUI: attach `.toolbar` to content inside `NavigationStack` / `NavigationSplitView` (or `TabView`).
   - UIKit: use `UINavigationController` and `UITabBarController`, which manage their own bars. Set items on the view controller (`navigationItem`, toolbar items) and put it inside a navigation controller.
   - Content in a **custom `UIToolbar`, `UINavigationBar` or `UITabBar` you build yourself is ignored** and won't move to the vertical bar.

```swift
NavigationStack {
    ContentView()
        .toolbar {
            ToolbarItem(placement: .bottomBar) { … }
        }
}
```

### 8.2 The shared bar region

Navigation, toolbar and tab bar controls all live in **one shared vertical region**. Picture your top and bottom bars rotated 90° into a single column. Depending on the screen it might contain only a toolbar (Notes), only a tab bar (Clock), or both (Fitness). The status bar, the Dynamic Island and Live Activities share this space too.

**Mapping from your current layout:**

| Today (horizontal) | On iPhone Duo (vertical) |
|--------------------|--------------------------|
| Top toolbar / navigation bar items | Top of the vertical region |
| Bottom toolbar items | Bottom of the vertical region |
| Tab bar | Stays **bottom-aligned** |
| Separation between top and bottom groups | A **vertical spacer**, even though they're now one bar |
| Items too wide for the column (text buttons, segmented controls) | **Stay horizontal** in the navigation bar |

**Participation rules:**

- Toolbar items move to the vertical bar **only when their container sits along the display edge**.
- **Split views:** only the **detail column** participates. Items in other columns stay horizontal.
- **Inspectors** don't get their own vertical bar when expanded, since the detail column already has one.
- **Sheets:** see [§7](#sheets).
- **Right-to-left languages:** the bar is tied to the hardware, so it stays on the **same physical side**. Content adapts around it.
- **Keyboard accessory bars** stay attached to the keyboard and never move to the vertical axis.
- Keep controls with **their container**. Don't flip everything to the other axis yourself.

### 8.3 Ordering

A vertical bar reads top to bottom:

1. **Primary navigation** (Back, Close) at the top. Navigation controllers add Back automatically.
2. **Prominent actions** (Done) right after.
3. **Everything else** keeps its original grouping.

```swift
// Custom back/close button
// SwiftUI
.toolbar { ToolbarItem(placement: .cancellationAction) { … } }
// UIKit: a leading item that replaces (not supplements) Back
navigationItem.leftItemsSupplementBackButton = false   // default
navigationItem.leadingItemGroups = [UIBarButtonItemGroup(…)]

// Prominent action
// SwiftUI
.toolbar { ToolbarItem(placement: .topBarPinnedTrailing) { … } }
// UIKit
navigationItem.pinnedTrailingGroup = UIBarButtonItemGroup(…)
```

Not every pose shows vertical bars, so **keep control placement consistent** and people won't have to relearn where actions are.

### 8.4 Preparing toolbar content

Horizontal bars have a fixed height and flexible item width. Vertical bars are the opposite: **fixed width, flexible height**. **Symbol-only items** suit them best.

- **Always provide both a title and an image** (SwiftUI `Label`, or `UIBarButtonItem`'s title and image) and let the system pick the representation. Symbol-only items still need a title, because it's shown in the overflow menu and in expanded forms.
- **Default placement is inferred from content.** Items with an icon (Back, Share) can go vertical. Text-only items (Edit) stay horizontal.

**Overriding the axis with `axisBehavior`:**

| Case | What to do |
|------|------------|
| An item that switches between a symbol and text (for example Select ⇄ Done) | `.horizontalOnly`, so related items stay on one axis. The **system Edit button** already does this. |
| A UIKit custom view or a complex SwiftUI view | Stays horizontal by default. If it has a vertical representation (a compass, a profile button), use `.verticalPreferred`. |

```swift
// SwiftUI
.toolbar {
    ToolbarItem { CompassView() }
        .axisBehavior(.verticalPreferred)
    ToolbarItem { SelectOrDoneButton() }
        .axisBehavior(.horizontalOnly)
}

// UIKit
let item = UIBarButtonItem(customView: CompassView())
item.axisBehavior = .verticalPreferred
selectItem.axisBehavior = .horizontalOnly
```

**Reduce text in bars:**

- Cut down **title-only** items and custom views that show **both text and an image**. Most can become symbol-only.
- Show counts as a **badge**, not as inline text (the badge API is from iOS 26):

```swift
// SwiftUI
InboxButton().badge(7)
// UIKit
item.badge = .count(7)
```

- The test: if the text only **reinforces** the symbol, drop it. If it carries **standalone information**, such as a cart button that shows a price, keep the control in the horizontal bar.

### 8.5 Adapting custom views

- When you opt a custom view into the vertical bar, it must **fit the bar's fixed width** or have a **vertical layout**. Adjust metrics as needed. In the talk's example, an action panel hides its titles and gets shorter when vertical.
- To tell whether a vertical bar is present, read the vertical bar edge. It's populated when items can be on the vertical axis and `nil`/unspecified when they can't. You can read it in your content view or in the item's custom view.

```swift
// SwiftUI
@Environment(\.toolbarVerticalEdge) var edge
// UIKit
switch traitCollection.verticalBarEdge { … }
```

- **Appearance:** like horizontal bars, a vertical bar has **no scroll-edge effect** by default, but it gets a **background when Reduce Transparency is on**. Make sure custom content stays legible either way.
- **Spacing:** **flexible spacers have zero size** on the vertical axis, while **fixed spacers keep their minimum size**. Don't add extra spacing in either orientation. If you haven't moved to the new bar design (grouping items along the leading and trailing edges; see *Get to know the new design system*, WWDC25), now is the time.

### 8.6 Managing overflow

Items overflow more often on the **outer display in landscape**, where there's less vertical space. It also happens when other UI competes for room, such as the **keyboard**, **Picture in Picture in open portrait**, Live Activities or the status bar.

**1. Decide what compresses first: the toolbar or the tab bar.**

| Experience | Compresses first | Example |
|------------|------------------|---------|
| Navigation-focused (**default**) | Toolbar, so top-level destinations stay reachable | Podcasts |
| Task-oriented | Tab bar, so frequent actions stay visible | Games |

```swift
// SwiftUI: set per view (for example, inside a Tab)
ContentView()
    .toolbarVerticalCompressionBehavior(.prefersToolbarItems)
// UIKit
navigationItem.verticalBarCompressionBehavior = .prefersBarItems
```

**2. Merge any custom overflow into the single system overflow menu.**

```swift
// SwiftUI
.toolbar {
    ToolbarOverflowMenu {
        Button("Scan") { … }
        Button("Connect") { … }
    }
}
// UIKit
navigationItem.additionalOverflowItems = UIDeferredMenuElement { provider in
    provider(self.persistentOverflowItems())
}
```

- The **ellipsis** is the standard overflow symbol on iPhone. Use it only for overflow, not symbols borrowed from other platforms, and give other menus a different symbol.
- Not every existing menu should become overflow.

**3. Set visibility priorities.** By default items overflow **from bottom to top**. Assign `high`, `low` or a custom priority, first **per group** and then within a group.

```swift
// SwiftUI
ToolbarItem { Button(…) { … } }
    .visibilityPriority(.high)
// UIKit
item.visibilityPriority = .high
```

Keep **frequent actions** (Compose in Mail, New Note in Notes) and **status items** (anything with a badge) visible the longest.

### 8.7 When to opt out of vertical bars

Most apps should adopt vertical bars. Consider opting out for:

- **Single-page apps with bottom-heavy layouts** (such as Calculator), where horizontal bars let the content expand fully.
- **Control-heavy sheets with a single bar item** (such as a lone Close button), where a vertical bar would only take space away.

```swift
// SwiftUI
NavigationStack {
    ContentView()
        .toolbarVerticalBehavior(.disabled)
}

// UIKit
override var preferredVerticalBarBehavior: UIVerticalBarBehavior { .disabled }
```

**Sources:** [Raise the bar › Opt in](https://developer.apple.com/videos/play/tech-talks/111462/?time=120) · [Shared bar region](https://developer.apple.com/videos/play/tech-talks/111462/?time=189) · [Ordering](https://developer.apple.com/videos/play/tech-talks/111462/?time=269) · [Toolbar content](https://developer.apple.com/videos/play/tech-talks/111462/?time=356) · [Axis](https://developer.apple.com/videos/play/tech-talks/111462/?time=480) · [Custom views](https://developer.apple.com/videos/play/tech-talks/111462/?time=607) · [Overflow](https://developer.apple.com/videos/play/tech-talks/111462/?time=700) · [Opt out](https://developer.apple.com/videos/play/tech-talks/111462/?time=861) · [Design › Positioning controls](https://developer.apple.com/videos/play/tech-talks/111466/?time=307)

---

## 9. Designing around the fold

When iPhone Duo is partially folded, the display curves through the center. Like a photo across a book's spine, content in that area is hard to see and controls are hard to tap. The fold becomes a **natural divider** in your layout.

### What the system already does

- **Book pose:** text and images shift away from the curve, and buttons and interactive elements move toward the sides.
- **Laptop/tabletop pose:** interactive elements move to the **bottom half**, where they're easier to reach and the device stays stable.
- **Fold avoidance is built into system components,** including sheets, alerts, menus, toolbar buttons and more. Use them wherever you can.
- Split views re-balance to **50/50** around the fold.

### What *not* to move

**Continuous scrolling content** (articles, feeds, documents, lists) should **not** avoid or be displaced from the fold. Scrolling already lets people move it, and splitting it between regions would break the flow.

### Displacement: adapting your own UI

*Displacement* changes the frame of existing elements based on the space available, keeping them **visible, reachable and unobstructed**.

- **Scope.** It can apply to a button, a container or a whole section of the layout. Elements that adapt independently move alone. Elements that work together move **together**.
- **Don't move too far.** Moving an element far from its source weakens the connection. In the photo example, the selected photo and its context menu move together and align on either side of the fold, instead of the menu centering itself in the trailing region.
- **Purpose decides where things go, and the pose matters:**
  - *Book pose:* alerts move to the **trailing** side, close to where they'll be when the device closes and the experience continues on the outer display.
  - *Tabletop pose:* the **top region** suits content that should be visible from a distance, such as alerts or media. The **bottom region** suits touch, such as media controls, because it's a stable surface.
  - When several regions would work, **stay contextual**. A focused search field stays over the view it searches, adjusting its width and position as the device folds.
- **Adapt more than position.** Size, spacing and other visual properties can change too. In the Fitness grid example, the outer margins stay the same while spacing around the hinge grows, so every tile stays in one region and remains tappable.
- **Never remove things.** Displacement only moves, resizes or reorganizes. Every pose keeps the full content and functionality.

### Optional: a dedicated tabletop layout

The tabletop pose suits hands-free use: media at the top, tappable controls on the stable bottom half. If you build a layout for it, keep **the same controls and general hierarchy** as in other poses. Never make a feature available in only one pose.

### The inner FaceTime camera

On the inner display, your UI may also need to make room for the under-display camera **while it's active**. If its viewfinder is central to your experience, keep important content and controls away from that spot.

### Where to start

1. Audit **centered layouts**. Should each become a two-column layout, or which displacement pattern fits?
2. Use **system containers and presentations** for the free behavior.
3. Replace custom horizontal-split or overlay layouts with **arrangements** ([§11](#11-arrangements)).
4. Find your **highest-priority manually laid-out controls** and use **reserved regions** ([§10](#10-reserved-regions-api)) to displace them where needed.

**Sources:** [Design › Fold avoidance](https://developer.apple.com/videos/play/tech-talks/111466/?time=568) · [Strike a pose › Designing around the hinge](https://developer.apple.com/videos/play/tech-talks/111463/?time=89) · [Displacement patterns](https://developer.apple.com/videos/play/tech-talks/111463/?time=146) · [Choose where content moves](https://developer.apple.com/videos/play/tech-talks/111463/?time=240) · [Next steps](https://developer.apple.com/videos/play/tech-talks/111463/?time=994)

---

## 10. Reserved regions API

*New in iOS 27.1.* A **reserved region** is an area of your view shaped by hardware features: the hinge and the two cameras. Adapt to reserved regions as you would to any other system area, such as iPadOS window controls. The types are `ReservedRegion`† (SwiftUI) and `UIViewReservedRegion`† (UIKit).

### Kinds

| Kind | Meaning | On iPhone Duo | Active when… |
|------|---------|---------------|--------------|
| `.division` | Divides an area into several smaller usable areas | The **fold** | The device is **partially folded**. When flat it's inactive and **zero width**. |
| `.occlusion` | Covers part of an area, like a smaller frame inside your bounds | The inner **FaceTime camera** | The camera is **in use** |

The outer camera is always present. System toolbars and tab bars lay out vertically within the new safe area around it.

### Querying

By default only **active** regions are returned. Pass `.includeInactive` to get the others too. Use each region's `frame` in your own layout.

```swift
// SwiftUI: from a GeometryReader (or the onGeometryChange modifier)
GeometryReader { proxy in
    let folds = proxy.reservedRegions(kind: .division)
    let allFolds = proxy.reservedRegions(kind: .division, options: .includeInactive)
    let cameras = proxy.reservedRegions(kind: .occlusion)
    let frames = folds.map(\.frame)
    // …
}

// UIKit
let regions = view.reservedRegions(kind: .division)
let frames = regions.map(\.frame)
```

**Use inactive regions for high-level decisions.** For example, a grid can prefer an **even number of columns** whenever a division region exists, active or not, so columns never straddle the fold once the device folds.

**Sources:** [Strike a pose › Reserved regions on iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111463/?time=27) · [Query reserved regions](https://developer.apple.com/videos/play/tech-talks/111463/?time=399) · [Division and occlusion](https://developer.apple.com/videos/play/tech-talks/111463/?time=470) · [Prepare › Use reserved regions](https://developer.apple.com/videos/play/tech-talks/111461/?time=488)

---

## 11. Arrangements

*New in iOS 27.1.* **Arrangements** are layout containers that sit **between navigation containers** (`NavigationStack`, `NavigationSplitView`, `TabView`) **and content containers** (`List`, `ScrollView`). An arrangement places a **primary** and a **secondary** view according to rules. It turns these inputs:

- horizontal and vertical size classes
- aspect ratio (width ÷ height)
- whether a **division region** (the fold) is active

into these outputs: **whether each view is shown**, and **what frame it gets**. The same code works on every device you support, including iPad and non-folding iPhones.

Apple's example is Podcasts, with Now Playing as primary and Transcript as secondary. On iPad, showing the transcript splits the screen in half. On a folded iPhone Duo it's similar, but hiding the transcript keeps Now Playing in its **own region** instead of re-centering it across the fold. With the device taller than it is wide, there's no split and the transcript is shown inline.

### Adding one

```swift
// SwiftUI
NavigationStack {
    ArrangementView {
        PlayerView()          // primary
    } secondary: {
        UpNextView()
    }
    .arrangementViewStyle(.split)   // .split is the default
}
```

```swift
// UIKit
let arrangementVC = UIArrangementViewController()
let navController = UINavigationController(rootViewController: arrangementVC)
arrangementVC.setViewController(PlayerViewController(), for: .primary)
arrangementVC.setViewController(UpNextViewController(), for: .secondary)
```

### Split arrangement (default)

- Places views **side by side**, dividing the available bounds.
- Splits **horizontally** when the view is wider than tall and **vertically** when taller than wide.
- You can restrict the axes. If the arrangement **can't split along its primary axis**, it shows **only the primary view**.

```swift
// SwiftUI
.arrangementViewStyle(.split.axes(.horizontal))
// UIKit
arrangementVC.updateArrangement(.split.axes(.horizontal))
```

### Overlay arrangement

- Prefers to place views **above or below each other** (in z).
- When the device **folds**, it puts the two views **side by side**, which gives the secondary view room to grow.
- Read the **z-index** to adapt, for example by collapsing or expanding a panel:

```swift
// SwiftUI
ArrangementView {
    UpNextView()
} secondary: {
    PlayerView()
}
.arrangementViewStyle(.overlay)

struct UpNextView: View {
    @Environment(\.overlayArrangementZIndex) private var zIndex: Int
    var body: some View {
        UpNextList(minimization: zIndex > 0 ? .collapsed : .expanded)
    }
}
```

```swift
// UIKit
let primaryState = arrangementVC.state(for: .primary)
model.minimization = (primaryState?.zIndex ?? 0) > 0 ? .collapsed : .expanded
```

### Choosing an arrangement

| If you… | Use |
|---------|-----|
| Already build a split-like layout with `HStack`/`VStack` | **Split** |
| Already build an overlay layout with `ZStack` | **Overlay** |
| Have a clear **foreground/background** relationship, where partial obscuring is fine (for example, reading content scrolling under controls in Accessibility Reader) | **Overlay** |
| Have a **main/detail** relationship where neither side may ever be hidden (for example, Podcasts Now Playing and its transcript, or a player and its Up Next list) | **Split** |

### When *not* to use one

- Arrangements **provide no navigation infrastructure**. Don't put navigation containers such as `NavigationSplitView` **inside** an `ArrangementView`.
- Don't put an `ArrangementView` **inside** scrollable containers such as `List` or `ScrollView`.

**Sources:** [Strike a pose › System containers that adapt](https://developer.apple.com/videos/play/tech-talks/111463/?time=519) · [Introducing arrangements](https://developer.apple.com/videos/play/tech-talks/111463/?time=560) · [Build with ArrangementView](https://developer.apple.com/videos/play/tech-talks/111463/?time=677) · [Choose between arrangements](https://developer.apple.com/videos/play/tech-talks/111463/?time=879) · [When not to use an arrangement](https://developer.apple.com/videos/play/tech-talks/111463/?time=969)

---

## 12. The hinge as an input

Your app can react to the hinge, much as the system wallpaper zooms with the hinge angle when the device opens. You get:

- a **discrete status**: closed, partially open, fully open
- a **continuous angle** that updates live

| SwiftUI | UIKit |
|---------|-------|
| `onHingeChange { oldContext, newContext in … }` | `UIHingeInteraction`† |

`context.hinge` is **`nil` on devices without a hinge**. Check for it, and reset any hinge-driven state when you stop reading the angle.

```swift
struct InstrumentView: View {
    @State private var pitchBend: Double = 0   // 0 = none, 1 = deepest

    var body: some View {
        GuitarView(pitchBend: pitchBend)
            .onHingeChange { _, context in
                if let hinge = context.hinge, hinge.status == .partiallyOpen {
                    pitchBend = calculatePitchBend(angle: hinge.angle)   // hinge.angle is an Angle
                } else {
                    pitchBend = 0
                }
            }
    }
}
```

> Hinge data is meant for **interactions and effects**. For **layout**, use arrangements and reserved regions ([§10](#10-reserved-regions-api), [§11](#11-arrangements)).

**Sources:** [Leverage › Respond to the hinge](https://developer.apple.com/videos/play/tech-talks/111464/?time=49) · [Hinge data versus layout APIs](https://developer.apple.com/videos/play/tech-talks/111464/?time=155)

---

## 13. Multitasking and multiple scenes

### Split View multitasking

- **Every app takes part.** People drag an app to the side with the familiar home gesture to get a **50/50 split**. The two halves work independently.
- In Split View, controls sit along each app's **outer edge**, so an app on the left gets its controls on the **left** edge, away from the center.
- **Stacked layout:** a Picture in Picture video can be **pinned to the top**. The app underneath resizes **vertically** to fill the rest, in real time, and partially folding the device expands the video to half the screen.
- Handle both layouts the same way, with size classes and scene geometry. If you already support resizing on iPad or with iPhone Mirroring, you're in good shape.

### Multiple windows (scenes)

- iPhone Duo is the **first iPhone that supports multiple instances of your app's UI**. If you support multiple scenes on iPad, that support carries over.
- **Unlike on iPad, new windows can't be created on the outer display.** Only the inner display can create them, and this changes dynamically.
  - **Handle errors** when you request a new scene.
  - Use **`UIWindowSceneActivationAction`**, which **hides itself automatically** when new windows aren't available.
- If you're adding multiple-scene support for the first time, start with Apple's scene documentation.

**Sources:** [Leverage › Split view multitasking](https://developer.apple.com/videos/play/tech-talks/111464/?time=179) · [Support multiple scenes](https://developer.apple.com/videos/play/tech-talks/111464/?time=218) · [Design › Principles (split view, PiP)](https://developer.apple.com/videos/play/tech-talks/111466/?time=28)

---

## 14. Scene accessories

**Scene accessories** pair extra content with your app's main UI on **another display**, such as an iPhone acting as a game controller for an external display. The **system controls their availability**. They're enabled by default but can be switched on or off at any time, so observe availability and stay in sync.

### `CameraCaptureAccessory` (new on iPhone Duo)

This accessory shows extra UI on the **outer display** while the main camera UI stays on the **inner display**. Use it to show something to the person being photographed or recorded, such as a teleprompter or something fun for a child.

It's **available** only when:

- your app is **full screen on the inner display**, **and**
- your app has an **active camera session**.

**Register it on the same view as your camera UI**, so it's visible only while that view is.

```swift
struct CameraRootView: View {
    @State private var model = TeleprompterModel()

    var body: some View {
        CameraView(model: model)
            .sceneAccessory {
                CameraCaptureAccessory(isEnabled: $model.isEnabled) {
                    TeleprompterView(model: model)
                }
                .onAvailabilityChange { isAvailable in
                    model.isAvailable = isAvailable     // e.g. false when the device is closed
                }
            }
            .toolbar {
                TeleprompterToggle(isEnabled: $model.isEnabled)
                    .disabled(!model.isAvailable)
            }
    }
}
```

**Sources:** [Leverage › Scene accessories](https://developer.apple.com/videos/play/tech-talks/111464/?time=262) · [The camera capture accessory](https://developer.apple.com/videos/play/tech-talks/111464/?time=300)

---

## 15. Camera apps

### 15.1 Front cameras

| Capture device | How to get it | Max video | Notes |
|----------------|---------------|-----------|-------|
| **Virtual Front Camera** | `AVCaptureDevice.DiscoverySession` with position `.front` and a Wide or Ultra Wide device type | **1080p, 60 fps** | Switches automatically: inner camera when open, outer camera when closed. Only features **common to both** cameras. |
| **Inner ultra wide** (under-display) | `.builtInInnerUltraWideCamera` | 1080p, up to 60 fps | Full capabilities of that camera. |
| **Outer ultra wide** | `.builtInOuterUltraWideCamera` | Up to **4K**, up to **120 fps** | Full capabilities of that camera. |

- **Depth** is available **only through the individual cameras**, never through the virtual camera.
- If you use individual cameras, **your app switches between them** when the device opens or closes.

### 15.2 Camera direction

`AVCaptureDevice.position` is fixed, and both front cameras report `.front`. On iPhone Duo the displays can face opposite ways, so a "front" camera isn't always facing the user:

- Looking at the inner display while streaming from the outer front camera: that camera faces **away** from you.
- Closing the device mid-stream swings the outer front camera **toward** you.
- Flipping the open device lets you take a selfie with the **rear** cameras.

**`AVCaptureDeviceDirectionCoordinator`** (in **AVKit**) tells you where each camera faces **relative to your view** and keeps you updated as that changes. Create it with **your `UIView`**, the **device types to monitor**, and a **change handler**:

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

What it reports:

| Your view is on… | Forward-facing | Backward-facing |
|------------------|----------------|-----------------|
| Outer display (closed) | Outer front camera | Rear cameras |
| Inner display (open) | Inner front camera | Outer front camera and rear cameras |
| Outer display while open (device flipped) | Rear cameras **and** outer front camera | — |

Using **both displays at once** (with a scene accessory, [§14](#14-scene-accessories)) gives you two views, so create **one coordinator per view**. Each one reports directions relative to its own view. For example, the outer view's coordinator reports the rear cameras as forward-facing while the inner view's reports them as backward-facing.

**Concurrency rules:**

- The coordinator is tied to a view, so it's **main-actor isolated**.
- The change handler gives you **`AVCaptureDeviceDescriptor`**† values, not `AVCaptureDevice`s. A descriptor is a **`Sendable`, main-actor-safe** description with everything needed to create the device.
- **Don't call AVFoundation from the change handler.** Pass the descriptor to your camera actor and do the capture work there.

**In the change handler:**

1. **Reconfigure your `AVCaptureSession`** so it keeps streaming from the forward-facing camera.
2. **Revisit preview mirroring.** When the **rear** camera is forward-facing, mirror the preview for a natural selfie experience.
3. **Update your UI** for the camera change.

See also the article *Choosing a Camera by the Direction It Faces* in Apple's developer documentation.

### 15.3 Preview polish

- **Rear camera on the inner display.** Streaming at the rear camera's full field of view leaves **extra space** around the preview. Either **offset the preview** and group controls in the free space, or **fill** the display. Control this with `AVCaptureVideoPreviewLayer.videoGravity`.
- **Front ultra wide cameras.** The **square sensors** can fill the display. Use `AVCaptureDevice.dynamicAspectRatio` to pick a **landscape aspect ratio on the inner display**. See *Support the Center Stage front camera in your iOS app* (WWDC26).
- **Rotation.** Adopt **`AVCaptureDevice.RotationCoordinator`** so previews and photos stay upright. On iPhone Duo it **updates when your app moves between displays**. See the article *Supporting Device Rotation in Your Camera App*.
- **Performance.** After adopting the rotation coordinator, **turn off sensor-orientation compensation**. It's on by default for all of iPhone Duo's front cameras.

```swift
photoOutput.isCameraSensorOrientationCompensationEnabled = false
```

### 15.4 Camera next steps

1. Build with the **iOS 27.1 SDK**.
2. Decide how the app handles **camera switches when the device opens or closes**: the Virtual Front Camera, or individual cameras with a direction coordinator.
3. **Test on iPhone Duo** and make sure the preview looks right in every pose.

**Sources:** [Camera › Meet the new front cameras](https://developer.apple.com/videos/play/tech-talks/111465/?time=31) · [Virtual front camera](https://developer.apple.com/videos/play/tech-talks/111465/?time=58) · [Individual cameras](https://developer.apple.com/videos/play/tech-talks/111465/?time=113) · [Direction coordinator](https://developer.apple.com/videos/play/tech-talks/111465/?time=243) · [Device descriptors](https://developer.apple.com/videos/play/tech-talks/111465/?time=338) · [Handle a direction change](https://developer.apple.com/videos/play/tech-talks/111465/?time=377) · [Preview polish](https://developer.apple.com/videos/play/tech-talks/111465/?time=423) · [Rotation](https://developer.apple.com/videos/play/tech-talks/111465/?time=483)

---

## 16. Release checklist

### Build and test
- [ ] Built with **Xcode 27.1 / iOS 27.1 SDK**
- [ ] Tested in the **iPhone Duo simulator (Device Hub)**: closed portrait, closed landscape, open landscape, open portrait, book pose, tabletop pose
- [ ] Tested **Split View** with the app on the **left and right** of the inner display
- [ ] Tested with a **Picture in Picture video pinned to the top**, which resizes the app vertically
- [ ] Ran the **App Resizability** agent skill
- [ ] Tested with **Reduce Transparency** on (vertical bar backgrounds)
- [ ] Tested in a **right-to-left** language

### Layout
- [ ] Layout uses **size classes**, not idiom, interface orientation or screen size
- [ ] No **fixed widths, breakpoints or screen-specific metrics**
- [ ] No `UIScreen.main`. Uses traits, environment, scene bounds or `windowScene.screen`
- [ ] Interactive content is inside the **safe area**. Backgrounds extend past it.
- [ ] Code handles **asymmetric** safe-area insets and layout margins
- [ ] The outer display supports **landscape** (tent pose)
- [ ] Screen-corner shapes use **`ConcentricRectangle` / `UICornerConfiguration`**
- [ ] The inner display is **more than a stretched iPhone layout** (split view, two-column reflow or sidebar)
- [ ] **Same hierarchy and features** in every pose

### Bars
- [ ] Bars come from **navigation/tab containers**, not custom `UIToolbar`/`UINavigationBar`/`UITabBar`
- [ ] Every bar item has **both a title and a symbol**
- [ ] Back or Close at the top (`cancellationAction` / leading item). Prominent action pinned (`topBarPinnedTrailing` / `pinnedTrailingGroup`).
- [ ] Symbol/text toggling items set to `.horizontalOnly`. Custom views that can go vertical set to `.verticalPreferred`.
- [ ] Inline counts replaced with **badges**
- [ ] Custom views adapt to the vertical bar (`toolbarVerticalEdge` / `verticalBarEdge`)
- [ ] No extra spacing. Flexible spacers collapse vertically.
- [ ] **Compression behavior** chosen per view
- [ ] Custom overflow merged into **`ToolbarOverflowMenu` / `additionalOverflowItems`**. The ellipsis is used only for overflow.
- [ ] **Visibility priorities** set. Frequent and badged items stay visible longest.
- [ ] Vertical bars **disabled** only where justified (bottom-heavy single-page UI, single-button sheets)

### Fold
- [ ] No **interactive** element or important content rests **in the fold**. Scrolling content may cross it.
- [ ] Custom controls **displaced** using `reservedRegions(kind: .division)`
- [ ] Important UI avoids the inner **FaceTime camera** (`.occlusion`) when it matters
- [ ] Custom split and overlay layouts moved to **`ArrangementView` / `UIArrangementViewController`**
- [ ] No navigation containers **inside** an arrangement, and no arrangement **inside** a scroll view
- [ ] (Optional) Tabletop layout keeps every control and the same hierarchy

### Scenes and multitasking
- [ ] New-scene requests **handle failure** (no new windows on the outer display)
- [ ] `UIWindowSceneActivationAction` used for "open in new window"
- [ ] Scene-accessory **availability** is observed

### Camera (if applicable)
- [ ] Chosen approach: **Virtual Front Camera** (1080p60, no depth) or **individual cameras** with `AVCaptureDeviceDirectionCoordinator`
- [ ] Change handler reconfigures the session **off the main actor**, using device descriptors
- [ ] Mirroring is correct when a **rear camera faces the user**
- [ ] One coordinator **per view** when using both displays
- [ ] Preview layout uses `videoGravity` and `dynamicAspectRatio` where it helps
- [ ] `RotationCoordinator` adopted, and `isCameraSensorOrientationCompensationEnabled = false`
- [ ] (Optional) `CameraCaptureAccessory` for outer-display content

---

## 17. API quick reference

† = named in the talk narration only; check the exact spelling in the SDK.

| Purpose | SwiftUI | UIKit / AVFoundation |
|---------|---------|----------------------|
| Size classes | `@Environment(\.horizontalSizeClass)`, `\.verticalSizeClass` | `traitCollection.horizontalSizeClass` / `.verticalSizeClass` |
| Display scale (replaces `UIScreen.main.scale`) | `\.displayScale` | `traitCollection.displayScale` |
| Screen from scene | — | `window?.windowScene?.screen` |
| Concentric corners | `ConcentricRectangle` | `UICornerConfiguration` |
| Tab sidebar on the inner display | `.defaultTabBarPlacement(.sidebar)` | `tabBarController.sidebar.preferredPlacement = .sidebar` |
| Back/close item | `ToolbarItem(placement: .cancellationAction)` | `leadingItemGroups` + `leftItemsSupplementBackButton = false` |
| Pinned prominent action | `ToolbarItem(placement: .topBarPinnedTrailing)` | `navigationItem.pinnedTrailingGroup` |
| Item axis | `.axisBehavior(.verticalPreferred / .horizontalOnly)` | `UIBarButtonItem.axisBehavior` |
| Badge | `.badge(7)` | `item.badge = .count(7)` |
| Is a vertical bar present? | `@Environment(\.toolbarVerticalEdge)` | `traitCollection.verticalBarEdge` |
| Toolbar vs tab bar compression | `.toolbarVerticalCompressionBehavior(.prefersToolbarItems)` | `navigationItem.verticalBarCompressionBehavior = .prefersBarItems` |
| System overflow menu | `ToolbarOverflowMenu { … }` | `navigationItem.additionalOverflowItems` |
| Overflow priority | `.visibilityPriority(.high)` | `item.visibilityPriority = .high` |
| Disable vertical bar | `.toolbarVerticalBehavior(.disabled)` | `override var preferredVerticalBarBehavior: UIVerticalBarBehavior { .disabled }` |
| Reserved regions | `GeometryProxy.reservedRegions(kind:options:)` → `ReservedRegion`† | `UIView.reservedRegions(kind:options:)` → `UIViewReservedRegion`† |
| Region kinds / options | `.division`, `.occlusion` / `.includeInactive` | same |
| Arrangement container | `ArrangementView { } secondary: { }` | `UIArrangementViewController` + `setViewController(_:for:)` |
| Arrangement style | `.arrangementViewStyle(.split / .split.axes(.horizontal) / .overlay)` | `updateArrangement(.split.axes(.horizontal))` |
| Overlay z-index | `@Environment(\.overlayArrangementZIndex)` | `state(for: .primary)?.zIndex` |
| Hinge | `.onHingeChange { old, new in }`; `context.hinge?.status`, `.angle` | `UIHingeInteraction`† |
| New window action | — | `UIWindowSceneActivationAction` |
| Scene accessory | `.sceneAccessory { CameraCaptureAccessory(isEnabled:) { } }`, `.onAvailabilityChange` | — |
| Individual front cameras | — | `.builtInOuterUltraWideCamera`, `.builtInInnerUltraWideCamera` |
| Camera direction | — | `AVCaptureDeviceDirectionCoordinator(view:deviceTypes:changeHandler:)` (AVKit), `AVCaptureDeviceDescriptor`† |
| Preview fill / aspect | — | `AVCaptureVideoPreviewLayer.videoGravity`, `AVCaptureDevice.dynamicAspectRatio` |
| Rotation | — | `AVCaptureDevice.RotationCoordinator`; `AVCapturePhotoOutput.isCameraSensorOrientationCompensationEnabled` |

---

## 18. Glossary

- **Pose:** a physical configuration of the device, such as closed, open, rotated, book, tabletop or tent.
- **Outer / inner display:** the screen on the outside of the closed device, and the large folding screen inside.
- **Vertical bar:** the column along the side of the screen that combines navigation, toolbar and tab bar items.
- **Overflow menu:** the system ellipsis menu that collects bar items that don't fit.
- **Reserved region:** an area of a view shaped by hardware. It's either a *division* (the fold) or an *occlusion* (the inner FaceTime camera), and can be active or inactive.
- **Displacement:** moving, resizing or reorganizing existing UI so it stays visible and reachable around a reserved region.
- **Arrangement:** a layout container that places a primary and a secondary view based on size classes, aspect ratio and the fold. The styles are *split* and *overlay*.
- **Scene accessory:** extra UI your app shows on another display alongside its main scene, for example `CameraCaptureAccessory`.
- **Virtual Front Camera:** a capture device that switches automatically between the inner and outer front cameras.
- **Direction coordinator:** an AVKit object that reports which cameras face toward or away from the user, relative to a given view.

---

## 19. Further reading

**iPhone Duo tech talks**
- [Design for iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111466/)
- [Prepare your app for iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111461/)
- [Raise the bar with iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111462/)
- [Strike a pose with adaptive layouts on iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111463/)
- [Leverage multiple displays and scenes on iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111464/)
- [Build a great camera experience for iPhone Duo](https://developer.apple.com/videos/play/tech-talks/111465/)

**Related sessions**
- [Modernize your UIKit app (WWDC26)](https://developer.apple.com/videos/play/wwdc2026/278/): flexible layout principles and the app modernization / App Resizability skill
- [What's new in SwiftUI (WWDC26)](https://developer.apple.com/videos/play/wwdc2026/269/): toolbar visibility priority and more
- [Get to know the new design system (WWDC25)](https://developer.apple.com/videos/play/wwdc2025/356/): bar grouping and spacing
- [Support the Center Stage front camera in your iOS app (WWDC26)](https://developer.apple.com/videos/play/wwdc2026/341/): square-sensor front cameras and dynamic aspect ratio

**Documentation articles mentioned in the talks**
- *Choosing a Camera by the Direction It Faces*
- *Supporting Device Rotation in Your Camera App*
- Multiple-scene support (UIKit / SwiftUI scene documentation)
