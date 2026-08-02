# Brief: Track-Info Background Tool

Design brief for a **second tool in this repo** that generates the background
layer sitting *underneath* the IAMSIAM VU visualizer output. The background
carries the track's information (title, artist, release, etc.); the VU sequence
carries the animated logo with its alpha channel.

**Read `iamsiam-vu.html` and this document before designing.** The two tools
share a hard contract — canvas size, frame rate, and the area the logo occupies
— and a background that ignores it will not composite correctly.

---

## 1. What already exists

`iamsiam-vu.html` — a self-contained, dependency-free browser tool that:

- Loads an audio file and analyses it offline with a deterministic FFT
- Turns each horizontal bar of the IAMSIAM ASCII logo into a VU meter that
  reacts side to side
- Exports a **transparent PNG sequence** (straight, non-premultiplied alpha)
  named `<prefix>00000.png`, `<prefix>00001.png`, …
- Writes a companion `<prefix>info.txt` alongside every export

The VU output is the **foreground**. This new tool produces the **background**.

### The `info.txt` contract

Every VU export includes a text file in exactly this shape:

```
IAMSIAM VU — png sequence export
audio file: my_track.wav
frames: 1800 (00000..01799)
fps: 30
size: 1080x1080 (straight alpha)
audio range: 60.000s -> 120.000s

ProRes 4444 with alpha:
  ffmpeg -framerate 30 -i my_track_%05d.png ...
VP9 webm with alpha:
  ffmpeg -framerate 30 -i my_track_%05d.png ...
```

**Strong suggestion:** let the background tool accept this `info.txt` as a drag-
and-drop input and auto-configure itself from it — canvas size, fps, frame count,
and the audio filename all come free, and the two layers can never drift out of
sync. This is the main payoff of keeping both tools in one repo.

---

## 2. What to build

A background generator that outputs a full-frame image (or sequence) matching a
VU export, containing the track's information laid out as a designed composition
rather than a caption slapped in a corner.

### Output requirements

- **Same pixel dimensions** as the VU sequence it pairs with
- **Same frame rate**, if animated
- A **still PNG** is the default and correct choice when the design is static —
  composite one still under the whole VU sequence. Only emit a PNG sequence if
  the background actually moves.
- Opaque background (this is the bottom layer), unless the user explicitly wants
  it to sit over footage — in which case offer alpha as an option.

---

## 3. Safe areas — the hard geometric constraint

The VU logo is centred and scaled to fit the canvas. Its opaque artwork occupies
the box below, measured in output pixels. **Track info must live outside it**, or
it will sit behind the animating bars.

| Canvas | Margin | Logo artwork occupies | Free space (L / R / T / B) |
|---|---|---|---|
| 1920×1080 | 0% | x 546–1373, y 68–1002 | **546 / 547** / 68 / 78 |
| 1080×1920 | 0% | x 126–953, y 488–1422 | 126 / 127 / **488 / 498** |
| 1080×1080 | 0% | x 126–953, y 68–1002 | 126 / 127 / 68 / 78 |
| 1080×1080 | 15% | x 250–829, y 210–863 | 250 / 251 / 210 / 217 |
| 1080×1080 | 25% | x 333–746, y 304–771 | 333 / 334 / 304 / 309 |

Read from this:

- **16:9 (1920×1080)** gives two generous ~546px side gutters. This is the most
  comfortable canvas for an information-rich layout — think a title block in one
  gutter, metadata in the other, logo pulsing between them.
- **9:16 (1080×1920)** gives ~490px bands top and bottom. Natural for
  Reels/TikTok: title above the logo, details below.
- **Square at margin 0%** is nearly full-bleed — there is almost no room. If the
  design is square, the VU tool's **margin** slider must be raised (15–25%) to
  open space, and the brief should state which margin the background assumes.

**Whatever canvas and margin the design targets, record it in the tool's UI and
in its exported info file,** so the VU tool can be set to match.

The logo geometry itself: source art is 1080×1080 with the opaque bounding box
x 126–953, y 68–1002. Scale factor is
`min(W·(1−2m)/1080, H·(1−2m)/1080)`, centred, where `m` is margin as a fraction.

---

## 4. Track information — the data model

See `docs/track-info.example.json` for a filled-in example. Suggested fields,
all optional so the layout must degrade gracefully when any are absent:

| Field | Example | Notes |
|---|---|---|
| `title` | `"Nightdrive"` | Usually the largest type on the frame |
| `artist` | `"IAMSIAM"` | |
| `release` | `"Neon Arterials EP"` | Album/EP name |
| `catalog` | `"IAM-007"` | Catalogue number |
| `date` | `"2026-03-14"` | Release date |
| `bpm` | `128` | |
| `key` | `"F# minor"` | |
| `duration` | `"5:42"` | |
| `credits` | `"Written & produced by …"` | May be multi-line |
| `links` | `"iamsiam.net"` | URL / handle |
| `notes` | free text | Catch-all |

The tool should let these be typed into a form **and** loaded/saved as JSON, so a
whole release's worth of tracks can be batched without retyping.

---

## 5. Suggested architecture

Match the existing tool's conventions so the two feel like one product:

- **Single self-contained HTML file** (e.g. `iamsiam-bg.html`) at the repo root —
  no build step, no dependencies, no network calls, opens straight from disk
- Vanilla JS, canvas-based rendering, same dark control-panel UI language
  (see the `:root` CSS custom properties in `iamsiam-vu.html` and reuse them)
- Left control panel, live preview on the right, export buttons at the bottom
- Export via canvas `toBlob` → download, matching the VU tool's approach; reuse
  its zip writer if a sequence is needed
- Fonts must be **embedded or web-safe** — the tool has to work offline, so no
  Google Fonts links. If a specific typeface is required, embed it as a base64
  data URI the way the logo is embedded in `iamsiam-vu.html`.

Keep `index.html` working as the Pages entry point; consider turning it into a
small launcher linking to both tools rather than a bare redirect.

---

## 6. Decisions to make at the start of the session

These are open and should be settled before designing:

1. **Primary canvas** — 16:9, 9:16, square, or all three as presets?
2. **Static or animated** background?
3. **Aesthetic direction** — the VU tool is monospace/terminal (ASCII bars,
   green-on-near-black UI). Does the background continue that, or contrast
   against it?
4. **Does the background need alpha too**, so the whole stack can sit over
   video footage?
5. **Batch mode** — one track at a time, or feed a JSON array and emit a
   background per track?

---

## 7. Definition of done

- Opens offline from disk, no dependencies
- Produces a background whose dimensions match a VU export exactly
- All track-info fields render legibly and degrade gracefully when empty
- Nothing important falls inside the logo's safe-area box for the chosen canvas
- Exports a companion info file recording canvas size, margin assumption, and
  which track it belongs to — mirroring the VU tool's `info.txt`
- Documented in `README.md` alongside the VU tool
