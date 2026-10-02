# iPhone Duo Prompt

A one-shot prompt that gets a coding agent (Claude Code, Codex, Cursor and others) to adapt an existing iOS app for **iPhone Duo**, Apple's foldable running iOS 27.1, using only Apple's components and documented APIs.

The result:

- **Native on the Duo.** The tab bar, toolbar items and status bar move into the system vertical bar on both screens.
- **Two columns when open.** Screens with a main/supporting relationship show their supporting content side by side, through `ArrangementView`.
- **Nothing changes on any other iPhone.** Every Duo change is gated on the split actually showing, and the app still builds with the release SDK.

## How to use it

1. Open [`iphone-duo-agent-prompt.md`](iphone-duo-agent-prompt.md).
2. Copy everything below the line into your agent, in the repo of the app you want to adapt.
3. Have Xcode 27.1 (for the Duo simulator) and your release Xcode (27.0) installed side by side. The agent builds with both.

## What's inside

| Section | Covers |
|---|---|
| 0. Ground rules | Read Apple's docs first (with a trick to fetch the JS-rendered doc pages as JSON), measure instead of guessing, verify on screen |
| 1. Build with the 27.1 SDK | Why 27.0 builds run letterboxed, and how to keep the app compiling for App Store submission (`canImport` + `#available`) |
| 2. The vertical bar | Why custom tab bars don't join it, and why every bar item needs a title and an SF Symbol |
| 3. Two columns | `ArrangementView` structure, detecting the split, fold-side margins, the background behind the hinge, a steady shared navigation bar, scroll edge effects, choosing what goes in each column, plus a reference SwiftUI implementation |
| 4. Custom layouts and the fold | Reserved regions (`.division`, `.occlusion`) |
| 5. The cover screen | What to check on the wide, short folded display |
| 6. What not to do | Every shortcut that was tried and failed |
| 7. Verification checklist | Screenshotting each display, changing poses, folding and launching while folded, a UI test that proves the columns scroll independently, and regression runs on an iPhone 14 Plus |

## Dead ends, so your agent skips them

Every one of these was tried in a real app and failed:

- Allowing landscape to "fix" the layout: the app ran letterboxed with a black strip.
- Hand-built `HStack` / `GeometryReader` splits, or wrapping `UIArrangementViewController`: wrong margins and duplicated vertical-bar insets.
- A `NavigationStack` per column: each one reserves vertical-bar space.
- `contentMargins(for: .scrollContent)` or `listSectionMargins` on the `List` for column gutters: no effect.
- Hiding the navigation bar to stop cross-column movement: you lose the title.
- `.toolbarTitleDisplayMode(.inlineLarge)`: still shrinks when the other column scrolls.
- `scrollEdgeEffectHidden` on one column only: leaves a hard seam.
- Trusting `splitArrangementAxis` to detect the split: it read nil.
- Trusting the secondary column's `onAppear`/`onDisappear` alone: it reported "split" on the folded cover screen, so content went missing there.
- Swapping between a custom tab bar and the system `TabView` on every environment change: it resets navigation and open sheets.

## Requirements

- Xcode 27.1 with the iOS 27.1 simulator runtime and the "iPhone Duo" device type
- Your release Xcode (27.0) for App Store builds and regression runs
- A SwiftUI app (the UIKit equivalents are named in the prompt)

## Feedback

Found another dead end or a better API? Open an issue or a pull request.
