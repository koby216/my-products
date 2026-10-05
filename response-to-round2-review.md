# Response to Round 2 Review (5 Oct 2026)

Thanks for this — the two critical regressions are real and well-diagnosed. Went through the whole document against current source on `experiment/on-device-whisper` before responding. Summary below, organized to match your structure.

## Crucial now — confirmed, not yet fixed

**#53 — wake-word interrupt destroys saved messages.** Confirmed exactly as described. `cancelEverythingToIdle()` is shared by the Cancel button and the wake-word-interrupt path, and it unconditionally clears + marks-handled both `pendingMessageQueue` and `dndQueue`. Agreed diagnosis: two different intents sharing one function. Fix plan: add a `clearQueues: Boolean = true` parameter, default true (Cancel button keeps today's behavior), pass `false` from the wake-word-interrupt call site (`SmileyService.kt:1573`) so it only stops speech/current action.

**#54 — any cancel permanently kills the car-audio watchdog.** Confirmed. `carAudioWatchdog` is a self-reposting `Runnable` (`postDelayed(this, 1000L)` at the end of `run()`), started exactly once on Bluetooth connect (`onAudioDevicesAdded`, ~line 4695). `cancelEverythingToIdle`'s `mainHandler.removeCallbacksAndMessages(null)` wipes it along with everything else, and since nothing else restarts it, the keepalive dies until the next BT disconnect/reconnect. Fix plan: re-post `carAudioWatchdog` at the end of `cancelEverythingToIdle` when `carModeActive`.

Both live in the same function — fixing together in one change once Koby signs off on the approach (asked him, waiting on go-ahead as of this message).

**#56 — messages not announced.** This is very likely resolved, not just logged-for-next-time. What actually happened, traced live with the user today: the `EXTRA_PROGRESS_MAX` filter (added to ignore WhatsApp's chat-backup-progress notification) turned out to match real message notifications too — confirmed with direct evidence, `WA_SKIP_PROGRESS` fired on `flags=512`, which is WhatsApp's normal per-message summary flag, not anything backup-specific. Filter was reverted entirely (`7a32c4c`). User re-tested immediately after and messages are being announced again. The `WA_NOTIF_SEEN` / `NOTIF_LISTENER_CONNECTED` logging from Batch A stays in place either way — it's what let us catch this in the first place, and it's cheap insurance if it recurs. Original backup-notification-flooding theory itself was never independently confirmed; if it resurfaces, next step is reading an actual captured backup notification's extras before writing any filter, not guessing again.

**#36 — VAD.** Agreed priority, unchanged status: can't be done in this environment (no way to source/bundle/test a Silero VAD ONNX model here). Needs you or someone with a full Android Studio + model-sourcing setup.

## Corrections — these already landed, doc may be stale on them

- **#41** (leading `00` in `toInternational`) — already fixed, `SmileyService.kt:3068`, dated 2026-10-04.
- **#42** (non-digit stripping on contact numbers) — already fixed, `SmileyService.kt:2810-2820`, dated 2026-10-04.
- **#12** (`GET_RECENT_MESSAGES` unhandled) — already wired, `"GET_RECENT_MESSAGES" -> respondWithRecentMessages()`.

Everything in Batch A2, Batch B, Batch B2, and Batch C2 (#6/#9/#32/#31) that your doc lists as still-open is confirmed already landed against current source — #38/#39 whole-word matching, #1/#45/#46/#48/#47/#2/#4/#5/#3/#18, the car whitelist (#50), relay auth + rate limit + kill switch + daily cap, Whisper fallback, storage check. Batch D items #15/#13/#43/#44/#21 are also done.

## Still open, not yet started

- **#55** — barge-in delay (3500 + 250/word) may have overcorrected. Agreed low priority; will check against a real log before touching it again, and agree it's moot if #49 lands.
- **#29** — the two dead read-toggles (`isWhatsappReadEnabled`/`isSmsReadEnabled`) still have no UI. Need Koby's call: expose or delete.
- **#14** — `carModeMinScore` (0.19) vs wake threshold (0.18) is worth revisiting now that B2 means car mode only fires on an actual car.
- **#17** — `silentlyAbandonCommand` no longer matches its name (it speaks now). Cosmetic rename, not yet done.
- **#16** — `ACTION_DND_OFF` still logs nothing.

## Needs Koby, not code

- **#20** — privacy policy page: content is ready (`store-listing/privacy-policy.md`), converted to HTML and pushed to a `gh-pages` branch on `Smiley`, but that repo is private so Pages won't serve it. Needs a small separate public repo (Koby's call on timing — he asked to hold this until closer to store submission).
- **#22** — Soniox audio retention policy, still needs confirming directly with them.
- **#11** — branch merge/rename, still pending Koby's decision.
- **#33** — real RAM distribution across current testers, and whether a Hebrew Whisper model smaller than 834MB exists — need Koby's input/research.
- **#28** (always-on message announcements) and the beep-vs-spoken-wake question (#49) were both put to Koby directly on 5 Oct — he chose to keep current behavior on both, so these are closed as product decisions, not gaps.

Waiting on Koby's go-ahead to implement #53/#54 now.

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
