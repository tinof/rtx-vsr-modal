# RTX Video Super Resolution on Modal

Upscale images and video with NVIDIA RTX Video Super Resolution (VSR) on
[Modal](https://modal.com) GPUs. You deploy it to your own Modal workspace, send it a file
over an authenticated HTTP API, and get the upscaled file back.

A 1080p video upscaled to 4K comes back as HEVC Main 10, with every audio track kept. The
whole video path runs on the GPU (NVDEC decode, VSR, NVENC encode), so a 1080p to 4K job runs
at about 36 fps on a short clip and about 75 fps on a long one, where four hardware encoders
work in parallel. Optional extras: a denoise or deblur pass before upscaling, SDR to HDR10
conversion with NVIDIA TrueHDR, and Matroska output that copies the source's audio, subtitles
and chapters untouched.

The project is also a working template for other Modal GPU services: a CPU web tier in front
of a GPU worker, a job queue that gets around Modal's 150-second HTTP limit, and file handoff
through a Modal Volume. See [Using this as a template](#using-this-as-a-template).

Working on the code with an AI agent? Point it at [CLAUDE.md](CLAUDE.md) (also linked as
`AGENTS.md`). It holds the conventions and the hardware traps that have already cost days.

## Contents

- [Requirements and cost](#requirements-and-cost)
- [Quick start](#quick-start)
- [Client options](#client-options)
- [Choosing a quality mode](#choosing-a-quality-mode)
- [HDR10 output (TrueHDR)](#hdr10-output-truehdr)
- [Output format](#output-format)
- [Speed](#speed)
- [HTTP API](#http-api)
- [Deploy settings](#deploy-settings)
- [Choosing a GPU](#choosing-a-gpu)
- [How it works](#how-it-works)
- [Three pins that are easy to break](#three-pins-that-are-easy-to-break)
- [Limits and guarantees](#limits-and-guarantees)
- [Troubleshooting](#troubleshooting)
- [Comparing results and measuring](#comparing-results-and-measuring)
- [Using this as a template](#using-this-as-a-template)
- [License and credits](#license-and-credits)

## Requirements and cost

- A Modal account with GPU access. The worker runs on an **RTX PRO 6000** (Blackwell) by
  default. It is the only Modal GPU with 9th-generation NVENC, and several larger Modal GPUs
  have no video encoder at all (see [Choosing a GPU](#choosing-a-gpu)).
- Python 3.13 or newer for the local client, plus the `modal` CLI. You do not need a local
  GPU or a local ffmpeg; everything runs in the Modal containers.
- A Modal proxy auth token pair. The endpoint requires one, because every request spends GPU
  money.

At the default settings a clip under 30 s, encoded by one NVENC session, takes roughly 42
minutes of GPU time per hour of 25 fps video (1080p to 4K), about $2.40 at Modal's 2026-10
RTX PRO 6000 rate. Inputs of 30 s or more use four encoders in parallel: that takes about
half the GPU time per frame on long inputs and two thirds on a 61-second clip. Modal bills
while a container is up, and a worker stays warm for 120 s after its last job. Check
Modal's current prices before a large batch.

At the default quality target (CQ 20), grainy 4K lands roughly between 35 and
80 Mbps. A 61-second 1080p Blu-ray sample came out at 78 Mbps, about 35 GB per hour.

## Quick start

1. Install the client dependencies and log in to Modal:

   ```bash
   git clone https://github.com/tinof/rtx-vsr-modal.git
   cd rtx-vsr-modal
   uv sync && source .venv/bin/activate    # or: pip install -r requirements.txt
   modal token new
   ```

2. Deploy the app. The first deploy also builds the GPU image:

   ```bash
   modal deploy modal_app.py
   ```

   Modal prints the endpoint URL, which looks like
   `https://<workspace>--rtx-media-upscaler-api.modal.run`.

3. Create a proxy auth token pair and export it for the client:

   ```bash
   modal workspace proxy-tokens create
   export MODAL_KEY=... MODAL_SECRET=...
   export ENDPOINT="https://<workspace>--rtx-media-upscaler-api.modal.run"
   ```

4. Upscale a file:

   ```bash
   # 2x video with the default settings (HIGHBITRATE_ULTRA, HEVC Main 10, mp4)
   python modal_client.py --endpoint "$ENDPOINT" --input ./input.mkv --scale 2.0

   # An image, fitted inside 1920x1080 (add --no-keep-aspect-ratio to stretch to it)
   python modal_client.py --endpoint "$ENDPOINT" --input ./input.png \
       --resize-type "target dimensions" --width 1920 --height 1080 --quality HIGH
   ```

The client uploads the file, waits for the worker, and saves `*_upscaled.png`,
`*_upscaled.mp4` or `*_upscaled.mkv` next to the input unless you pass `--output`. It prints
the job's `call_id` at submit time. If the client stops, resume with `--call-id <call_id>`
instead of uploading again. Network errors and proxy 502/503/504 answers are retried with
backoff. The download goes to `<output>.part` and is renamed only when its size matches
`Content-Length`.

The first job in a fresh container also pays about 10 to 14 s of NGX initialization.

## Client options

| Option | Default | What it does |
|---|---|---|
| `--endpoint` | (required) | The URL `modal deploy` printed. |
| `--input` | | Local image or video file. Video is recognised by extension (`.mp4`, `.mov`, `.mkv`, `.avi`, `.webm`, `.m4v`) or MIME type. |
| `--output` | `*_upscaled.<ext>` | Where to save the result. |
| `--resize-type` | `scale by multiplier` | Or `target dimensions`. |
| `--scale` | `2.0` | Multiplier for `scale by multiplier`. Must be at least 1.0. |
| `--width`, `--height` | `1920`, `1080` | Box for `target dimensions`. |
| `--no-keep-aspect-ratio` | aspect kept | Stretch to the exact box instead of fitting inside it. |
| `--quality` | `HIGHBITRATE_ULTRA` | Upscaling model. See [Choosing a quality mode](#choosing-a-quality-mode). |
| `--preprocess` | none | Optional `DENOISE_*` or `DEBLUR_*` pass at input size before upscaling. |
| `--strength`, `--preprocess-strength` | `1.0` | Effect strength, 0.0 to 1.0. Lower values blend the model's effect back towards a plain resample. |
| `--truehdr [settings]` | off | SDR to HDR10 output for video. See [HDR10 output](#hdr10-output-truehdr). |
| `--master-display`, `--max-cll` | see HDR10 | HDR10 static metadata. |
| `--container` | `mp4` | `mkv` copies the source's audio, subtitles and chapters as they are. |
| `--call-id` | | Resume waiting for a job that was already submitted. |
| `--timeout`, `--poll-interval` | 3 h, 5 s | How long and how often to poll. |

## Choosing a quality mode

The service accepts 17 of NVIDIA's VSR models, in four families, all probed working on the
RTX PRO 6000. They have two roles:

| Parameter | Accepted values | What it does |
|---|---|---|
| `quality` | `BICUBIC`, `LOW`, `MEDIUM`, `HIGH`, `ULTRA` | Upscales and suppresses compression artifacts, which also softens clean sources. |
| `quality` | `HIGHBITRATE_LOW` to `HIGHBITRATE_ULTRA` | Upscales without artifact suppression, for clean or high-bitrate sources. |
| `preprocess` | `DENOISE_LOW` to `DENOISE_ULTRA` | Same-resolution pass before upscaling. Removes macro-blocking and mosquito noise. |
| `preprocess` | `DEBLUR_LOW` to `DEBLUR_ULTRA` | Same-resolution pass before upscaling. Sharpens soft footage. |

A mode in the wrong role (a deblur mode as `quality`, say) is rejected with `400` before any
GPU work.

**The default is `HIGHBITRATE_ULTRA`, not `ULTRA`.** "High bitrate" means few compression
artifacts, not high resolution. Broadcast archive material is usually soft but cleanly
encoded, so the standard family has no real artifacts to remove. It removes texture instead
and leaves a blotchy, painterly look. On the reference clip, mode 19 (`HIGHBITRATE_ULTRA`)
beat mode 4 (`ULTRA`) on both axes: about 15% more edge detail and roughly 8x less
flat-region noise. Use `ULTRA` for heavily compressed, blocky encodes.

Check what the source really holds before judging the output. A "1080p" broadcast file is
often an SD master that someone already upscaled. Downscale a frame to 540p and back up. If
that reproduces the file almost exactly (mean absolute difference below 1/255), there is no
real detail for VSR to recover, and the output will look close to a Lanczos resize at any
setting.

Preprocess costs a second inference pass per frame, so use it only when the source is the
limiting factor. Measured on a soft, already-upscaled SD master, all with `HIGHBITRATE_ULTRA`:

| Preprocess | Edge detail | Flat-region noise | Temporal flicker |
|---|---|---|---|
| none | 3.7 | 0.10 | 1.89 (baseline) |
| `DEBLUR_LOW` | 6.5 | 0.11 | +22% |
| `DEBLUR_MEDIUM` | 6.6 | 0.12 | +18% |
| `DEBLUR_HIGH` | 5.9 | 0.12 | +12% |
| `DEBLUR_ULTRA` | 5.6 | 0.09 | +5% |

Two results contradict the mode names:

- Deblur intensity is not monotonic. `DEBLUR_LOW` and `DEBLUR_MEDIUM` give more apparent
  detail than `DEBLUR_HIGH` and `DEBLUR_ULTRA`. The upper rungs seem to regularise more
  alongside the sharpening.
- Judge deblur in motion. Flat-region noise that changes from frame to frame is what looks
  bad on screen. `DEBLUR_ULTRA` adds about 50% detail for 5% more flicker. `DEBLUR_LOW` looks
  best on a still and flickers the most.

Start without preprocess. If a soft source needs more bite, try `DEBLUR_ULTRA` on a short clip
and watch it in motion before a batch run:

```bash
python modal_client.py --endpoint "$ENDPOINT" --input ./soft_source.mkv \
    --scale 2.0 --quality HIGHBITRATE_ULTRA --preprocess DEBLUR_ULTRA
```

### Effect strength

`strength` scales the upscale model's effect and `preprocess_strength`
the preprocess pass, in every family. Detail on one 1080p web-encode still at 2x (mean
absolute Laplacian; a plain `BICUBIC` upscale scores 1.01):

| Mode | strength 1.0 | 0.5 | 0.0 |
|---|---|---|---|
| `ULTRA` | 2.98 | 1.78 | 1.24 |
| `HIGHBITRATE_ULTRA` | 2.40 | 1.68 | |
| `HIGHBITRATE_ULTRA` + `DENOISE_MEDIUM` preprocess | 1.59 | 1.86 | 2.40 (no preprocess) |
| `HIGHBITRATE_ULTRA` + `DEBLUR_ULTRA` preprocess | 3.48 | 2.52 | 2.40 (no preprocess) |

These come from a single still; noise and flicker in motion were not measured. At the default,
1.0, no strength is passed to the SDK, so output matches a job that never set it.

## HDR10 output (TrueHDR)

`--truehdr` turns an SDR video into HDR10 with NVIDIA's TrueHDR model. It runs on every frame
after super-resolution, at output size, in the same GPU loop. The output is HEVC Main 10
tagged BT.2020 / PQ (SMPTE ST 2084) / BT.2020 non-constant luminance, limited range, chroma
sited top-left. Every keyframe carries the HDR10 mastering-display and content-light-level
SEI, and the mp4 gets a `colr` box.

The parameters follow NVEncC and StaxRip, so a desktop command line carries over:

| StaxRip / NVEncC | This service | Default |
|---|---|---|
| `--vpp-ngx-truehdr contrast=…,saturation=…,middlegray=…,maxluminance=…` | `truehdr` (same `key=value` list, or `on`) | `contrast=102,saturation=102,middlegray=46,maxluminance=680` |
| (no switch) | `truehdr=…,debanding=off` skips TrueHDR's debanding pass | debanding on |
| `--master-display "G(…)B(…)R(…)WP(…)L(max,min)"` | `master_display` (same string) | `G(8500,39850)B(6550,2300)R(35400,14600)WP(15635,16450)L(<maxluminance>×10000,1)` |
| `--max-cll "MaxCLL,MaxFALL"` | `max_cll` (same string; 0 = unknown) | measured per job |
| `--colormatrix bt2020nc --colorprim bt2020 --transfer smpte2084 --chromaloc 2` | always set with `truehdr` | |
| `--vpp-resize algo=ngx-vsr,vsr-quality=4` | `quality=ULTRA` | `HIGHBITRATE_ULTRA` |

```bash
python modal_client.py --endpoint "$ENDPOINT" --input ./input.mkv --scale 2.0 \
    --truehdr "contrast=102,saturation=102,middlegray=46,maxluminance=680"
```

- Order: VSR first, then TrueHDR, as in NVEncC, whatever the command-line order. In a
  probe, running VSR on PQ data instead made bright edges overshoot from about 650 to about
  3200 nits.
- No crop or pad. The output keeps the source aspect. Black stays at 0 nits, so bars
  inside the source stay black.
- The default primaries are BT.2020, as in the reference StaxRip command
  (G 0.170/0.797, R 0.708/0.292). Discs usually signal P3-D65,
  `G(13250,34500)B(7500,3000)R(34000,16000)WP(15635,16450)L(…)`; pass that if you prefer it.
- MaxCLL and MaxFALL are measured per job. Without `max_cll`, the worker measures the
  brightest pixel and the brightest frame average of the encoded frames and writes those
  values (rounded up) into every keyframe. An explicit `max_cll="a,b"` is written as given.
  The 300 nits of the StaxRip command overstate MaxFALL on dark drama, where test clips
  measured 30-52 nits. Letterbox bars count in the frame average, so MaxFALL reads slightly
  low on letterboxed sources (about 1.33x low for 2.39:1 in a 16:9 frame). The worker log
  prints the values: `HDR10 light levels: measured … ; written … (measured)`.
- TrueHDR adds about 1 ms of GPU work per 4K frame and the BT.2020 conversion about
  1.2 ms. Short inputs run at SDR speed. On the four-encoder path the encode phase measured
  14-18% slower (55-63 against 67-72 fps). `debanding=off` cut TrueHDR from 0.31 to 0.14 ms
  per frame at 1080p.
- Rejected with `400`: image inputs, sources that are already HDR (PQ or HLG), and outputs
  above 8192x4320. If the GPU encoder path is unavailable the job fails before any GPU
  work instead of writing SDR.

## Output format

Video is HEVC Main 10 (`yuv420p10le`, Level 5.1, High tier), tagged bt709 for matrix,
primaries and transfer, or HDR10 with `truehdr`. 10-bit is used even for 8-bit sources: it
costs nothing on NVENC and keeps upscaled gradients from banding. Images come back as PNG.

### Container

mp4 is the default. Every source audio track is kept. A track is stream-copied
when mp4 can carry its codec (AAC, AC-3, E-AC-3, Opus, ALAC, MP3, FLAC); any other codec (DTS,
TrueHD, PCM, ...) is re-encoded to AAC 192k for that track only, with a warning in the worker
log. mp4 output drops subtitles.

`container=mkv` writes Matroska from the same encode. `mkvmerge` copies the source's audio,
subtitles, chapters, attachments and tags without re-encoding, and converts mp4 timed text to
SRT. The MKV track header repeats the colour signalling and, for HDR10, the mastering display
and MaxCLL/MaxFALL, because players that read colour from the container would otherwise see
an untagged stream. A source mkvmerge cannot read is rejected with `400` before any GPU work.

### 10-bit sources

10-bit SDR sources (for example `yuv420p10le` HEVC) are decoded as packed 10-bit RGB, and
VSR and any preprocess pass run in the SDK's 10-bit mode, so the extra precision reaches the
model. 8-bit sources are unaffected.

### Interlace-flagged sources

Many UK and EU HD discs carry progressive 25p pictures in a
1080i container (PsF). The in-process GPU decoder deinterlaces any interlace-flagged stream,
and on such a disc it removed 32% of vertical detail before VSR saw the frame. The worker
therefore decodes these inputs through ffmpeg's NVDEC path, which weaves the fields
losslessly, and logs a warning saying so. Truly interlaced 50i/60i video would need a real
deinterlacer, which is not included.

### Encoder settings

The quality point was chosen on 2026-10-03 by VMAF against a lossless encode of the same VSR
frames (a BBC Blu-ray, 1080p to 4K). As an NVEncC command line:

    --qvbr 20 --codec h265 --preset p5 --tune uhq --output-depth 10 --profile main10
    --tier high --level 5.1 --max-bitrate 100000 --multipass 2pass-full --aq
    --aq-strength 10 --aq-temporal --bframes 5 --lookahead 32

| Setting | Default | Why |
|---|---|---|
| Preset | `p5` | P6 and P7 add only 0.06 to 0.15 VMAF over P5 and cost about 30% more encode time. |
| Multipass | `fullres` (2pass-full) | About 1.1 VMAF better than `qres` at the same bitrate. |
| Tuning | `uhq` | 1.7 VMAF better than `hq` with 19% fewer bits. |
| CQ | `20` | A file-size choice. CQ 18 measured about 80 Mbps (VMAF 94.3), CQ 24 about 35 Mbps (VMAF 91.3). |
| Level / ceiling | `5.1` High, 100 Mbps | Above 4096x2176 the driver picks the level. |

The worker sets the bitrate ceiling itself. PyNvVideoCodec zeroes NVENC's `maxBitRate`
whenever a CQ target is set, and the driver then applies its own VBR ceiling: about 20 Mbps at
Level 5.0, 32 Mbps at 5.1, 48 Mbps at 6.0. Grainy 4K hit that ceiling, so CQ 18 and CQ 24 gave
the same file size. With the level unset the driver also picked it per session, so the same
job sometimes gave a 2.4x larger file. The worker sets Level 5.1 for outputs up to 4K and
restores a 100 Mbps ceiling with a 1 s buffer. After a job with a different output size, NVENC
sometimes rejects the explicit level; the worker then retries with the driver's level and logs
a warning. The ceiling, and so the file size, is the same either way.

All of these are deploy-time settings: see [Deploy settings](#deploy-settings).

## Speed

Measured on the RTX PRO 6000 Blackwell Server Edition, 1080p to 4K, `HIGHBITRATE_ULTRA`, warm
containers:

| Stage, measured alone | Speed |
|---|---|
| VSR `run()` | 3.5 ms/frame (about 286 fps) |
| VSR + integrity check + P010 conversion | 4.4 ms/frame |
| NVENC, P4 + qres + UHQ | 65.0 fps |
| **NVENC, P5 + 2pass-full + UHQ (default)** | **38.5 fps** |
| NVENC, P5 + 2pass-full + HQ | 68.1 fps |
| NVENC, P6 + 2pass-full + UHQ | 27.5 fps |
| NVENC, P7 + 2pass-full + UHQ | about 24 fps |

The encoder is the bottleneck, not the model. A single session runs a job at about **36 fps**
end to end.

The card has four NVENC engines. Inputs of 30 s or more are cut at source keyframes into
segments of up to 30 s, encoded by four sessions in parallel, and joined without re-encoding.
That runs at about **75 fps** end to end on a 10-minute 1080p clip (83 fps in the encode phase)
and 55 fps on the 61-second sample, where the fixed costs (creating four encoders, joining)
weigh more. Four UHQ sessions do not scale linearly: each slows to about 26 fps when all four
run. Mean VMAF against a lossless encode is the same either way (91.27 vs 91.25), and the
segmented path keeps memory bounded on feature-length inputs.

The worker logs one summary line per job. A segmented job:

    Video done: 1526 frames in 27.6s = 55.4 fps (setup 2.8s [effects 0.0s, encoders 2.7s],
    encode 20.7s = 73.9 fps, join 4.1s; sessions=4, segments=4; …)

A single-session job:

    Video done: 1526 frames in 42.8s = 35.7 fps (decode-wait 1%, infer 34%, encode 54%;
    nvdec=yes, batch=1, gpu-encoder=yes)

`gpu-encoder=yes` means NVENC encoded from GPU memory. `nvdec=in-process` is expected for
progressive input on the single-session path; `nvdec=yes` means the piped NVDEC decoder ran,
which is normal for interlace-flagged input and on the segmented path. The `infer` share
includes time spent waiting for the encoder.

The full measurement history, from the original 12 fps pipeline onward, is in
[docs/PERF-HISTORY.md](docs/PERF-HISTORY.md).

## HTTP API

The client is a thin wrapper around three endpoints. Every request needs the proxy auth
headers.

Submit a job. The response contains a `call_id`:

```bash
curl -X POST "$ENDPOINT/upscale" \
    -H "Modal-Key: $MODAL_KEY" -H "Modal-Secret: $MODAL_SECRET" \
    -F "file=@./input.png" \
    -F "resize_type=scale by multiplier" \
    -F "scale=2.0"
```

The form fields match the client options: `resize_type`, `scale`, `width`, `height`,
`keep_aspect_ratio`, `quality`, `preprocess`, `strength`, `preprocess_strength`, `truehdr`
(`on` or a `contrast=…,maxluminance=…` list), `master_display`, `max_cll` and `container`.

Poll for the result. While the job runs this returns HTTP 202; when it is done it returns the
file:

```bash
curl "$ENDPOINT/result/fc-XXX" \
    -H "Modal-Key: $MODAL_KEY" -H "Modal-Secret: $MODAL_SECRET" \
    -o upscaled.png
```

The output is deleted from the Volume once it has been downloaded, so a second request returns
410. A failed job's files are deleted when it fails. Errors return JSON with a `detail`
message: 400 for invalid parameters, 404 when the result has expired, 500 when upscaling
failed.

Health check (returns 401 without the headers):

```bash
curl "$ENDPOINT/health" -H "Modal-Key: $MODAL_KEY" -H "Modal-Secret: $MODAL_SECRET"
```

## Deploy settings

These environment variables are read when the image is built and baked into it. Setting one
in your shell later does nothing; it must be on the `modal deploy` command line. The worker
prints the resolved values at startup (`Selected video encoder: … preset=p5, cq=20, …`), which
is the quickest way to check.

| Variable | Default | Effect |
|---|---|---|
| `MODAL_NVENC_CQ` | `20` | Quality target, equivalent to NVEncC `--qvbr`. Lower means higher quality and larger files. |
| `MODAL_NVENC_PRESET` | `p5` | NVENC preset, `p1` to `p7`. |
| `MODAL_NVENC_MULTIPASS` | `fullres` | `qres` = 2pass-quarter, `fullres` = 2pass-full. |
| `MODAL_NVENC_TUNING` | `uhq` | `uhq` or `hq`. UHQ is slower and better per bit. |
| `MODAL_NVENC_LEVEL` | `5.1` | HEVC level for outputs up to 4096x2176. Empty lets the driver pick. |
| `MODAL_NVENC_MAX_MBPS` | `100` | VBR ceiling in Mbit/s with a 1 s buffer. `0` leaves the driver's ceiling, which caps CQ. |
| `MODAL_NVENC_AQ_STRENGTH` | `10` | Adaptive quantisation strength, 1 to 15. |
| `MODAL_NVENC_BFRAMES` | `5` | B-frames. |
| `MODAL_NVENC_REFS` | `5` | Reference frames, piped ffmpeg encoder only. |
| `MODAL_NVENC_SESSIONS` | `4` | Parallel encoder sessions on the segmented path. |
| `MODAL_SEGMENT_SECONDS`, `MODAL_SEGMENTED_MIN_SECONDS` | `30`, `30` | Segment length, and the input length from which the segmented path is used. |
| `MODAL_GPU_ENCODER` | `1` | Encode from GPU memory. `0` pipes frames to ffmpeg instead. |
| `MODAL_GPU_DECODER` | `1` | Decode in-process with NVDEC. `0` uses a decode subprocess. |
| `MODAL_GPU` | `RTX-PRO-6000` | Worker GPU type. |
| `MODAL_WORKER_CPU` | `6` | Worker CPU cores. Measured faster than 12. |
| `MODAL_WORKER_MAX_CONTAINERS` | `0` (no cap) | Cap on worker containers. |
| `MODAL_APP_NAME` | `rtx-media-upscaler` | Deploy a variant beside the main app instead of replacing it. |

To run a second configuration, give it its own app name. Both apps share the job Volume:

```bash
MODAL_APP_NAME=rtx-media-upscaler-cq24 MODAL_NVENC_CQ=24 modal deploy modal_app.py
modal app stop rtx-media-upscaler-cq24 -y    # when you no longer need it
```

`MODAL_WORKER_CONCURRENCY` exists but is not safe above 1 yet; CLAUDE.md explains why.

## Choosing a GPU

NVENC is a separate hardware engine from the CUDA cores, and several Modal GPUs have none:

| GPU | NVENC | Usable |
|---|---|---|
| RTX PRO 6000 (Blackwell) | Yes, 9th generation, 4 engines | Yes, the default |
| L40S | Yes | Yes, but slower per dollar |
| H100, H200, A100, B200, B300 | **None** | No: video falls back to CPU `libx264` |

A GPU without NVENC still produces correct output, but encodes on the CPU at roughly a tenth
of the speed, so a pricier card buys less throughput. Compare GPUs by cost per frame, not by
fps or price per second. On the older pipeline the L40S gave 56% of the throughput for 72% of
the cost. Before you benchmark another GPU, read
[docs/PERF-HISTORY.md](docs/PERF-HISTORY.md) and the benchmark rules in
[CLAUDE.md](CLAUDE.md).

The GPU choice also fixes the torch version. Blackwell is `sm_120` and needs
`torch==2.13.0`; torch 2.6 fails there with "no kernel image is available".

## How it works

```
client ──upload──▶ api (CPU, FastAPI) ──spawn──▶ UpscaleWorker (GPU)
   ▲                  │  Volume API                 │ NVDEC → VSR → NVENC
   └──download────────┴──── rtx-upscaler-jobs ◀─────┘  (frames stay in GPU memory)
```

- **Web tier (`api`).** A CPU-only FastAPI app, so GPU time is billed only while frames are
  being upscaled. It writes the upload to the `rtx-upscaler-jobs` Modal Volume through the
  Volume API, without mounting it, spawns the GPU job, and streams the finished file back.
  One container handles up to 20 concurrent requests.
- **Job queue.** Modal web endpoints time out after 150 seconds, and video jobs run far
  longer. `POST /upscale` therefore returns a `call_id` at once, and `GET /result/{call_id}`
  returns 202 until the file is ready.
- **GPU worker (`UpscaleWorker`).** Its `@modal.enter()` warms CUDA and probes the encoder once
  per container. The VSR model stays loaded between jobs that share quality and output size.
- **GPU-resident video path.** NVDEC decodes into CUDA memory, VSR runs on the decoded tensor,
  and NVENC encodes straight from GPU memory. Removing the host round-trips between stages was
  the largest single speed-up in the project's history (12 to 59 fps).
- **Segmented encode.** Long inputs are split at keyframes and encoded on all four NVENC
  engines at once (see [Speed](#speed)).

Everything is in two files: `modal_app.py` (the web tier and the GPU worker) and
`modal_client.py` (the local client). `scripts/` holds standalone GPU probes and quality
tools; [CLAUDE.md](CLAUDE.md) lists what each one is for.

## Three pins that are easy to break

All three live in `modal_app.py`. Read this before you change the image.

**1. The NGX driver library (`DRIVER_VERSION`).** RTX VSR is an NGX feature. The `nvidia-vfx`
wheel loads the driver-side `libnvidia-ngx.so.1` at runtime, and Modal's container runtime
does not provide that library. Without it, `NvVFX_Load` fails on every GPU type:

```
NvVFX_Load failed: The effect has not been properly initialized (code -12)
```

So the image installs that one library from the matching NVIDIA driver package.
`DRIVER_VERSION` should track the host driver from `nvidia-smi`, and the worker logs a warning
at startup when they differ. The image pins `580.95.05`. On 2026-11-25 Modal's containers ran
on `610.57.04` and `620.70.06`, none on 580; VSR and TrueHDR still load with the 580 library,
so the warning fires at every start. The pin stays because containers land on hosts with
different drivers and no single value matches all of them.

**2. The ffmpeg build (`FFMPEG_URL`).** Each ffmpeg build links an NVIDIA Video Codec SDK with
a minimum driver version:

| ffmpeg build | Video Codec SDK | Requires driver | Usable here |
|---|---|---|---|
| BtbN `master-latest` | 13.1.15 | 610.0+ | No: NVENC silently unavailable |
| BtbN `n8.1` (pinned) | 570-compatible | 570.0+ | Yes, including `av1_nvenc` |
| Debian apt ffmpeg 5.1 | | | Yes, but no `av1_nvenc` |

With `master-latest`, every NVENC encoder fails to initialise and the worker falls back to CPU
`libx264`: correct output, roughly a tenth of the speed. The pin is a month-end BtbN release
with a SHA-256 check, because BtbN rebuilds its `latest` tags daily. The table was measured on
580 hosts; newer hosts meet `master-latest`'s requirement, but the fleet is mixed and that
build has not been re-tested.

**3. mkvmerge (`MKVTOOLNIX_VERSION`).** Matroska output uses `mkvmerge` 104 from the
MKVToolNix author's apt repository, pinned by version, with the repository key checked by
hash. Debian bookworm's own v74 does not see mp4 timed-text subtitles and drops them without
an error.

## Limits and guarantees

- Upscaling only. `scale` must be at least 1.0, and a target smaller than the input is
  rejected with `400`.
- Output dimensions are rounded to multiples of 8 (minimum 8). In `target dimensions`
  mode the output fits inside the box unless you pass `--no-keep-aspect-ratio`.
- Maximum output edge: 16384 pixels. Larger requests are rejected with `400` before any
  GPU work. The SDK corrupts output at 16K (the blue channel collapses to 0), so between 15360
  and 16384 the worker runs VSR at 15360 and resamples the rest of the way with GPU bicubic
  interpolation.
- Every frame is checked on the GPU for non-finite values and channel collapse, so corrupt
  SDK output fails the job instead of reaching your file.
- Fallbacks are never silent. If the GPU decoder or encoder cannot be used, the worker
  falls back and logs a warning that names the cause.
- One job per worker container. Modal scales out with more containers instead.
- Resize types, the quality mapping, the multiple-of-8 alignment and the
  `MAX_PIXELS = 1024 * 1024 * 16` batching heuristic come from the ComfyUI node this project
  started as. They are part of the API contract.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `NvVFX_Load failed … (code -12)` | The NGX library does not match the host driver. The worker's startup line `WARNING: host NVIDIA driver is …` names the host driver; set `DRIVER_VERSION` to it and redeploy. It is not a GPU-model problem. |
| The log says `Selected video encoder: libx264` | NVENC is unavailable: a GPU without NVENC, or an ffmpeg build that needs a newer driver. Output is correct but about ten times slower. |
| `no kernel image is available for execution on the device` | torch is too old for the GPU. Blackwell needs `torch==2.13.0`. |
| `401` from every endpoint | Missing or wrong `Modal-Key` / `Modal-Secret` headers. |
| `410` from `/result` | The file was already downloaded; outputs are deleted after the first download. |
| A deploy setting has no effect | It was set in the shell after the deploy. Put it on the `modal deploy` command and check the worker's startup line. |
| The output looks no sharper than a Lanczos resize | The source may be an upscaled SD master. See the 540p check in [Choosing a quality mode](#choosing-a-quality-mode). |
| `NVENC CreateEncoder returned error 8 (intermittent) … retrying` | Expected now and then; the worker retries. A `WARNING: NVENC rejected explicit HEVC level` means the retries ran out and the driver's level was used. |

## Comparing results and measuring

`scripts/compare_frames.py` builds side-by-side images from an input and an output, matching
frames by index:

```bash
uv run --with pillow --with numpy python scripts/compare_frames.py \
    --original ./input.mkv --upscaled ./output.mp4
```

It needs a local `ffmpeg` and `ffprobe`, and its `FONT_PATH` points at a macOS font; change
it on other systems. For each frame it writes a full-frame comparison, a centre crop (where differences show at
screen zoom), and an amplified difference map. The original side is Lanczos-resized to the
output size and labelled as such, so you compare against the best conventional resampler.

If you benchmark settings or GPUs:

1. Compare warm containers only. The same configuration measured 8.0 fps cold and 12 fps
   warm on a 12-second clip. Run the clip three or four times in one container and read the
   later runs.
2. Deploy the variant beside the main app with its own `MODAL_APP_NAME`, and set
   `MODAL_WORKER_MAX_CONTAINERS=1` so the autoscaler does not spread the runs over
   containers.
3. Use real content. On synthetic noise the `cq` response measured inverted.
4. Check pixels, not only metadata. For a full day this pipeline produced visibly striped
   4K that passed frame count, `ffprobe`, a clean decode and the built-in integrity check. Two
   cheap checks catch it: count frames whose standard deviation is above 1 (fewer than the
   frame count means flat grey frames), and take the mean absolute Laplacian of a 512-pixel
   crop (about 1 to 3 on the reference clip; about 190 means column striping).
   `scripts/probe_p010_encode.py` runs both on a synthetic clip.
5. Compute cost per frame, from Modal's current per-second price for the GPU plus the CPU
   and memory you requested.

The full verification protocol for code changes is in [CLAUDE.md](CLAUDE.md).

## Using this as a template

Only the upscaling logic is specific to this project. Keep these for any Modal GPU service:

- The CPU web tier and GPU worker split, so GPU time is billed only for GPU work.
- The job-queue API, which avoids the 150-second limit on web endpoints.
- The `modal.Volume` file handoff: the web tier uses the Volume API (`batch_upload`,
  `read_file`, `remove_file`) and only the GPU worker mounts the Volume.
- `requires_proxy_auth=True` and the client's submit-and-poll loop.
- `@modal.enter()` for anything expensive to initialise.

Replace `UpscaleWorker._upscale_image` / `_upscale_video` and the `nvvfx` calls. Drop the
`DRIVER_VERSION` / `libnvidia-ngx` install unless the new job also uses an NGX feature. Keep
the ffmpeg pin only if the job encodes video, and re-verify NVENC against the host driver of
the day. Rename the app (`rtx-media-upscaler`) and the Volume (`rtx-upscaler-jobs`), or the new
service will collide with this one in the same workspace.

## License and credits

The code in this repository is licensed under the [Apache License 2.0](LICENSE).

It started as a port of the Comfy-Org ComfyUI RTX node and also draws on
`whmc76/ComfyUI-NVIDIA-RTX-VSR-Pro`; both are Apache-2.0 and credited in [NOTICE](NOTICE).

The image build downloads NVIDIA software that this repository does not contain or
redistribute: the `nvidia-vfx` and `PyNvVideoCodec` wheels from PyPI, and one library from
NVIDIA's Linux driver package. Those components are under NVIDIA's own license terms, which
you accept when you build the image. The same applies to the BtbN ffmpeg build and to
MKVToolNix under their licenses.
