# DM Video Clipper (Flourish Breakdown)

Self-contained, single-file HTML/CSS/JS tool for drum major mace flourish
practice. Load a video, scrub through it with fine nudge controls, capture
stills at chosen frames, crop in on a detail if needed, and arrange/caption
them into a step sequence. Save named guides to an on-device library for
fullscreen, swipeable practice viewing anytime — works with zero
connectivity — export a still-by-still PDF to share or print, or export a
guide file to move it to another device (e.g. built on a PC, viewed on a
phone).

No build step, no dependencies, no backend.

**Note:** as of 2026-08-06, video loading/preview is known to be broken in
iOS Safari specifically (a live platform bug, not something fixable from
here — see [docs/progress.md](docs/progress.md)). The recommended workflow
is to build guides on desktop Chrome, then export/import the guide file to
view on a phone.

## Live version

Deployed on Vercel from `index.html` on the `main` branch (auto-deploys on
every push, no build step):
https://dm-video-clipper.vercel.app/

(Previously hosted on GitHub Pages; moved to Vercel on 2026-08-06 after
GitHub's Pages builder repeatedly failed to deploy — see
[docs/progress.md](docs/progress.md). Pages is now disabled for this repo.)

## Files

- `index.html` — the tool itself (open directly in a browser, or use the
  Vercel URL above on a phone).
- `Initial_notes.txt` — original working notes from the first drafting
  session (kept for history).
- `docs/progress.md` — running log of what's done and what's next.

## Status

See [docs/progress.md](docs/progress.md) for current state and open issues.
