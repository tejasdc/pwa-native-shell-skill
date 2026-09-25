# A notification content extension cannot win a downward drag

**Rule: inside an expanded notification, reading back through content belongs to controls, never to
a scroll gesture.** A fast downward flick anywhere in the custom view is iOS's own "put this
notification away" gesture, and iOS takes it. Put paging marks (one page per tap, disabled at each
end) in the view and treat dragging as a bonus that works when the user happens to be slow.

**Measured**, iPhone 17 Simulator, iOS 27, 2026-09-24, four variants of the same panel:

| Gesture inside the panel | Result |
| --- | --- |
| Fast flick up | scrolls, panel stays |
| Slow drag down (≈0.7 s press first) | scrolls, panel stays |
| Fast flick down | **notification dismissed, every time** |

The fast flick down dismissed with a SwiftUI `ScrollView`, with a plain `UITableView`, with the
category's three actions showing, and with `extensionContext.notificationActions = []`. So neither
the UI framework nor the action rows under the panel are the cause, and no in-process change fixes
it: the extension is hosted remotely by SpringBoard, gesture arbitration happens there, and Apple
documents no way to make the system's recognizer wait for the extension's scroll view (checked by
an independent GPT-6 Sol investigation, which found the same silence).

**What this also means:** a user who reports "scrolling sometimes dismisses it" is describing
velocity, not randomness — slow drags survive, so it feels intermittent and like a "special
manoeuvre". Do not spend a build on bounce behaviour, `delaysContentTouches`, gesture delegates or
`require(toFail:)`; none of them can reach the other process's recognizer.

**Two things worth doing anyway, for height rather than for the gesture:**
- `NSExtensionContext.notificationActions` can be filtered while the panel is showing, so actions
  the panel already offers as its own controls stop taking a row beneath it. The category keeps
  registering them for other surfaces (a paired Watch shows the category's actions, not the
  extension's).
- Keep the reply controls in a fixed row outside the scrolling content; inside it they scroll away
  exactly when the user wants them.

**Source:** thnkr.ing `fa32cd3`, after `9f54b66` (`.scrollBounceBehavior(.always)`) shipped on the
guess that the scroll view could claim the drag, and his second report of the same bug.
