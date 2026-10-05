# Smiley – Changes Since the Code Review (2026-10-04 → 2026-10-05)

Branch: `experiment/on-device-whisper` on `koby216/Smiley`
Backup branch (pre-Batch D snapshot): `backup/pre-batch-d-2026-10-05`
versionCode progressed: 204 → 215

All of this builds on your "Smiley Fix Queue" review (52 issues, batches A/A2/B/B2/C/C2/D + the car-whitelist idea). Thanks for that — it was accurate end to end. Here's what's shipped since, batch by batch, each verified against the real source before being implemented.

## Batch A — Diagnostic logging
- Event log now survives service restarts (was in-memory only).
- Crash/device-info reporting carries real device model, Android version, RAM, battery-optimization exemption, and permission state, sent to the relay's `log-crash` endpoint — so a tester's report is diagnosable without Logcat access.

## Batch A2 — Contact-name matching
- `isHardCancel`/`isNewSearch`/`isNextPage` switched from raw substring matching to the existing whole-word matcher (`matchesWordList`, already used by `isCancel`/`isYes`) — fixes common Hebrew names breaking contact selection when a name happened to contain a command substring.

## Batch B — 6 bugs from the 4 Oct user log, + universal wake-word interrupt
- Fixed the cancel-ignored-on-incoming-message bug family (4 related fixes).
- CONFIRM/CONFIRM_CALL no longer cancel on a valid yes/no answer.
- Yes/no confirmation questions get as much time to answer as dictation does.
- Mic-capture failures are now visible in the shareable log, not just Logcat.
- "Say wake word mid-command" now re-says "אני מקשיבה" and resets the timer instead of silently ignoring it.
- Generalized the wake word into a universal interrupt: saying "היי סמיילי" while any other stage is active (not just COMMAND) now stops everything and starts a fresh conversation, via a new shared `cancelEverythingToIdle()` helper.
- Added `MessageMatchingTest.kt` (JUnit/Robolectric) covering the real failing transcript and the Hebrew-name whole-word-matching cases.

## Batch B2 — Car Bluetooth whitelist
- New screen, "הרכבים שלי" (`CarDevicesActivity`): lists all paired Bluetooth devices, user checks which ones are actually their car(s).
- `isKnownCarAudioDevice()` checks this list; if it's empty (user hasn't opened the screen), falls back to the old behavior (any Bluetooth audio device = car) so nothing breaks silently for existing users.
- Sort hint only, never a decision: a device whose Bluetooth class reports `AUDIO_VIDEO_CAR_AUDIO` sorts near the top, user can still uncheck it.
- 2026-10-05 addition (Koby's follow-up): most users would never discover this screen unprompted, but announcing it on every drive would be annoying, and a user with only one Bluetooth audio device has no ambiguity to resolve. So we now track distinct device addresses seen connecting, and speak a one-time hint pointing to the screen only once at least two different devices have connected and the user still hasn't set up a whitelist.

## Batch C — Car-mode VAD stopgap
- Raised `continueSpeechThreshold` in `SonioxListener` from 450ms to 650ms in car mode, to reduce premature cutoff mid-sentence.
- Full VAD (a proper Silero VAD ONNX model for real noise rejection) is **not done** — this environment can't download/bundle/test a model. Left for whoever has a full Android Studio + model-sourcing setup.

## Batch C2 — Resilience
- Server kill-switch + daily request cap (`SERVICE_ENABLED`, `DAILY_REQUEST_LIMIT`) in `soniox-relay/server.js`, hard 503 at the top of both the HTTP and WS upgrade paths.
- `WhisperListener` now falls back to `SonioxListener` automatically on a Whisper engine error (`onFallbackNeeded` callback), instead of going silent. `SmileyService.sonioxListener` converted from a fixed `by lazy val` to a swappable backing field to support this without touching existing call sites.
- `WhisperListener.hasEnoughStorage()` checks available disk space before downloading the ~874MB model.

## #6 — Relay authentication
- `soniox-relay`: shared-secret header (`x-smiley-secret`) + per-IP rate limiting (`RATE_LIMIT_PER_MINUTE`), staged via `SHARED_SECRET_MODE` (`log` vs `enforce`), same pattern as the existing `APP_CHECK_MODE`.
- Secret is baked into the app via `BuildConfig.RELAY_SHARED_SECRET`, sourced from a gitignored `relay_secret.properties` (mirrors the existing `keystore.properties` pattern) — **never committed to git**. (We did have a near-miss here: it was briefly hardcoded in source and the push was blocked by an automated credential-leak check before it ever reached GitHub; fixed by switching to this file-based approach.)

## Batch D — approved cleanup items
- DND message readback capped at 15 messages (was unlimited — "say yes to read 45 messages, get all 45").
- Removed dead SMS-permission broadcast code (`ACTION_REQUEST_SMS_PERMISSION` and its call sites) — SMS compose was already fully disabled upstream.
- ProGuard: `-keepattributes SourceFile,LineNumberTable` (readable crash traces) + a keep rule for `dev.ffmpegkit.whisper.**` (JNI lookups), mirroring the existing ONNX Runtime rule.
- Corrected `planning/premium-ai-plan.md`: it referenced Vertex AI/Gemini, but `server.js` actually calls Anthropic's API directly with `claude-haiku-4-5-20251001`.

**Deferred, need your input, not done:** merging/renaming the `experiment/on-device-whisper` branch; restoring `isTrialExpired()` (intentionally disabled for testers since Sept 30); a public privacy-policy page before Play Store submission; confirming Soniox's audio retention policy.

**Two product decisions** (Koby's call, no code changed): messages still announce regardless of lock state; wake-word acknowledgment stays spoken ("מקשיבה"), not a beep.

## Post-Batch-D fixes (today)
- **WhatsApp backup-notification bug**: WhatsApp posts its own chat-backup-to-Drive progress notification from the same `com.whatsapp` package, with a rising percentage. Each percentage tick looked like new message text, so it bypassed duplicate detection and got announced as an incoming message — including right after the user said "cancel," and piling up in both `pendingMessageQueue` and `dndQueue` while Do Not Disturb was active. A real WhatsApp message notification never carries a progress bar, so `NotificationListener.onNotificationPosted` now skips anything with `Notification.EXTRA_PROGRESS_MAX` before it's ever treated as a message.
- **Build break fix**: `cap_car_desc` in `strings.xml` had an unescaped apostrophe in "בלוטות'" — valid plain XML (passes `xmllint`), but AAPT2's resource compiler requires it escaped as `\'`, same as every other string in that file. Crashed `mergeDebugResources` with `TableExtractor.extractResourceValues`. Fixed.

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
