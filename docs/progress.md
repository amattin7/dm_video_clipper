# Progress log

## 2026-08-04 — Initial draft imported, moved to GitHub Pages

**Done:**
- Imported the first-draft single-file tool (built in a Claude.ai artifact)
  as `index.html`, committed to a new GitHub repo, and enabled GitHub Pages
  so it's reachable at a real `https://` URL.

**Why the move to Pages:** the tool was hitting an issue when run inside
Claude's in-app artifact viewer on iOS — selecting a video from the native
photo picker reset the whole page/JS state, even for a 2-second clip (so not
memory/size-related). Leading theory: Claude's in-app webview gets torn down
when iOS hands control to the native picker and back, unlike a real Safari
tab. Moving to a real hosted URL that opens in actual mobile Safari should
avoid that teardown.

**Features already in place (from the first draft):**
- Video file picker (native iOS photo picker via `<input type=file accept=video/*>`)
- Custom scrub slider + 6 nudge buttons (±1s, ±0.1s, ±1/30s "frame")
- Capture button → draws current video frame to canvas → JPEG dataURL
- Reorderable/deletable/relabelable filmstrip of captured stills
- Practice Mode: fullscreen step-through with swipe + prev/next nav
- Export: generates a second standalone offline HTML file (images inlined)
  for on-phone offline use, downloaded via Blob
- Loading-state / timeout warning after video selection (handles iCloud
  videos still downloading, or an unconfirmed picker selection)

**Note on persistence:** the draft still has a `window.storage`-based
session-restore path left over from being built as a Claude.ai artifact.
That API doesn't exist outside Claude's artifact host, so on GitHub Pages
it's a silent no-op — not a bug, just dead weight to swap for real
`localStorage` (or drop) later if we want restore-on-reload to actually work
on the hosted version.

**Next step:** verify on an actual iPhone in mobile Safari (not Claude's
iframe, not a Files-app preview) that picking a video no longer resets the
page state. If it's fixed, that confirms the webview-teardown theory. If the
reset still happens, the picker-handling logic itself needs a rework (e.g.
checking `file.length`, or adding a manual "confirm loaded" step).

## 2026-08-05 — Webview theory disproven; switched to IndexedDB resilience

**Tested:** same reset reproduced on the real Pages URL in actual mobile
Safari, with a 2-second clip. That rules out the "Claude webview gets torn
down" theory — this is a genuine iOS/WebKit behavior, not an artifact-host
quirk.

**Root cause (best-effort diagnosis, not officially documented by Apple):**
iOS Safari has a known habit of silently reloading a tab right after the
native Photos picker hands a *video* (not photo) back to the page. The
picker does an on-device compatibility export of the asset before returning
it, and that — plus the picker UI itself — pushes memory pressure high
enough that iOS jetsam-kills the backgrounded Safari tab process. Control
returns to what looks like the same page, but it's actually a fresh page
load: every JS variable, the `<video>` element, all of it is gone. Matches
what was seen exactly: no error, no warning text, just back to the pristine
"01 Load a clip" screen. Overhead is mostly fixed cost (export + picker
memory footprint), which is why duration barely matters.

**Decision:** rather than add a backend (would mean the video briefly
leaves the device to upload, breaking the original fully-offline/no-deps
intent for the editing step — ruled out for now), stayed fully client-side
and made the reload survivable instead of trying to prevent it:

- Replaced the dead, Claude-artifact-only `window.storage` calls with real
  **IndexedDB**, which — unlike `localStorage` — can hold actual Blobs, not
  just strings.
- The picked video `File` is now written to IndexedDB **immediately** on
  selection, before anything else happens with it (before `video.src` is
  even set). That's the moment most likely to precede a jetsam reload, so
  it's the moment that most needs to already be saved.
- Captured stills, label edits, reordering/deletion, and scrub position are
  all persisted (debounced) the same way they were before, just to
  IndexedDB instead of the dead API.
- On every page load, the app now checks IndexedDB first: if a video and/or
  stills are found, it auto-restores them (recreates the object URL from the
  saved Blob, restores scrub position, re-renders the filmstrip) and shows
  a "Recovered … after an interruption" status line — instead of forcing
  the user to notice the reset and start over.
- Also fixed an unrelated latent bug found while in there: the file's real
  closing `</script>` tag had been accidentally escaped as `<\/script>`
  (copy-paste artifact from escaping the *nested* script inside the
  exported Practice Guide template). Harmless in practice since nothing
  meaningful followed it, but technically malformed — now a plain
  `</script>`.

**Still to verify on-device:** whether the reload itself still happens
(likely — it's OS-level, outside our control) but now resolves itself
automatically instead of losing the video/stills. If IndexedDB writes
aren't completing before the jetsam kill (i.e. the reload happens too fast
even for the "save immediately on selection" write), the auto-resume won't
have anything to recover — that's the case to watch for in testing.

**Known caveat, not yet a problem:** Safari's Intelligent Tracking
Prevention can clear script-writable storage (including IndexedDB) for
origins the user hasn't interacted with in 7 days. Not a concern for active
use of the tool; only relevant if someone loads the page, walks away for a
week, and expects an in-progress session to still be sitting there.
