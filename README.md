# Fixing mains-light flicker in video (ffmpeg)

How I removed the flicker from a DJI 4K night clip shot at 29.97 fps under Indian
50 Hz lighting, without blurring motion. Everything needed to repeat it is here, so
no more research.

---

## TL;DR: just run this

```zsh
./deflicker.sh DJI.MP4 out.mp4                 # whole video
./deflicker.sh DJI.MP4 out.mp4 180             # from 3:00 to the end
./deflicker.sh DJI.MP4 test.mp4 420 10         # 10 s test from 7:00 (always test first!)
```

The full command, if you want to type it yourself:

```zsh
S=40; FC="[0:v]format=yuv420p10le,setpts=PTS-STARTPTS,split=3[a][b0][c0];\
[b0]gblur=sigma=${S}[b];\
[c0]tmix=frames=3:weights='1 1 1',trim=start_frame=1,setpts=PTS-STARTPTS,gblur=sigma=${S}[c];\
[a][b]blend=all_expr='clip(A-B+512,0,1023)':shortest=1[hp];\
[c][hp]blend=all_expr='clip(A+B-512,0,1023)':shortest=1[out]"

ffmpeg -hwaccel videotoolbox -ss 180 -i DJI.MP4 \
  -filter_complex "$FC" -map "[out]" -map 0:a:0 -map_metadata 0 \
  -c:v libx265 -preset fast -b:v 69848k -maxrate 90000k -bufsize 140000k \
  -pix_fmt yuv420p10le -x265-params log-level=error -tag:v hvc1 \
  -c:a copy -shortest -movflags +faststart out.mp4
```

---

## 1. Why the video flickers

- Mains in India is **50 Hz**, so lights pulse at **100 Hz** (twice per cycle).
- The camera recorded at **29.97 fps**.
- 100 ÷ 29.97 = **3.336** light pulses per frame. The leftover 0.336 of a pulse
  means each frame catches the light at a different brightness.
- That leftover adds up to a whole pulse about every **3 frames**, so the brightness
  pattern repeats every 3 frames. That's a visible **~10 Hz flicker**.

**Key insight:** if the pattern repeats every 3 frames, averaging 3 frames cancels it
completely.

### Working out the frame count for other cameras or countries

```
light_hz = 2 × mains_hz                (India/EU 50 Hz → 100; US 60 Hz → 120)
beat_hz  = |light_hz − round(light_hz / fps) × fps|
frames   = round(fps / beat_hz)        → use as tmix=frames=N
```

| Mains | fps | Pulses per frame | Beat | `tmix frames` |
|---|---|---|---|---|
| 50 Hz | 29.97 / 30 | 3.33 | ~10 Hz | **3** |
| 50 Hz | 59.94 / 60 | 1.67 | ~20 Hz | **3** |
| 50 Hz | 23.976 / 24 | 4.17 | ~4 Hz | **6** |
| 50 Hz | 25 / 50 | 4 / 2 (whole number) | none | no flicker |
| 60 Hz | 25 | 4.8 | ~5 Hz | **5** |

**Prevent it next time:** in India, shoot at **25 or 50 fps**, or lock the shutter to
**1/50 or 1/100 s**. Then there's no flicker to fix.

---

## 2. Why plain averaging isn't enough, and the trick that fixes it

`tmix=frames=3` alone kills the flicker, but anything that moves gets
**ghosted or blurred**, because 3 frames are blended together.

The fix: **flicker only lives in the broad brightness, not in the fine detail.** So
split each frame into two layers:

| Layer | How it's made | Taken from |
|---|---|---|
| **Low frequency** (broad brightness, where the flicker is) | `gblur sigma=40` | the **3-frame average**, so the flicker is gone |
| **High frequency** (edges, texture, detail) | frame − its blur | the **current frame only**, so it stays sharp |

```
out = blur( tmix3(frame) )  +  ( frame − blur(frame) )
```

In 10-bit, the "detail" layer is stored offset by 512 (mid-grey) so negative values
survive. That's what `A-B+512` and then `A+B-512` do in the two `blend` steps.

`trim=start_frame=1` on the tmix branch centres the 3-frame window on the current
frame (frames n−1, n, n+1) instead of lagging one frame behind.

**Measured on the 7:00 test clip** (frame-to-frame brightness change, 10-bit scale):

| | Average jump | Worst jump |
|---|---|---|
| Original | 7.3 | 21 |
| This method | **1.8** | **8.5** |

The ~10 Hz flicker component went from strong to zero.

### Tuning `S` (blur sigma)

- **40** worked best here.
- **Lower (20):** follows smaller flicker regions more tightly, but more of the
  averaging (and any ghosting) shows up in the image.
- **Higher (60+):** sharper motion, but flicker in smaller areas may come back.
- Known limit: **small bright lights** (street lamps, string lights) count as
  "detail", so they come from the current frame and can still twinkle a little.

---

## 3. What didn't work (don't try these again)

| Tried | Result |
|---|---|
| `deflicker=size=61:mode=am` (ffmpeg's built-in deflicker) | Basically no improvement. It evens out slow overall brightness, not a fast 3-frame pattern. |
| `tmix=frames=3` alone | Flicker gone, but moving things ghost and blur. |
| Hardware encoder `hevc_videotoolbox` | Only ~1.6× faster, because the filter is the slow part, not the encoder. Its quality wasn't compared, so I stuck with x265. |
| Splitting the render and joining with the concat demuxer | Broke sync: video started 0.8 s late and skipped at the join. **Render in one go.** If you must split, keep the parts as separate files. |

---

## 4. Encoding settings (keep the quality)

- **Video:** `libx265 -preset fast`, 10-bit (`yuv420p10le`), `-tag:v hvc1` so
  QuickTime/Apple devices play it.
  - Same size as the original: `-b:v <source video bitrate>` (DJI 4K ≈ 70 Mbps).
  - Or quality-based: `-crf 14` (what the 10 s HQ test used, ~58 Mbps).
- **Audio:** `-c:a copy` copies the original AAC track exactly, with no loss.
- **Metadata:** `-map_metadata 0` keeps the creation date.
- An exact byte-for-byte size match is impossible with any re-encode. Targeting the
  bitrate gets within a few %.

---

## 5. Gotchas on this Mac (i5-7267U, ffmpeg in `/usr/local/bin`)

- **The software HEVC decoder in this ffmpeg build segfaults** (exit 139). Always add
  `-hwaccel videotoolbox` before `-i` for DJI HEVC files, including when you're just
  extracting frames.
- `ffprobe` isn't installed. Use `ffmpeg -hide_banner -i file.mp4` to see stream info.
- **Speed:** about **0.6 fps**, i.e. ~57× slower than real time, for 4K with this filter.
  - 10 s test ≈ 10 min.
  - 9.5 min of video ≈ **8 hours**.
  - Run long jobs in the background so they survive closing the terminal:
    ```zsh
    nohup caffeinate -i ./deflicker.sh in.mp4 out.mp4 > render.log 2>&1 &
    tail -c 200 render.log      # check progress
    ```
    Keep the laptop plugged in with the **lid open**.
- A normal MP4 can't be played until it's finished. To be able to watch while it
  renders, add `-movflags +frag_keyframe+empty_moov` (plays in VLC; QuickTime may not).
- Stopping ffmpeg with Ctrl+C (SIGINT) still writes a valid, playable file of
  what's done so far.
- The DJI clip is portrait (stored rotated). ffmpeg auto-rotates, so the output is
  2160×3840 with the rotation applied to the pixels. That's normal.
- In zsh, `F="ffmpeg -hwaccel videotoolbox"; $F ...` does **not** split into words.
  Write the options out directly.
