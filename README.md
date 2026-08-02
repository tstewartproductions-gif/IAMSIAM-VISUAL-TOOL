# IAMSIAM · VU

An audio-reactive VU-meter visualizer built from the IAMSIAM ASCII logo.
Every horizontal bar of the ASCII art becomes a VU meter that reacts **side to
side** to whatever audio you load, and the result exports as a **PNG sequence
with a built-in alpha channel** — ready to drop over any footage in your
editor or compositor.

Everything runs in the browser. No install, no server, no dependencies.

## Quick start

1. Open `iamsiam-vu.html` in Chrome or Edge (Firefox/Safari work too, but only
   Chrome/Edge support exporting straight into a folder — others use ZIP).
2. Drop an audio file anywhere on the window (or use **LOAD AUDIO**).
   mp3 / wav / flac / ogg / m4a all work.
3. Press **space** or ▶ to preview. The checkerboard behind the logo is the
   transparency — it is not part of the export.
4. Hit **EXPORT SEQUENCE → FOLDER** (or → ZIP) to render the PNG sequence.

## How it reacts

The tool scans the logo's alpha channel and detects every horizontal ASCII
bar (61 bars across 28 rows in the bundled logo). Each row is assigned a
frequency band — lows at the bottom, highs at the top by default — and each
bar's width breathes with the level of its band, growing and shrinking
horizontally like a meter laid on its side.

The audio analysis is done offline with a deterministic FFT over the decoded
file, so the exported frames always line up with the audio exactly — what you
preview is what you render, and re-renders are identical.

## Controls

| Section | What it does |
|---|---|
| **grow from** | Bars extend from their center (default), left edge, or right edge |
| **react to** | Per-row frequency bands, or the whole logo pumping to overall loudness |
| **low freq at top** | Flips the frequency-to-row mapping |
| **idle width** | How much of each bar stays visible in silence (0% = logo vanishes) |
| **gain / contrast** | Overall drive and response curve of the meters |
| **attack / release** | How fast bars jump out and how slowly they fall back |
| **auto-level bands** | Normalizes each band to its own peak so every row dances regardless of the mix |
| **peak-hold ticks** | Classic floating peak markers that decay at the "peak fall" rate |
| **tint color** | Recolor the logo (export keeps the alpha either way) |

Replace the logo with any other PNG that has an alpha channel — the bar
detection is automatic, so any similar line-based artwork works.

## Output

- PNG sequence, straight (non-premultiplied) alpha, named `iamsiam_00000.png`,
  `iamsiam_00001.png`, …
- Set resolution, frame rate (24/25/30/50/60), export range, and filename
  prefix in the **OUTPUT** panel. The prefix auto-fills from the loaded
  audio's filename (`my_track.wav` → `my_track_00000.png`) until you type
  your own; clear the field to re-enable auto-naming.
- **EXPORT → FOLDER** streams frames straight to disk (Chrome/Edge) — use this
  for long tracks. **EXPORT → ZIP** builds everything in memory first, best
  for shorter clips. **EXPORT CURRENT FRAME** saves a single PNG for a quick
  alpha check in your compositor.
- Every export includes an `info.txt` with the settings used and ready-made
  ffmpeg commands.

### Turning the sequence into a video with alpha

```sh
# ProRes 4444 (Premiere / Final Cut / DaVinci)
ffmpeg -framerate 30 -i iamsiam_%05d.png -c:v prores_ks -profile:v 4444 -pix_fmt yuva444p10le iamsiam_alpha.mov

# VP9 WebM (web)
ffmpeg -framerate 30 -i iamsiam_%05d.png -c:v libvpx-vp9 -pix_fmt yuva420p -b:v 0 -crf 24 iamsiam_alpha.webm
```

Import the PNG sequence directly into After Effects / Premiere / Resolve as an
image sequence and the alpha comes with it.

## Files

- `iamsiam-vu.html` — the whole tool (logo embedded, works offline)
- `index.html` — redirect stub so the GitHub Pages root URL still works
- `assets/IAMSIAM_ASCII.png` — the source ASCII logo
