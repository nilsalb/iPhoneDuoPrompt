# Prompt: make this app great on iPhone Duo

Copy everything below the line into the agent working on another app.

---

You're adapting this iOS app for **iPhone Duo**, Apple's foldable, which ships with iOS 27.1. It has a **cover screen** (folded) and a large **inner screen** (open flat or partly folded like a book). On both screens a system **vertical bar** on one edge holds the status bar, the tab bar and some bar items.

Goal:
- The app looks native on the Duo.
- When the device is open, screens with a main/supporting relationship show **two columns side by side**.
- **Nothing changes on any other iPhone.**

Use Apple's components and documented APIs only. **Don't hand-build layouts.** Every shortcut below was tried in another app and failed.

## 0. Ground rules

1. **Read Apple's material before writing layout code.**
   - Tech talk: "Strike a pose with adaptive layouts on iPhone Duo" (developer.apple.com/videos/play/tech-talks/111463/).
   - Doc pages are rendered with JavaScript. Fetch the raw JSON instead:
     `curl -s "https://developer.apple.com/tutorials/data/documentation/swiftui/arrangementview.json"`
     Swap in any doc path, for example `swiftui/view/listsectionmargins(_:_:)`.
   - The SDK is the ground truth. Grep the 27.1 `.swiftinterface` files and headers. Diff them against 27.0 to see what's new:
     `<Xcode 27.1>/…/iPhoneSimulator27.1.sdk/System/Library/Frameworks/{SwiftUI,SwiftUICore}.framework/Modules/*.swiftmodule/arm64-apple-ios-simulator.swiftinterface`
     and `UIKit.framework/Headers/{UIArrangementViewController,UISplitArrangement,UIViewReservedRegion,UIVerticalBarEdge,UIBarMinimization}.h`.
2. **Measure, don't guess.** When spacing looks wrong, log the real frames and safe areas, then reason from the numbers:
   ```swift
   .background(GeometryReader { g in Color.clear.onAppear {
       print("col", g.frame(in: .global), g.safeAreaInsets)
   }})
   ```
3. **Verify every change on screen**, on the Duo simulator in every pose (see §7), and on a regular iPhone for regressions. Green unit tests aren't enough.
4. **Duo changes must be inert elsewhere.** Gate them on the split actually showing, not on device names or screen sizes.

## 1. Build with the iOS 27.1 SDK

- Edge-to-edge content and the vertical bar only apply to apps **built with the 27.1 SDK**. Built with 27.0, the app runs letterboxed with a black strip.
- The Duo simulator needs the 27.1 runtime. Build with the 27.1 Xcode:
  `DEVELOPER_DIR=/path/to/Xcode-27.1.app/Contents/Developer xcodebuild …`
- **Keep the app compiling with the release SDK (27.0)**, because the App Store won't accept beta-SDK builds. Wrap every 27.1-only API in a compile-time check plus a runtime check:
  ```swift
  #if canImport(SwiftUICore, _version: 8.0.85)   // 27.0 SDK is 8.0.84.x, 27.1 is 8.0.85.x
  if #available(iOS 27.1, *) { /* Duo code */ } else { /* existing */ }
  #else
  /* existing */
  #endif
  ```
  Check the module versions in your own SDKs with `head -5 <SwiftUICore swiftinterface>` and look for `user-module-version`. iOS 27.0 APIs, like the bar minimization ones below, only need `#available(iOS 27.0, *)`.
- **Don't change `UISupportedInterfaceOrientations` as a "Duo fix."** Adding landscape to a portrait-only app made the system run it in a window with a black vertical strip. Keep the existing orientation settings and confirm on the simulator.

## 2. The vertical bar: use system containers

- Use the standard `TabView`, `NavigationStack` and `.toolbar`. The system moves the tab bar, navigation items and status bar into the vertical bar by itself.
- Custom tab bars and standalone `UINavigationBar`/`UIToolbar` instances don't take part in the vertical bar.
- Give every tab and toolbar item **both a title and an SF Symbol**, so the system can choose the compact vertical representation.
- You can't set where the tab rail sits vertically; the system positions it. Don't fight it.
- `.toolbarVerticalBehavior(.disabled)` and `preferredVerticalBarBehavior = .disabled` opt a screen out of the vertical bar. They're only for screens like full-screen video or a calculator. Never use them to fix spacing.

## 3. Two columns side by side: `ArrangementView`

Use a split when a screen has a **main/supporting relationship**: the primary is the screen's job and the secondary is what you'd otherwise tap into, such as analytics, details or a preview.

**Structure (Apple's pattern):** one `NavigationStack` outside, with an `ArrangementView` inside.

```swift
NavigationStack {
    ArrangementView {
        MainScreen()          // primary
    } secondary: {
        SupportingScreen()    // secondary
    }
    .arrangementViewStyle(.split.axes(.horizontal))
}
```

- `.split.axes(.horizontal)` shows **only the primary** wherever a side-by-side split doesn't fit. That covers every other iPhone and the Duo closed. This is what makes the change Duo-only; you don't need device checks.
- **Don't** put `NavigationStack` or `NavigationSplitView` *inside* an `ArrangementView`, and don't put an `ArrangementView` inside a `List` or `ScrollView`.
- **Don't** build splits with `HStack`, `GeometryReader` or division frames, or by wrapping `UIArrangementViewController` in a representable. Each column then gets its own navigation controller, and each reserves the vertical bar's width (about 84pt) as trailing padding. You get big asymmetric margins.
- Pushing from either column pushes onto the shared stack, full screen. That's expected.
- Use the overlay arrangement (`.arrangementViewStyle(.overlay)`) instead of split only for a foreground/background relationship, such as controls over media.

### 3a. Knowing whether the secondary is showing

The documentation says `@Environment(\.splitArrangementAxis)` is non-nil inside a split. **In testing it read nil in both columns even with both visible.** Use this instead: the arrangement only builds the secondary when it's shown, so track it.

```swift
@State private var besideShowing = false
…
} secondary: {
    SupportingScreen()
        .onAppear { besideShowing = true }
        .onDisappear { besideShowing = false }
}
```

Pass `besideShowing` down through a custom `EnvironmentValues` entry. The primary uses it to **leave out content the secondary already shows**: no duplicate charts or summaries. Each piece of information appears once.

### 3b. Margins: standard outer margins, standard gutter at the fold

- The system gives each column its normal inset-grouped margin (20pt) **only on the screen-edge side**. On the fold side the margin is 0, so cards touch at the fold.
- Apple's guidance is "preserve outer margins; increase spacing around the hinge."
- Add the same standard margin on each column's fold-side edge, **only while split**:
  ```swift
  primaryColumn.contentMargins(.trailing, 20)    // default placement
  secondaryColumn.contentMargins(.leading, 20)
  ```
- Use the **default** placement. `.contentMargins(_, _, for: .scrollContent)` did nothing to list sections.
- `listSectionMargins(_:_:)` only works when applied to each `Section`, not to the `List`.
- Don't apply any of this when not split. Setting 0 elsewhere overrides the system default.
- Measure the result: the outer margin should equal the fold-side margin, which gives a 40pt gutter.

### 3c. Background behind the fold

Backgrounds extend behind bars and the fold; content stays clear of them. Without this, a white strip shows at the hinge when the device is partly folded:

```swift
ArrangementView { … } secondary: { … }
    .arrangementViewStyle(.split.axes(.horizontal))
    .background(Color(uiColor: .systemGroupedBackground).ignoresSafeArea())
```

Match the background to whatever your columns use.

### 3d. One navigation bar over two columns

Both columns share one navigation bar, and **the bar reacts to whichever column scrolls**. That collapses the large title, minimizes the bar, or shifts the safe area, so scrolling the right column moved or resized the left. While split, use the iOS 27 APIs on the primary screen:

```swift
.toolbarTitleDisplayMode(.inline)                                       // title keeps one size
.toolbarMinimizationBehavior(.never, for: .navigationBar)               // iOS 27.0
.toolbarMinimizationSafeAreaAdjustment(.disabled, for: .navigationBar)  // iOS 27.0: content never shifts
```

- Use `.inline`, **not** `.inlineLarge`. A UI test showed `.inlineLarge` still shrinks (34pt to 21pt) when the other column scrolls. `.inline` held steady.
- Apply these modifiers inside the primary screen, where its `.navigationTitle` is set. They do reach the shared bar from there.

- **Don't hide the navigation bar to fix this.** Keep the title. It names the screen for VoiceOver and labels the back button when you push.
- Apply these only while split, so the screen keeps the system's normal collapsing title everywhere else.

### 3e. Scroll edge effects

- The blur under bars is clipped to each scroll view. Here `.automatic` resolved to the **hard** style: an opaque band ending in a sharp line at the column edge.
- While split, give **both** columns the soft style so the fade matches and meets in the gutter:
  `.scrollEdgeEffectStyle(.soft, for: .top)`
- Hiding the effect on one column only (`scrollEdgeEffectHidden`) leaves the other ending on a hard line.

### 3f. Choosing what goes in each column

- **Primary:** what the screen is for, including actions, today's items and editing.
- **Secondary:** context that supports it, such as charts, history, details and previews.
- Decide per tab. Don't reuse one secondary everywhere. For example, beside "Today" show this week's numbers; beside "Plan" show the season's tracking. If a chart lives in one tab's secondary, don't repeat it in another tab's secondary.
- Remove anything from the primary that the secondary now shows, but only while split.
- Within cards, use standard components: `LabeledContent` for label/value rows, a separate `NavigationLink` row for drill-ins (never a card that's a link *and* draws its own chevron), and Swift Charts' `AxisMarks(values: .automatic(desiredCount:))` so axis labels don't collide at new widths.

### Reference implementation (SwiftUI)

```swift
struct TodayTab: View {
    var body: some View {
        NavigationStack { BesideSupport(kind: .week) { TodayView() } }
    }
}

struct BesideSupport<Main: View>: View {
    let kind: SupportKind
    @ViewBuilder var main: Main
    @State private var besideShowing = false

    var body: some View {
        #if canImport(SwiftUICore, _version: 8.0.85)
        if #available(iOS 27.1, *) {
            ArrangementView {
                ArrangedColumn(edgeAtFold: .trailing, split: besideShowing) { main }
                    .scrollEdgeEffectStyle(besideShowing ? .soft : .automatic, for: .top)
            } secondary: {
                ArrangedColumn(edgeAtFold: .leading, split: true) { SupportView(kind: kind) }
                    .scrollEdgeEffectStyle(.soft, for: .top)
                    .onAppear { besideShowing = true }
                    .onDisappear { besideShowing = false }
            }
            .arrangementViewStyle(.split.axes(.horizontal))
            .background(Color(uiColor: .systemGroupedBackground).ignoresSafeArea())
        } else { main }
        #else
        main
        #endif
    }
}

@available(iOS 27.1, *)
private struct ArrangedColumn<Content: View>: View {
    let edgeAtFold: Edge.Set
    let split: Bool
    @ViewBuilder var content: Content
    var body: some View {
        let column = content.environment(\.supportBeside, split && edgeAtFold == .trailing)
        if split { column.contentMargins(edgeAtFold, 20) } else { column }
    }
}

extension View {
    /// Apply on the primary screen (where its navigationTitle is set).
    @ViewBuilder func steadyNavigationBar(_ steady: Bool) -> some View {
        if #available(iOS 27.0, *), steady {
            self.toolbarTitleDisplayMode(.inline)
                .toolbarMinimizationBehavior(.never, for: .navigationBar)
                .toolbarMinimizationSafeAreaAdjustment(.disabled, for: .navigationBar)
        } else { self }
    }
}

extension EnvironmentValues { @Entry var supportBeside = false }
// In the primary screen:
//   @Environment(\.supportBeside) var supportBeside
//   .navigationTitle("Today").steadyNavigationBar(supportBeside)
//   if !supportBeside { DuplicateChart() }
```

## 4. Custom layouts and the fold

- For anything you lay out manually (canvases, custom controls), query reserved regions:
  `GeometryReader { g in g.reservedRegions(kind: .division, options: .includeInactive) }`
  - The fold's division region is **active (width > 0) only when partly folded**. Flat, it's inactive with width 0, so pass `.includeInactive` to know a fold exists at all.
  - `.occlusion` regions are the cameras.
- Keep controls and critical content out of the division. Continuously scrolling content (lists, feeds, articles) doesn't need moving.
- System sheets, alerts and menus already avoid the fold.

## 5. The cover screen

- It's wider and shorter than a normal iPhone, with the vertical bar on one side.
- Standard containers adapt automatically. Check nothing is clipped, the tab rail is reachable, and no layout assumes a portrait-phone aspect ratio.

## 6. What not to do (all tried, all wrong)

- Allowing landscape to "fix" the layout: it caused a letterboxed window with a black strip.
- Hand-built `HStack`/`GeometryReader` splits, or wrapping `UIArrangementViewController`: wrong margins and duplicated bar insets.
- Two `NavigationStack`s, one per column: each reserves vertical-bar space.
- `contentMargins(for: .scrollContent)` or `listSectionMargins` on the `List` for column gutters: no effect.
- Hiding the navigation bar to stop cross-column movement: you lose the title.
- `.toolbarTitleDisplayMode(.inlineLarge)` to stop the title resizing: it still shrinks when the other column scrolls. Use `.inline`.
- `scrollEdgeEffectHidden` on one column only: leaves a hard seam.
- Trusting `splitArrangementAxis` to detect the split: it read nil.

## 7. Verification checklist

Use the Duo simulator: device type "iPhone Duo", 27.1 runtime, with the 27.1 Xcode via `DEVELOPER_DIR`.

- **The screens are separate displays.**
  - List them with `xcrun simctl io <udid> enumerate`. The inner screen is 2007×2853 px; the cover is 1398×2034.
  - Capture one with `xcrun simctl io <udid> screenshot --display=<display UUID> out.png`.
  - Captures are sometimes black. Retry until the file is a plausible size.
  - `XCUIScreen.main.screenshot()` captures the cover even when the device is open.
- **Change poses in the Simulator** and check each one: folded, open flat and partly open (book).
  - Folded: one column, vertical bar with tabs.
  - Flat: two columns, 20pt outer margins, 40pt gutter.
  - Book: the same, plus no white strip at the hinge.
- **Write a UI test that proves the columns scroll independently** and run it on the Duo:
  ```swift
  func testColumnsScrollIndependently() throws {
      let secondary = app.descendants(matching: .any)["secondaryColumn"].firstMatch
      guard secondary.waitForExistence(timeout: 3) else { throw XCTSkip("no split here") }
      let anchor = app.staticTexts["<text in the primary column>"].firstMatch
      let title = app.navigationBars.staticTexts["<Title>"].firstMatch
      let (y, h) = (anchor.frame.minY, title.frame.height)
      secondary.swipeUp(velocity: .slow); sleep(1)
      XCTAssertEqual(anchor.frame.minY, y, accuracy: 0.5)   // primary didn't move
      XCTAssertEqual(title.frame.height, h, accuracy: 0.5)   // title didn't resize
      secondary.swipeDown(velocity: .slow); sleep(1)
      XCTAssertEqual(anchor.frame.minY, y, accuracy: 0.5)
  }
  ```
  - Also assert the swipe really scrolled the secondary: one of its elements moved or left the screen.
  - Assert the title **exists** before comparing its height; otherwise a missing title passes silently.
- **Regression-test on the iPhone 14 Plus simulator** (iOS 27.0 runtime) with the release Xcode. Use only that device for regression runs, not some other iPhone. Run the full UI test suite; nothing should change there, and the Duo split tests should report "skipped".
- Build with **both** Xcodes: 27.1 for the Duo, 27.0 for the release build.
