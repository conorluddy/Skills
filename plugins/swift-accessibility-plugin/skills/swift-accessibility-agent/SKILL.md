---
name: swift-accessibility-agent
description: Audit, fix, and initialise SwiftUI accessibility so an app is navigable by VoiceOver users, XCUITest, and AI agents. Use this skill whenever a user mentions an accessibility audit, accessibility modifiers, identifiers/labels/hints/values/traits, VoiceOver, Dynamic Type, Reduce Motion, colour contrast, making an iOS app navigable by agents, CoordinateTracker, trackElement, or wants to improve SwiftUI accessibility coverage. Also trigger on "audit accessibility", "add accessibility", "make navigable", "init accessibility", "VoiceOver can't reach X", or a ticket whose acceptance criteria list VoiceOver or Dynamic Type. Three modes: init (scaffold CoordinateTracker infrastructure), audit (report gaps), fix (add missing modifiers and repair the traps that silently break VoiceOver).
compatibility: Requires Xcode project with SwiftUI views
allowed-tools: Bash(find:*) Read Write Edit Glob Grep
---

# Swift Accessibility Agent

Make SwiftUI apps fully navigable by VoiceOver, XCTest, and AI agents by ensuring every
interactive element carries the five accessibility properties — **identifier**, **label**,
**hint**, **value**, **traits** — and by catching the modifiers that quietly *remove*
accessibility that was already working.

## Why this matters

Most AI agents navigate iOS apps via screenshots — slow (~2-5s per step), expensive
(~1,600 image tokens per screenshot), and fragile. A fully populated accessibility tree
lets agents query structured text (~200-400 tokens), tap by identifier (deterministic),
and verify via logs — no vision model needed. The same work also makes the app properly
accessible to humans using VoiceOver, Switch Control, and Voice Control.

The failure mode worth designing against: an app whose *missing* modifiers are obvious and
get fixed, while its *wrong* ones survive the pass untouched. A screen with no labels reads
badly. A screen with a well-meaning `.accessibilityElement(children: .combine)` wrapped
around its only button is unusable, and looks fine in code review. Read "Traps that silently
break VoiceOver" below before fixing anything.

## Three modes

The user will tell you what they want, or you can suggest the right mode based on context.

### 1. `init` — Scaffold CoordinateTracker

Creates the `CoordinateTracker.swift` file in the project. This is the infrastructure
that lets agents query exact screen coordinates for any tracked element without screenshots.

**When to use**: First time setting up a project for agent navigation, or when the user
says "init", "set up tracking", or "add coordinate tracker".

**Steps**:

1. Ask the user where Swift source files live (e.g. `Sources/`, `App/`, etc.) — or detect
   the most likely location by looking for existing `.swift` files
2. Check if `CoordinateTracker.swift` already exists anywhere in the project
3. If not, create it using the CoordinateTracker reference implementation below
4. Confirm the file location with the user

### 2. `audit` — Report accessibility gaps

Scans SwiftUI files and reports which interactive elements are missing accessibility
modifiers, without changing any code.

**When to use**: The user wants to understand current coverage before making changes,
or says "audit", "check accessibility", "what's missing".

**Steps**:

1. Identify the target scope — a single file, a directory, or a glob pattern
2. Find all `.swift` files in scope
3. For each file, scan for interactive SwiftUI elements (see "What to scan for" below)
4. For each element, check which of the five properties are present
5. Check the same files for the traps in "Traps that silently break VoiceOver" — these
   are findings, not gaps, and belong at the top of the report where they can't be missed
6. Produce a structured report:

```
## Accessibility Audit Report

### Blockers

| File | Line | Finding |
|------|------|---------|
| ActiveGoalScreen.swift | 85 | `children: .combine` spans the "Mark complete" button — action unreachable |
| ActiveGoalScreen.swift | 86 | Explicit label on combined element drops the countdown from the announcement |

### file: Views/SessionTimerView.swift

| Line | Element | Type | identifier | label | hint | value | traits |
|------|---------|------|:---:|:---:|:---:|:---:|:---:|
| 23   | "Save"  | Button | — | — | n/a | n/a | auto |
| 45   | HStack  | List row | — | — | — | — | — |
| 67   | Toggle  | Toggle | — | OK | — | — | auto |

### Summary
- Files scanned: 12
- Interactive elements found: 34
- Fully accessible: 8 (24%)
- Blockers: 2
- Missing identifiers: 26
- Missing labels: 18
- Missing values: 14 (of elements that carry state)
```

Mark a cell `n/a` rather than `—` when the property genuinely doesn't apply. `value` only
applies to elements that carry state (Toggle, Picker, Slider, Stepper, list rows with data,
progress indicators, a text field with a character budget). `hint` applies only where the
outcome isn't already obvious from the label — see the hints guidance below. `traits` are
usually inferred by SwiftUI (Button gets `.button`) — flag them only when ambiguous, such as
a tappable `HStack` that never announces itself as a button. Reporting a button as "missing"
a value it should never have inflates the gap count and buries the findings that matter.

### 3. `fix` — Add missing accessibility modifiers

Reads each file, identifies gaps, and adds the appropriate modifiers. This is the
main workhorse mode.

**When to use**: The user wants to actually improve their code, or says "fix",
"add modifiers", "make accessible", "augment".

**Steps**:

1. Run the audit logic first to identify gaps
2. Repair the blockers before adding anything — a screen whose button is unreachable is
   not improved by giving that button a better label
3. For each element with gaps, add the missing modifiers
4. Follow the naming convention and modifier patterns below
5. If `--track` or "with tracking" is mentioned, also add `.trackElement()` calls
   (requires `init` to have been run first — check for CoordinateTracker.swift)
6. Show the user what changed before applying (or apply directly if they've asked
   for that)

## Traps that silently break VoiceOver

These are the highest-value findings in any real codebase, because the code compiles, looks
deliberate, and reads as *more* accessible than the code without them.

### `children: .combine` wrapped around an interactive element

```swift
// Broken: the button is absorbed into the combined element.
VStack {
    Text(goal.title)
    Text(countdown)
    Button("Mark complete") { complete() }
}
.accessibilityElement(children: .combine)
.accessibilityLabel("This week: \(goal.title)")
```

Combining flattens descendants into one element. A VoiceOver user hears the goal and has no
way to complete it — the app's only action has been narrated out of existence. Combine the
informational part and leave interactive children outside it:

```swift
VStack {
    summary            // its own combined element, labelled
    Button("Mark complete") { complete() }
        .accessibilityIdentifier(AccessibilityIdentifier.completeButton)
}
```

Same trap with `NavigationLink` inside a combined list row, and with a container carrying
`.onTapGesture`. Rule of thumb: if the subtree can be activated, it stays its own element.

### An explicit label on a combined element replaces its children

`.accessibilityElement(children: .combine)` builds an announcement from the children's own
labels. Adding `.accessibilityLabel(...)` afterwards **replaces** that, so anything you
forget to restate is now silent. If a screen shows a title, a description, a countdown and
an urgency state, and the explicit label mentions three of them, the fourth is simply gone —
and nothing in the code says so.

When you write an explicit label over combined children, enumerate what the element displays
and confirm each fact appears. Where several surfaces describe the same model (a screen, a
widget, a Lock Screen accessory), build the sentence in one pure function they all call
rather than in each view — the divergence is otherwise invisible until someone listens to it.

### Formatted numbers read as loose digits

A monospaced `3d 21h 5m` is announced as "3 2 1 5". Any compact time, score, ratio or version
string needs a spoken form distinct from its displayed form:

```swift
Text(CountdownFormatter.displayString(remaining: remaining))
    .accessibilityLabel(CountdownFormatter.accessibilityLabel(remaining: remaining))
    // "3 days, 21 hours, 5 minutes remaining"
```

### Status carried only by colour

A red background meaning "urgent" does not exist for a VoiceOver user, and doesn't survive
Increase Contrast, Smart Invert, or a tinted Lock Screen widget either. Every colour-coded
state needs a text equivalent that reaches the accessibility layer — and the check is that
the state's own label appears in the announcement, not that the colour has a name somewhere.

### Decoration that isn't hidden, and duplication that is spoken twice

A chevron announced as "chevron.right", a divider announced as "image", a section header
repeated inside the row it heads. Hide decoration with `.accessibilityHidden(true)`, and
hide text that a parent's label already covers.

## What to scan for

These SwiftUI elements need accessibility modifiers when interactive or informational:

### Always needs full coverage
- `Button` / `Button(action:)` / `.onTapGesture`
- `NavigationLink`
- `Toggle`
- `Picker` / `DatePicker`
- `Slider`
- `Stepper`
- `TextField` / `SecureField` / `TextEditor`
- `Link`
- `Menu`

### Needs coverage when tappable or informational
- `HStack` / `VStack` / `ZStack` used as list rows (look for `onTapGesture`,
  `NavigationLink` wrapping, or `List { ... }` context)
- `Image` that conveys meaning (not decorative)
- `Label` when used standalone
- `Text` that displays dynamic state
- Custom view structs used as interactive components

### Should be hidden (`.accessibilityHidden(true)`)
- Decorative `Image(systemName: "chevron.right")` disclosure indicators
- Decorative shapes (circles, dividers used purely for visual effect)
- Redundant text already represented by a parent element's label

### View-level identifiers
- `ScrollView`, `List`, `Form`, `NavigationStack` — the top-level container of each
  screen should have `.accessibilityIdentifier("screen_name_view")` so agents can
  orient themselves

### Also worth flagging while you are in the file
- `.disabled(...)` with no explanation of *why* — VoiceOver says "dimmed" and stops there
- Buttons presented from `.confirmationDialog` / `.alert` whose labels duplicate a button
  on the screen beneath: `app.buttons["Lock it in"]` then matches two elements
- Fixed-height frames and `Spacer()` layouts holding text that must grow with Dynamic Type

## Naming convention

Use this structured pattern for identifiers:

```
{category}_{context}_{element}_{modifier?}
```

- **category**: The domain area (`technique`, `session`, `position`, `settings`, `navigation`)
- **context**: The screen or section (`editor`, `list`, `detail`, `timer`, `tab_bar`)
- **element**: The UI type (`button`, `row`, `textfield`, `toggle`, `picker`)
- **modifier** (optional): Disambiguator (`save`, `delete`, `name`, `filter`)

Examples:
```swift
"technique_editor_save_button"
"position_list_row_\(position.id)"
"session_timer_start_button"
"navigation_tab_bar_training"
"form_textfield_technique_name"
"settings_notifications_toggle"
```

Infer `category` and `context` from the file name, containing view struct, and
surrounding code. The identifier should be self-describing — someone reading
`"technique_editor_save_button"` in a log should immediately know the domain,
screen, and element without looking up code.

### Declare identifiers once, not at every call site

An identifier exists to be matched by something else — a UI test, an agent script. A literal
typed into the view and typed again into the test is two strings that agree today and drift
silently later; the test keeps passing against an element that no longer exists, or matches
nothing and fails for a reason that looks like a UI bug.

Prefer a single namespace, added to both the app target and the UI test target:

```swift
enum AccessibilityIdentifier {
    static let creationLockButton = "creation_button_lock"
    static let creationConfirmLockButton = "creation_button_confirm_lock"

    /// Rows need a per-item identifier; key it on something stable across relaunches.
    static func historyRow(id: UUID) -> String { "history_row_\(id)" }
}
```

Offer this when the project has a UI test target or an agent-driven test setup. If the user
prefers literals, follow the convention above and keep them consistent — don't argue the point
twice.

## How to write good labels, hints, and values

### Labels (`.accessibilityLabel()`)
- Describe **what the element is**, not how it looks
- Read it as if you're using the app without a screen
- Good: `"Save technique"`, `"Guard position"`, `"Session duration"`
- Bad: `"Button"`, `"MarqueeText"`, `"Blue circle"`

### Hints (`.accessibilityHint()`)
- Describe **what happens** when you interact, in present tense
- Good: `"Validates and stores the current technique"`
- Bad: `"Tap to save"` (VoiceOver already tells users to tap)
- Bad: `"Saves"` on a button labelled "Save" — a hint that restates the label is pure noise,
  and VoiceOver users hear it on every pass

Hints are optional by design, and they are read after a pause on every focus. Add one where
the consequence isn't obvious from the label — an action that is irreversible, one that
navigates somewhere unexpected, or a control that is disabled and should say why:

```swift
.accessibilityHint(isReady
    ? "Locks this goal for the rest of the week. It can't be changed afterwards."
    : "Unavailable until both a title and a description are entered.")
```

Leaving a hint off a self-evident button is the correct outcome, not a gap. Count it `n/a`.

### Values (`.accessibilityValue()`)
- The **current state** of the element
- Only for elements with state (toggles, pickers, counters, list rows with data)
- Good: `"3 of 5 selected"`, `"On"`, `"Page 2 of 4"`, `"\(position.transitionCount) transitions"`
- Bad: (omit entirely if the element has no state — don't set an empty value)

A visible counter next to a field is usually better expressed as that field's value than as
its own element: `"7/10"` alone tells a VoiceOver user nothing, while
`.accessibilityValue("7 of 10 characters")` on the field, with the visible counter hidden,
puts the count where the user already is.

## The five properties are not the whole job

A pass that stops at modifiers will still fail a real accessibility review. When the user
asks for an audit or a "pass" rather than a specific modifier, check these too and report
them alongside — each is cheap to verify and expensive to retrofit:

- **Dynamic Type.** Build the type scale on system text styles (`.title`, `.caption`), never
  fixed point sizes. Then check the largest accessibility sizes: text that must not truncate
  needs `fixedSize(horizontal: false, vertical: true)`, and a screen whose content overflows
  at AX5 needs a `ScrollView`, not a smaller font.
- **Contrast.** Every foreground/background pairing, in both light and dark appearance,
  against WCAG AA (4.5:1 for body text). If the palette is expressed as values in code, this
  is assertable in a unit test rather than checked by eye.
- **Reduce Motion.** Read `@Environment(\.accessibilityReduceMotion)` and honour it at every
  animation site — transitions and `withAnimation` blocks both.
- **Locale.** Dates and times via `.formatted(...)`, never a hardcoded format string.

Report these as their own section. Don't silently expand a narrow request ("add identifiers
to this file") into a full pass — mention what you noticed and let the user decide.

## Modifier placement pattern

Add modifiers directly after the element, before any layout modifiers like `.padding()`
or `.frame()`. Group accessibility modifiers together:

```swift
Button("Save") {
    saveTechnique()
}
.accessibilityIdentifier("technique_editor_save_button")
.accessibilityLabel("Save technique")
.accessibilityHint("Validates and stores the current technique")
.padding()
.frame(maxWidth: .infinity)
```

For list rows, apply modifiers to the outermost container and hide decorative children:

```swift
HStack(spacing: 12) {
    Circle().fill(.blue).frame(width: 8)
        .accessibilityHidden(true)
    VStack(alignment: .leading) {
        Text(position.name)
        Text("\(position.transitionCount) transitions")
            .foregroundStyle(.secondary)
    }
    Spacer()
    Image(systemName: "chevron.right")
        .accessibilityHidden(true)
}
.accessibilityIdentifier("position_list_row_\(position.id)")
.accessibilityLabel(position.name)
.accessibilityHint("Opens detailed information for \(position.name)")
.accessibilityValue("\(position.transitionCount) transitions")
```

Note that this row is safe to combine only because the whole row is the tap target. If the
row contained its own button *and* a navigation link, they would need to stay separate.

## `.trackElement()` (opt-in)

Only add `.trackElement()` when the user explicitly opts in (says "with tracking",
passes `--track`, or has run `init`). When adding it, use the same string as the
`accessibilityIdentifier`:

```swift
Button("Start session") { startSession() }
    .accessibilityIdentifier("session_timer_start_button")
    .accessibilityLabel("Start training session")
    .accessibilityHint("Begins a new timed training session")
    .trackElement("session_timer_start_button")
```

## CoordinateTracker reference implementation

Drop this into your project as `CoordinateTracker.swift` during `init` mode:

```swift
import SwiftUI

@MainActor
final class CoordinateTracker: ObservableObject {
    static let shared = CoordinateTracker()
    private init() {}

    struct TrackedElement {
        let id: String
        let frame: CGRect
        var center: CGPoint { CGPoint(x: frame.midX, y: frame.midY) }
    }

    private(set) var elements: [String: TrackedElement] = [:]
    private(set) var currentView: String?
    private(set) var viewMetadata: [String: String] = [:]

    func track(id: String, frame: CGRect) {
        elements[id] = TrackedElement(id: id, frame: frame)
    }

    func tapPoint(for id: String) -> CGPoint? {
        elements[id]?.center
    }

    func updateViewContext(viewName: String, metadata: [String: String] = [:]) {
        currentView = viewName
        viewMetadata = metadata
    }
}

extension View {
    func trackElement(_ id: String) -> some View {
        background(
            GeometryReader { geo in
                Color.clear.onAppear {
                    CoordinateTracker.shared.track(
                        id: id,
                        frame: geo.frame(in: .global)
                    )
                }
            }
        )
    }
}
```

Update view context on screen appear:

```swift
.onAppear {
    CoordinateTracker.shared.updateViewContext(
        viewName: "SessionTimerView",
        metadata: ["sessionId": session.id]
    )
}
```

## Verifying the work

Reading the diff proves the modifiers exist, not that the screen is usable. Where the project
has a simulator workflow available, finish by checking the tree the app actually publishes:
dump the accessibility hierarchy for each screen and confirm every action appears as its own
element with a label, then sweep the largest Dynamic Type size and both appearances.

The check that catches the combine traps: **count the actionable elements on screen, and
confirm the same number appear in the accessibility tree.** A button that vanished into a
combined parent is invisible in a diff and obvious in the dump.

## Quality checks

After fixing a file, verify:

1. No combined element swallows a button, link, or tap gesture
2. Every explicit label over combined children still states every fact the element displays
3. Every interactive element has at least `identifier` + `label`
4. Hints appear where the outcome isn't obvious, and nowhere else
5. Every element with state has `value`
6. Decorative elements are hidden, and nothing is announced twice
7. View-level containers have identifiers
8. Identifiers follow the naming convention and are declared once, not duplicated into tests
9. Labels describe meaning, not appearance
10. No duplicate identifiers within the same view
