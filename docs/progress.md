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
