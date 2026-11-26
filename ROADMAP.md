# rtx-vsr-modal Roadmap

This is the single source of truth for open `rtx-vsr-modal` work. Closed steps, measurements and
long evidence are in [`docs/ROADMAP_LOG.md`](docs/ROADMAP_LOG.md); measured performance history is
in [`docs/PERF-HISTORY.md`](docs/PERF-HISTORY.md).

> **North star:** NVIDIA RTX Video Super Resolution as an authenticated Modal job queue that turns
> 1080p sources into correct 4K (SDR or HDR10) at the encoder's speed limit; success is the
> CLAUDE.md verification protocol passing on the hosts Modal gives us.

Status meanings: **Open** has not shipped; **Partial** has useful pieces in place but does not meet
the gate; **Core complete** meets the gate but keeps named follow-up work; **Deferred** is
intentionally not scheduled. Items with no step left move to the log.

**Step format.** Every open step is one line that `/ship:roadmap-next` reads:
`- [<ID>][tags] <step>. Gate: <evidence>.` with an optional `Needs: <ID>[, <ID>]` for a prerequisite
(`Needs: <ID> (code)` also accepts a step that has landed with its gate pending). IDs are
`<item>.<n>` (for example `E1.2`), assigned once and never reused or renumbered. A new step takes 1 +
the highest number ever used for that item in `ROADMAP.md`, `docs/ROADMAP_LOG.md` and
`Roadmap-Step:` commit trailers. A step without a gate, or with `Gate: TODO(owner)`, waits for the
owner. Steps are shaped for `/ship:roadmap-batch` ("Batch-ready steps" in the format reference): each
names the files it changes, its gate runs unattended, and evidence only the owner can give is its
own `owner` step. GPU evidence costs Modal credit, so it is always an `[owner][modal]` step.

| Tag | Meaning |
|---|---|
| `desk` | Code or doc work; needs nothing |
| `owner` | Needs the owner: a decision, material or a test only they can run |
| `batch` | Routed to an unattended `/ship:roadmap-batch` run; set by `/ship:roadmap-init` or `/ship:roadmap-assess` with the owner's yes |
| `reassess` | A stop point: its outcome decides what follows, so the next run re-plans |
| `modal` | Deploys a bench app or runs a GPU job on Modal (probe `modal-auth`; spends credit) |

**How to update.** Format: `~/.claude/skills/ship/reference/roadmap-format.md`. This file holds open
work only; closed steps, Done notes and measurements go to the log.
- When a step ships, `/ship:land` removes its line, rewrites the item's Status to its current state (at
  most three lines), and adds the step's full text with the gate evidence to the log under
  `## <date>: <ID> <title> (PR #<n>)`.
- A step whose gate needs evidence that does not exist yet stays, with `Landed: PR #<n>; gate
  pending.` appended.
- An item with no steps left moves to the log.

**Which command.** All of them read the same ready queue (`roadmap-ready.sh`).

| Command | Does | Use it when |
|---|---|---|
| `/ship:roadmap-status` | Shows goals, lanes, what shipped and what waits for the owner; builds nothing | Any time, to see where the project stands and which command is next |
| `/ship:roadmap-next` | Builds one startable step, the first of the Focus and the queue, and ships it | The default. Always for a hard step or one that needs a decision on the way |
| `/ship:roadmap-batch` | Builds the startable `[batch]` steps unattended | The status page lists steps under "Batch now" |
| `/ship:roadmap-steer` | The weekly session with the owner: outcomes, decisions taken, open questions, Focus, batch routing | Once a week, or when the direction changes |

### Waiting for the owner

Steps that need a decision, material or a test only the owner can provide. Other steps of the
same item stay startable. `/ship:roadmap-next` reads this list; remove an entry when it is settled.

- **F2.1: driver pin policy for a mixed fleet.** Options: keep `DRIVER_VERSION` 580.95.05 and warn
  only for drivers not yet verified; move the pin to a newer driver after the verification protocol
  on both 610 and 620 hosts.
- **E2.1: pytest as a dev dependency.** A CPU test suite needs it; adding a dependency is the
  owner's call.
- **F1.1, F1.2, F3.1: paid GPU measurement runs.** Each needs a bench deploy and Modal credit.
- **F6.1: acceptance gate not stated.** Whether MaxFALL should leave out letterbox bars.
- **F7.1: acceptance gate not stated.** Whether to support truly interlaced 50i/60i sources.

## Priorities

Order (2026-11-26, owner): hygiene first. A clean log and an unattended test gate come first, then
the refactors those tests protect, then measurements and product work, then deferred refinements.

1. **Clean logs on a mixed driver fleet** (F2). Outcome: a worker log on any current Modal host
   shows no warning unless something is really wrong, so step 8 of the verification protocol can
   pass. Gate: F2.3.
2. **A CPU test gate** (E2). Outcome: the pure helpers that shape every HDR file and every request
   are checked on each PR without a GPU. Gate: E2.3.
3. **Code size** (E1). Outcome: no Python file over 800 lines and no function over 50 statements
   without a tracked step, and CLAUDE.md fits its budget. Gate: `size-check.sh --all` clean and the
   E1 evidence steps passed.
4. **Measurements the docs say are missing** (F1). Outcome: the strength advice and the HDR cost on
   current drivers rest on measured figures. Gate: F1.1 and F1.2 in docs/PERF-HISTORY.md.
5. **STREAMING quality modes** (F3). Outcome: modes 21 and 23 are either usable with documented
   behaviour or rejected for a recorded reason. Gate: F3.2, and F3.4 when they are accepted.
6. **Faster segmented HDR** (F4). Outcome: long HDR jobs run at close to SDR speed. Gate: F4.2.
7. **Two jobs per container** (F5). Outcome: concurrency above 1 is safe to turn on. Gate: F5.2.
8. **Deferred refinements** (F6, F7). Outcome: each has a decision. Gate: F6.1 and F7.1 decided.

## Features and measurements

### F1: Measurements the docs say are missing

- **Status:** Open; strength figures are still-frame only, and the HDR cost on 620.70.06 hosts is
  not known.
- Strength: README.md "Effect strength" and CLAUDE.md section 7 say noise and flicker at reduced
  strength were not measured. HDR: CLAUDE.md section 6 "Cost" says no SDR run of the same inputs was
  made on a 620 host.
- [F1.1][owner][modal] Measure detail, flat-region noise and temporal flicker (static-patch
  frame difference, `scripts/preprocess_sweep.py`) at strength 1.0, 0.5 and 0.0 for
  `HIGHBITRATE_ULTRA` and for the `DEBLUR_ULTRA` preprocess on the reference clip; record the
  figures in docs/PERF-HISTORY.md and the advice in README.md "Effect strength". Gate: the figures
  per strength in docs/PERF-HISTORY.md.
- [F1.2][owner][modal] Run the same short (<30 s) and segmented (>=30 s) clips as SDR and as HDR in
  one warm bench container, log the host driver, and record the HDR cost per path in
  docs/PERF-HISTORY.md; replace the "not known" sentence in CLAUDE.md section 6. Gate: SDR and HDR
  fps for both paths from one container in docs/PERF-HISTORY.md.

### F2: Host driver pin for a mixed fleet

- **Status:** Open; the image pins `DRIVER_VERSION` 580.95.05 while the 2026-11-25 hosts ran
  610.57.04 and 620.70.06, so every log carries the mismatch WARNING.
- `modal_app.py:138` (`DRIVER_VERSION`), `modal_app.py:1761` (`_warn_on_driver_mismatch`). VSR and
  TrueHDR loaded on both drivers (CLAUDE.md hard constraint 1).
- [F2.1][owner] Decide the policy: keep the 580 pin and warn only for drivers not yet verified, or
  move the pin after the verification protocol on both drivers. Gate: the decision recorded in this
  item.
- [F2.2][desk] Implement the F2.1 choice in `modal_app.py` (`DRIVER_VERSION`,
  `_warn_on_driver_mismatch`) and update README.md "Three pins that are easy to break". Gate:
  `gate`. Needs: F2.1.
- [F2.3][owner][modal] Verification protocol steps 3-8 on a bench deploy, on a 610 and on a 620
  host: NGX loads, and the log warns only where the F2.1 policy says. Gate: the protocol passes on
  both drivers. Needs: F2.2.

### F3: STREAMING_MEDIUM and STREAMING_ULTRA quality modes

- **Status:** Open; nvidia-vfx 0.2.0.0 adds modes 21 and 23, which the service rejects until they
  are probed and decided (CLAUDE.md section 5).
- `UPSCALE_QUALITIES` at `modal_app.py:251`, validated by `_normalize_quality` at
  `modal_app.py:335`.
- [F3.1][owner][modal] Probe modes 21 and 23 on RTX-PRO-6000: whether they change resolution, and
  detail, noise and flicker against `HIGHBITRATE_ULTRA` on the reference clip
  (`scripts/compare_frames.py`, `scripts/preprocess_sweep.py`); record in docs/PERF-HISTORY.md.
  Gate: the probe results in docs/PERF-HISTORY.md.
- [F3.2][owner] Decide whether modes 21 and 23 are accepted, and in which role (`quality` or
  `preprocess`). Gate: the decision recorded in this item. Needs: F3.1.
- [F3.3][desk][batch] Add the accepted modes to `UPSCALE_QUALITIES` (or `SAME_RES_QUALITIES`) in
  `modal_app.py` and to README.md "Choosing a quality mode". Gate: `gate`, and
  `.venv/bin/python -c "import modal_app"` with `_normalize_quality` accepting the new names in the
  decided role and rejecting them in the other. Needs: F3.2.
- [F3.4][desk] Update CLAUDE.md section 5 to the F3.2 decision. Gate: CLAUDE.md names the modes and
  their role. Needs: F3.3.

### F4: Segmented HDR encode speed (the `sz` stream scheme)

- **Status:** Open; the segmented HDR encode phase runs 14-18% slower than SDR because every
  thread's device-wide sync waits on all threads' extra work (CLAUDE.md section 6 "Cost").
- The `sz` scheme in `scripts/probe_parallel_encode.py` orders the encoder's input copy with events
  instead of a host sync and stayed byte-identical there (CLAUDE.md "Traps in this path"). In
  production the sync is still mandatory.
- [F4.1][desk] Replace the device-wide `torch.cuda.synchronize()` in `_upscale_video_segmented`
  (`modal_app.py`) with the `sz` event ordering, treating nvvfx `run()` as legacy-stream work.
  Gate: `gate`.
- [F4.2][owner][modal] Four warm runs byte-identical, exact P010 checksums equal across threads,
  verification protocol steps 4-10 (seams included), and segmented HDR encode fps against the
  pre-change deploy in the same window. Gate: those results in docs/PERF-HISTORY.md. Needs: F4.1.

### F5: Input concurrency per container

- **Status:** Open; `MODAL_WORKER_CONCURRENCY` above 1 is unsafe because the job input and final
  output still live on the Volume mount, where a `reload()` destroys in-flight jobs (CLAUDE.md
  "Input concurrency").
- [F5.1][desk] Move each job's input and final output into its local `scratch_dir` in
  `UpscaleWorker.run()` and the `_upscale_*` methods (`modal_app.py`), and copy only the finished
  output to the Volume before `commit()`. Gate: `gate`.
- [F5.2][owner][modal] Bench deploy with `MODAL_WORKER_CONCURRENCY=2` and
  `MODAL_WORKER_MAX_CONTAINERS=1`: two overlapping jobs both finish with outputs byte-identical to
  serial runs in the same container, and a default (`concurrency=1`) job is byte-identical to the
  pre-change deploy. Gate: those results in docs/PERF-HISTORY.md. Needs: F5.1.

### F6: Letterbox-aware MaxFALL

- **Status:** Deferred; letterbox bars count in the frame average, so MaxFALL reads about 1.33x low
  on 2.39:1 sources (CLAUDE.md section 6).
- [F6.1][owner] Decide whether MaxFALL should leave out letterbox bars. Gate: TODO(owner).

### F7: Truly interlaced sources

- **Status:** Deferred; interlace-flagged PsF input is woven correctly, but truly interlaced
  50i/60i content would need a deinterlacer (bwdif) ahead of VSR (CLAUDE.md "Traps in this path").
- [F7.1][owner] Decide whether to support truly interlaced 50i/60i sources. Gate: TODO(owner).

## Engineering backlog

### E1: Code size

- **Status:** Open; `modal_app.py` (3,326 lines) and `scripts/probe_parallel_encode.py` (914) are
  over 800 lines, 13 functions are over 50 statements, and CLAUDE.md is 51,486 bytes (budget 24,000).
- Ruff `PLR0915` is not selected in CI or `gate`, and Ruff's CLI `--select` overrides any
  `pyproject.toml` selection, so the command itself must change. A `modal_app.py` split is not a pure
  move: new modules need `add_local_python_source` in both images, and `scripts/probe_truehdr.py`
  reads `modal_app.py`'s sources through `inspect`. AGENTS.md is a symlink to CLAUDE.md.
- [E1.1][desk][batch] Turn on Ruff `PLR0915`: `[tool.ruff.lint.pylint] max-statements = 50` in
  `pyproject.toml`, `uvx ruff check --select F,E9,PLR0915 .` in `.github/workflows/ci.yml`, and mark the current
  violations with `# noqa: PLR0915 -- <step ID>` on their `def` lines: `modal_app.py`
  `_upscale_video_segmented` and its nested `worker` (E1.7), `_upscale_video_gpu` (E1.8),
  `_upscale_video` (E1.10), `api` (E1.13), `parse_hdr_settings` (E1.15); `modal_client.py` `run`
  (E1.12); `scripts/probe_parallel_encode.py` `_run_case` (E1.5), `scripts/probe_truehdr.py` `case`
  (E1.6), `scripts/probe_stages.py` `case` (E1.9), `scripts/probe_p010_encode.py` `encode_once`
  (E1.11), `scripts/probe_encoder_quality.py` `encode` (E1.14), `scripts/probe_nvdec.py` `probe`
  (E1.15). Gate: `uvx ruff check --select F,E9,PLR0915 .`
  green, and a 51-statement probe function in `modal_app.py` makes it fail.
- [E1.2][desk] Follow the lint in the agent files: `gate` in `.claude/workflow.md` to
  `uvx ruff check --select F,E9,PLR0915 . && uv lock --check`, its `python` size bullet to `function 50 by ruff PLR0915`, and CLAUDE.md
  "Verification protocol" step 1 to `uvx ruff check --select F,E9,PLR0915 .`. Gate: the new `gate` green and
  `size-check.sh --all` reads the section. Needs: E1.1.
- [E1.3][desk] Split `modal_app.py` (3,326 lines, limit 800) into modules by concern, with
  `add_local_python_source` for them in `web_image` and the GPU image, and keep
  `scripts/probe_truehdr.py`'s `inspect` source list working. Gate: `gate`,
  `.venv/bin/python -c "import modal_app"`, and `git diff --color-moved=dimmed-zebra` showing only
  moved blocks plus the import and image lines. Needs: E1.1.
- [E1.4][desk] Split `scripts/probe_parallel_encode.py` (914 lines, limit 800) into two probes that
  each stay standalone (own image, no shared local module). Gate: `gate`, and each new file's
  `--help` or case list covers every case of the old one. Needs: E1.1.
- [E1.5][desk] Split `_run_case` in `scripts/probe_parallel_encode.py` (537 statements) by case.
  Gate: `gate` with its marker removed. Needs: E1.4.
- [E1.6][desk] Split `case` in `scripts/probe_truehdr.py` (371 statements) by case. Gate: `gate`
  with its marker removed. Needs: E1.1.
- [E1.7][desk] Split `_upscale_video_segmented` (180 statements) and its nested `worker` (102) by
  phase, in the module E1.3 gives it. Gate: `gate` with both markers removed. Needs: E1.3.
- [E1.8][desk] Split `_upscale_video_gpu` (169 statements) by phase, in the module E1.3 gives it.
  Gate: `gate` with its marker removed. Needs: E1.3.
- [E1.9][desk] Split `case` in `scripts/probe_stages.py` (156 statements) by case. Gate:
  `gate` with its marker removed. Needs: E1.1.
- [E1.10][desk] Split `_upscale_video` (132 statements) by phase, in the module E1.3 gives it.
  Gate: `gate` with its marker removed. Needs: E1.3.
- [E1.11][desk] Split `encode_once` in `scripts/probe_p010_encode.py` (99 statements) by
  phase. Gate: `gate` with its marker removed. Needs: E1.1.
- [E1.12][desk][batch] Split `run` in `modal_client.py` (76 statements) by phase (argument
  handling, submit, poll, download). Gate: `gate` with its marker removed, and
  `.venv/bin/python modal_client.py --help` prints the same text as on `origin/main`. Needs: E1.1.
- [E1.13][desk] Split `api` (65 statements) in the module E1.3 gives it, one function per route.
  Gate: `gate` with its marker removed. Needs: E1.3.
- [E1.14][desk] Split `encode` in `scripts/probe_encoder_quality.py` (62 statements) by
  phase. Gate: `gate` with its marker removed. Needs: E1.1.
- [E1.15][desk] Split the two shortest: `probe` in `scripts/probe_nvdec.py` (60 statements)
  and `parse_hdr_settings` (57) in the module E1.3 gives it. Gate: `gate` with both markers
  removed, and a `.venv/bin/python` check that `parse_hdr_settings` returns equal settings and
  raises equal errors on `origin/main` and on the branch for the default, the README.md examples and
  an invalid string. Needs: E1.3.
- [E1.16][desk] Trim CLAUDE.md (51486 bytes, budget 24000): move detail to its topic doc or the
  log. Gate: `size-check.sh --all` shows CLAUDE.md within 24,000 bytes, and every hard constraint
  and the verification protocol are still in CLAUDE.md or behind a link from it.
- [E1.17][owner][modal] Verification protocol steps 3-11 on a bench deploy of the E1.3 split,
  against a bench deploy of the commit before it on the same host driver: byte-identical outputs.
  Gate: the protocol passes and the sizes match. Needs: E1.3.
- [E1.18][owner][modal] Verification protocol steps 3-11 on a bench deploy after the function
  splits, through `modal_client.py` for the image and video round-trips, against a bench deploy of
  the E1.3 commit on the same host driver: byte-identical outputs. Gate: the protocol passes and the
  sizes match. Needs: E1.7, E1.8, E1.10, E1.12, E1.13, E1.15.
- [E1.19][owner][modal] Run each split probe once on Modal and compare its printed results with a
  run of the pre-split probe in the same window. Gate: equal results per probe, or each difference
  explained by run-to-run noise the probe already reports. Needs: E1.4, E1.5, E1.6, E1.9, E1.11,
  E1.14, E1.15.

### E2: CPU tests for the pure helpers

- **Status:** Open; there is no test suite, and `gate` checks syntax and undefined names only.
- Pure helpers that run without a GPU: `hdr10_sei_nals` (`modal_app.py:530`),
  `insert_sei_before_irap_slice` (`:542`), `patch_cll_sei` (`:584`), `parse_hdr_settings`
  (`:407`), `_normalize_quality` (`:335`), `_hevc_level` (`:1019`).
- [E2.1][owner] Decide whether to add pytest as a dev dependency (`[dependency-groups]` in
  `pyproject.toml`). Gate: the decision recorded in this item.
- [E2.2][desk][batch] Add `tests/test_helpers.py`: golden MDCV and CLL SEI bytes for the default
  settings, captured from the current `hdr10_sei_nals` output (CLAUDE.md section 6 says they equal
  libx265's), emulation prevention for min luminance 1,
  `patch_cll_sei` keeping the escaped length (including a MaxCLL that is a multiple of 256),
  `insert_sei_before_irap_slice` placement, `parse_hdr_settings` defaults and rejections,
  `_normalize_quality` role checks, and `_hevc_level` up to and above 4096x2176. Add pytest to
  `pyproject.toml` and `uv.lock`, and a pytest step to `.github/workflows/ci.yml`. Gate:
  `uv run pytest tests/` green, and each test fails when its helper's output changes by one byte.
  Needs: E2.1.
- [E2.3][desk] Add `&& uv run pytest tests/` to `gate` and a `gate-focused` of
  `uv run pytest tests/ -k <pattern>` in `.claude/workflow.md`, and set its testing rules to the
  new suite. Gate: the new `gate` green. Needs: E2.2.

## Explicit non-goals

- Fixing in-process NVDEC's RGB truncation (about -0.43 luma levels) here: it needs a rebuilt
  PyNvVideoCodec wheel.
- `mdcv`/`clli` boxes in the mp4: the copy remux cannot attach the side data, and the in-band SEI
  carries the same metadata.
- NVEncC for VSR on Linux and GPUs other than `RTX-PRO-6000`: settled (CLAUDE.md "Settled").
- NVIDIA's Maxine VFX SDK: the fallback if the `libnvidia-ngx.so.1` install breaks, not scheduled
  work.
