# Recording audio from the background (App Intents, Action Button)

**Rule 1: a recorder asks for the microphone only.** Use `.record`. When the start happens in the background and has to mix, use `.playAndRecord` + `.mixWithOthers`, and set the input node's `auAudioUnit.isOutputEnabled = false` before `engine.prepare()`/`start()`. Never add `.defaultToSpeaker` unless something actually plays. At every start, log what was *requested* (category, options, `isInputEnabled`/`isOutputEnabled`, route) as well as what iOS answered.

**Why:** `AVAudioEngine.start()` also starts the input unit's output side. iOS grants a background App Intent *recording*, but grants background *playback* only to an app that was just on screen or already running audio. A fresh or long-idle process is refused with playback errors: `cannotStartPlaying` (561015905, '!pla') at `setActive`, and `'what'` (2003329396) at `engine.start()`. Retries inside the refused attempt never recover it.

**Source:** thnkr.ing `75672dd` (copied `.playAndRecord` + `.defaultToSpeaker` recorder) through `2a2a1da`, 2026-09-21/22: 13/13 first presses after a break refused, all with those two errors. Fixed as in Muesli-HQ/muesli-ios PR #39.

**Rule 2: a configuration change tests a hypothesis only if it can't fail for a documented reason of its own.** Before switching category or options, write down which documented refusal the new setting will itself trigger. For example, `.record` can't mix, so it is refused in the background with `cannotInterruptOthers` (560557684). A lever that fails for that documented reason hasn't tested the hypothesis, so don't treat it as refuting it.

**Source:** thnkr.ing `9c030b3` and `cc70ed6`, both `.record` attempts. They failed with `cannotInterruptOthers` and were read as ruling out "iOS refused the playback half", which was the correct explanation.
