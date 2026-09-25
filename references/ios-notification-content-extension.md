# What closes an expanded notification, and what does not

**Rule: inside an expanded notification, reading back through content belongs to controls, not to a
drag.** A downward drag is judged by the system's outer platter, from how far it went and how fast
(`PLExpandedPlatterPresentationView.scrollViewDidEndDragging:willDecelerate:`, read out of the
shipped binary). A long fast one closes the notification; a short or slow one reaches your scroll
view. Nothing inside the extension changes where that line falls, so give the user paging controls
(one page per tap, disabled at each end) and treat dragging as a bonus.

**Measured**, iPhone 17 Simulator, iOS 27, 2026-09-24, one long fast downward flick against each
arrangement of the same panel:

| Arrangement | Long fast flick down |
| --- | --- |
| SwiftUI `ScrollView` | closes the notification |
| Plain `UITableView` | closes the notification |
| `extensionContext.notificationActions = []` (no action rows) | closes the notification |
| 300-point panel | closes the notification |
| 900-point panel | closes the notification |
| Reply field focused, keyboard up | closes the notification |
| Any of the above, short or slow drag | scrolls, panel stays |

**Two wrong conclusions this table exists to prevent**, both of which shipped:
1. *"The action buttons under the panel are stealing the drag."* They are not; removing them changes
   nothing. (It is still worth dropping rows the panel duplicates, for height and for not showing
   the same control twice.)
2. *"Focusing the reply field fixes it, which is why Messages scrolls."* It does not. That run's
   drag was shorter, which is the whole difference. **When a variant survives, check that the
   gesture was identical** — measure the drag in screen points, not as a fraction of an element
   whose size your change just altered. Two rounds here were wasted on exactly that: a
   "panel-relative" drag turned out to be measured against a 66-point header, so it was a 34-point
   nudge, not a flick.

**Messages is not privileged, and is worth copying anyway.** It uses the same public extension point
(`com.apple.usernotifications.content-extension`, `CKMessagesNotificationViewController`), and its
ChatKit controller calls `grabFocus` from both `didReceiveNotification:` and `viewDidAppear:` — the
two-call technique WWDC16's *Advanced Notifications* shows — so it opens with the reply field
focused and no action rows. Match that for the reply experience; do not claim it as a scrolling fix.
If you do focus the field, ask only for the height left above the keyboard: a panel that requests
more than the host has is clipped **from the top**, which silently costs the header and its controls.

**Still unexplained:** the user reports Messages' own expanded notification scrolling smoothly on a
real iPhone. Simulator flicks are synthesized and this table may not transfer to a finger; the
Simulator also would not expand Messages' own notification, so Apple's panel could not be run
through the same harness. Do not resolve that gap by asserting either way.

**Source:** thnkr.ing `fa32cd3` (paging marks), `774a7a5` (keyboard-first, claim withdrawn), after
`9f54b66` shipped a guess. Investigations: GPT-6 Sol, then GPT-6 Astra read-only on the binaries.
