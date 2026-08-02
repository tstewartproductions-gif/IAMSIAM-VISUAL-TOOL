# IAMSIAM VU — technical notes

How `iamsiam-vu.html` works and why it was built the way it was. This is a
reference for anyone extending it or building a companion piece that has to
line up with its output.

---

## What it is

A single self-contained HTML file. No build step, no dependencies, no network
requests — it opens straight from disk or from a static host. The IAMSIAM ASCII
logo is embedded in the file as a base64 data URI, so the tool is one artifact
you can copy anywhere and it still works offline.

It loads an audio file, analyses it, animates each horizontal bar of the ASCII
logo as a sideways VU meter, and exports the result as a transparent PNG
sequence.

---

## How the logo becomes meters

The tool doesn't hardcode any knowledge of the artwork. It reads the PNG's
**alpha channel** and derives the bars from it:

1. Find every row of pixels containing at least one opaque pixel
   (alpha > 16), and group consecutive such rows into horizontal **strips**.
   Each strip is one line of the ASCII art.
2. Within each strip, scan columns for runs of opaque pixels to get **segments**.
3. Merge segments separated by less than `0.6%` of image width — this closes
   the small internal gaps inside a single glyph run so it reads as one bar.
4. Discard anything narrower than `0.3%` of image width, which removes specks.

For the bundled logo this yields **61 bars across 28 rows**. Because it's all
derived from alpha at load time, swapping in any other line-based PNG with
transparency works with no code changes.

Each bar is stored as `{row, y, height, x0, x1}` — its strip index and its
pixel box.

---

## Audio analysis

### Why not the Web Audio AnalyserNode

The obvious approach — a real-time `AnalyserNode` sampled once per animation
frame — was rejected. It returns whatever the audio thread happened to produce
at the moment you asked, which means results aren't reproducible: the exported
frames wouldn't match what you previewed, and two exports of the same track
wouldn't match each other.

Instead the file is decoded to a mono `Float32Array` and analysed **offline and
deterministically**. For each output frame the tool computes a windowed FFT
centred on that frame's exact sample position. Same audio plus same settings
always produces byte-identical frames, and the preview is a truthful
representation of the export.

### The analysis pass

- 4096-point radix-2 FFT, hand-written (no library), with a Hann window
- Frequency range 35 Hz to `min(16 kHz, sampleRate × 0.45)`
- That range is divided into **log-spaced bands, one per logo row**, so the band
  count follows the artwork rather than being fixed
- Per frame, per band, it stores raw power in dB

### Two-stage design

The expensive FFT pass runs **once** and caches raw dB values. A second, cheap
pass turns those into the 0–1 levels that drive the bars, applying:

- **Normalisation** over a 48 dB window. With per-band auto-levelling on, each
  band references its own peak — floored at `globalPeak − 30 dB` so a
  near-silent band doesn't get amplified into noise.
- **Gain** and a **contrast** exponent
- **Attack / release** via one-pole smoothing, with coefficients derived from
  the frame duration (`1 − e^(−dt/τ)`) and applied asymmetrically depending on
  whether the level is rising or falling

This split is why the meter sliders respond instantly — moving gain or release
re-runs only the cheap stage. Changing frame rate or loading new audio is what
triggers a full re-analysis.

---

## Rendering

Bars are drawn by blitting **slices of the actual artwork**, not by filling
rectangles: each bar's visible portion is a source-rect `drawImage` from an
offscreen copy of the logo. So the bars keep the real glyph edges. The anchor
setting decides which edge the slice grows from — centre, left, or right.

Peak-hold ticks are thin rectangles tracking a per-bar maximum that decays at a
configurable rate, mirroring the falling indicators on a hardware meter.

### Preserving alpha

- The canvas is **never filled with a background** — every frame starts with
  `clearRect` and only the bars are drawn.
- Tinting is done on an offscreen canvas using a `source-in` composite, which
  recolours the artwork while leaving its alpha shape untouched.
- Export is `canvas.toBlob('image/png')`, which yields **straight
  (non-premultiplied) alpha**.

The checkerboard behind the preview is a CSS backdrop on the container. It is
not part of the canvas and never appears in exports.

---

## Layout geometry

The logo is scaled to fit the output canvas and centred:

```
scale   = min( W·(1−2m)/1080 , H·(1−2m)/1080 )   // m = margin as a fraction
originX = (W − 1080·scale) / 2
originY = (H − 1080·scale) / 2
```

Source artwork is 1080×1080 with an opaque bounding box of **x 126–953,
y 68–1002**. Applying the above gives where the artwork actually lands:

| Canvas | Margin | Artwork occupies | Free space (L / R / T / B) |
|---|---|---|---|
| 1920×1080 | 0% | x 546–1373, y 68–1002 | **546 / 547** / 68 / 78 |
| 1080×1920 | 0% | x 126–953, y 488–1422 | 126 / 127 / **488 / 498** |
| 1080×1080 | 0% | x 126–953, y 68–1002 | 126 / 127 / 68 / 78 |
| 1080×1080 | 15% | x 250–829, y 210–863 | 250 / 251 / 210 / 217 |
| 1080×1080 | 25% | x 333–746, y 304–771 | 333 / 334 / 304 / 309 |

Worth knowing: at 16:9 the square logo leaves two ~546 px side gutters, at 9:16
it leaves ~490 px bands top and bottom, and on a square canvas at 0% margin it
is close to full-bleed with almost no clear space. The margin slider is the
lever for opening room around it.

---

## Export

Two paths, because they have different failure modes:

- **Folder** — uses the File System Access API (`showDirectoryPicker`) to stream
  each frame to disk as it is rendered. Memory stays flat, so long sequences are
  fine. Chromium-based browsers only.
- **ZIP** — accumulates frames in memory and packages them with a hand-rolled
  ZIP writer. Uses the *store* method with CRC32 and no compression, since PNGs
  are already compressed. Works everywhere, but memory grows with frame count.

Rendering yields to the event loop periodically, which keeps the UI responsive,
lets the progress bar update, and makes the cancel button work mid-export.

Frames are named `<prefix>00000.png`, `<prefix>00001.png`, … and are always
numbered **from zero regardless of the export range**, so a mid-track section
imports cleanly as an image sequence. The prefix auto-fills from the audio
filename unless the user types their own.

### The companion `info.txt`

Every export writes a text file recording what was rendered:

```
IAMSIAM VU — png sequence export
audio file: my_track.wav
frames: 1800 (00000..01799)
fps: 30
size: 1080x1080 (straight alpha)
audio range: 60.000s -> 120.000s

ProRes 4444 with alpha:
  ffmpeg -framerate 30 -i my_track_%05d.png -c:v prores_ks -profile:v 4444 -pix_fmt yuva444p10le my_track_alpha.mov
VP9 webm with alpha:
  ffmpeg -framerate 30 -i my_track_%05d.png -c:v libvpx-vp9 -pix_fmt yuva420p -b:v 0 -crf 24 my_track_alpha.webm
```

Since frame numbering restarts at zero, the `audio range` line is what ties a
partial export back to its position in the track.

---

## Constraints that shaped it

- **Offline and self-contained** — no CDN, no fonts fetched over the network, no
  build tooling. Assets are embedded as data URIs.
- **Deterministic** — reproducibility was prioritised over the simpler real-time
  analysis approach.
- **Artwork-driven** — bar layout and band count come from the image, not from
  constants, so the tool isn't tied to one logo.
- **Alpha throughout** — nothing in the pipeline composites against a background.
