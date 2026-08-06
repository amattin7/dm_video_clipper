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

## 2026-08-05 — Confirmed the fix; replaced offline-file export with a library + PDF

**Confirmed:** the reload still happens (it's OS-level, as expected), but
IndexedDB persistence is doing its job — the app now resumes cleanly instead
of losing the video/stills.

**New problem surfaced:** the exported standalone "Practice Guide" HTML file
(the original offline-distribution mechanism) doesn't render reliably once
downloaded and reopened on iOS — Safari's handling of a locally-opened
`.html` file is inconsistent about actually running its script (sometimes
just shows a Quick Look-style static preview). The in-app Practice Mode,
served live from the Pages URL, works great by contrast — so the fix is to
lean on that instead of fighting the download-and-reopen path.

**Changes:**
- Removed the "export standalone offline HTML" feature entirely (the Blob
  download of a second self-contained guide file) — superseded by the two
  items below.
- Added a **Saved Guides library**: a new always-visible section above the
  main workflow, backed by a second IndexedDB object store (`guides`,
  bumped `flourish-db` to version 2). "Save to library" prompts for a name
  and stores the current stills as a named, dated guide; the library lists
  every saved guide with thumbnail, still count, rename-in-place, a "View"
  button that opens the same fullscreen Practice Mode overlay, and delete.
  Lives entirely on-device (no accounts, no backend) — chosen over a
  cross-device cloud option since guides only ever need to be viewed on
  the phone they were captured on, and this keeps zero-connectivity
  behavior intact.
- Practice Mode was refactored to take an explicit stills array
  (`openPractice(stillsArr)`) instead of always reading the global working
  set, so it can show either the in-progress captures or a saved guide's
  stills.
- Added **"Save as PDF"**: renders the stills into a hidden print-only sheet
  (`@media print` stylesheet, one still per page) and calls the browser's
  native `window.print()`. On iOS this opens the standard print preview,
  where the share icon offers "Save to Files" as a real, non-interactive
  PDF — which iOS displays natively and reliably, unlike the old downloaded
  HTML file. Zero new dependencies; uses only built-in browser print
  support.

**Still to verify on-device:** that `window.print()` → Share → Save to
Files actually produces a clean multi-page PDF on the user's phone, and
that the saved-guides library survives normal day-to-day use (reopening the
app days later, per the ITP caveat above).

## 2026-08-05 — Blank preview on a cropped/HDR clip

**Reported:** a video cropped/zoomed in Photos loaded successfully (correct
1346×1810 / 18.23s reported) but the preview stayed blank, capture produced
a black image, and the scrub bar never moved regardless of drag position. A
raw, unedited iPhone recording loads fine.

**Diagnosis, in order of what was ruled out:**
1. Not an aspect-ratio/CSS issue — `video{width:100%; max-height:52vh}`
   scales any ratio proportionally; it wouldn't go fully blank over this.
2. Not (only) the "non-destructive crop metadata" theory floated initially
   — re-exporting via Photos Duplicate → Share → Save to Files → Save Video
   still failed, which should have forced a flattened re-render if that were
   the whole story.
3. The real signature: container-level metadata (duration, dimensions)
   parses fine — those come straight from the file header — but the actual
   video track never decodes a single frame in Safari's `<video>` element.
   No `seeked`/`timeupdate` ever fires. Most likely categories: HDR HEVC
   (common default on iPhone 12 Pro+) or variable-frame-rate exports (screen
   recordings in particular) hitting a WebKit decode limitation that native
   AVFoundation-based apps (Photos, QuickTime) don't have.

**Why this can't be "just record with HDR off" going forward:** clips also
come from other people and from iOS screen recordings (e.g. of a YouTube
video), so the source format is often outside the user's control. No
in-page fix is possible for a file the browser's decoder refuses to touch.

**What shipped:** a stall detector — if no frame has decoded ~1.8s after
`loadedmetadata`, show a specific warning instead of a silent blank screen,
naming the one source-agnostic workaround that exists on-device: import the
clip into iMovie, trim it, Share → Save Video. iMovie's export pipeline is
far more tolerant than Safari's `<video>` tag and flattens whatever was
wrong (HDR, VFR, an unfamiliar codec profile from someone else's phone)
into a standard file this page can read. Also added a persistent
width×height×duration readout under the scrub bar generally, so any future
report of "it won't load" comes with real numbers instead of guesswork.

**Separately noted, not yet hit:** screen-recording DRM/copy-protected
video (not plain YouTube) can come out solid black at the recording level —
a different failure mode that no re-export fixes, since the capture itself
never had real pixels.

**Still open:** whether the iMovie re-export workaround actually resolves
this specific clip, and whether the HDR-video-capture toggle (Settings →
Camera → Record Video) is confirmed as a contributing cause for
self-recorded clips going forward.

## 2026-08-06 — Root cause confirmed: iOS Safari-specific, not the app or the file

**Investigation, in the order it played out:**
- Added an automatic canvas-based blank-frame sampler (drawImage the current
  frame into an offscreen canvas, flag if it's ~all-zero). A retest of the
  clip that had originally worked came back black — immediately raising the
  question of whether the sampler itself was the problem.
- Removed the automatic sampler (kept only the passive, non-invasive stall
  detector that watches for zero decode-progress events). Retested — still
  black. Ruled out the sampler as the cause.
- Diffed the full code against the last confirmed-working commit — purely
  additive (diagnostics, footer, meta tags), nothing touching `video.src`,
  canvas capture, or `<video>` CSS. Not a code regression.
- Cleared the saved session, fully closed and reopened the Safari tab, and
  retried the *exact same, unedited* clip that had worked originally.
  Confirmed by the user to be the same file (only one clip of that
  duration, never edited). Still black — ruled out stale IndexedDB/session
  state and file misidentification.
- Confirmed the same clip plays correctly natively in the Photos app. Ruled
  out file corruption and OS/hardware-wide issues (thermal throttling,
  iCloud re-sync damage) — the asset itself is fine.
- Recorded a brand-new test clip on the spot and tried it — also black in
  Safari, with accurate metadata. This is the key result: it's not about
  any particular file's format/HDR/edit history at all. Every video, right
  now, fails to render in Safari's `<video>` element on this phone, while
  Photos' native player handles all of them fine.
- Tried the page in Chrome on the user's PC (different engine — Blink, not
  WebKit — and a different device entirely): **works correctly.**

**Conclusion:** this is a live Safari/WebKit-specific video-rendering
regression on this one iPhone, not a bug in this app and not a bug in any
particular source file. A phone restart (which had previously fixed an
unrelated stuck-picker-circle issue) did not clear it this time. Nothing
in a web page can force a browser to paint video it's refusing to paint —
there is no code-level fix available here. Left unresolved on the user's
end: whether an iOS/Safari software update changes this.

**Consequence — workflow pivot:** rather than keep chasing a platform bug,
the primary editing workflow moves to **desktop Chrome** (confirmed
working, and mouse/keyboard scrubbing is more precise than fighting
pinch-zoom on a phone anyway — tapping our own buttons was already causing
unwanted page-zoom on iOS). The phone's role becomes **viewing only**, via
the Saved Guides library and Practice Mode.

**New features to support that pivot:**
- **Import/Export guide files.** A saved guide can now be exported as a
  small `.flourishguide.json` file (name + stills + labels, images inline)
  via a Blob download, and imported back in via a file picker on the
  Saved Guides section. This is how a guide built on a PC gets onto a
  phone: transfer the file however (AirDrop, iCloud Drive, email, USB),
  then import it into the same hosted app running in Safari there. Unlike
  the original standalone-HTML export, this works on iOS because the
  receiving device is never asked to execute script from a locally-opened
  file — it's just handing a small data file to the already-properly-hosted
  page, which does the actual work. Still no accounts, no backend.
- **Crop tool for captured stills.** Deliberately proposed as a better fix
  than pre-cropping source video (which is what led into the whole HDR
  investigation above): crop happens *after* capture, on a plain JPEG, with
  zero video-decoder involvement, so none of the codec/HDR class of bugs
  can follow it here. A fullscreen overlay shows the still with a
  draggable/resizable crop rectangle (pointer events, so mouse and touch
  both work), Reset/Apply/Cancel controls, and applies via a canvas crop
  mapped from displayed to natural image coordinates. Scoped to the working
  filmstrip for now — not yet available on already-saved library guides.

**Still to verify:** that exported guide files actually import cleanly
across devices in practice, and that the crop tool's drag/resize handles
behave well on both mouse (PC) and touch (phone) input.
