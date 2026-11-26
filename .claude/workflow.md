# Workflow contract

What `/ship:land`, `/ship:roadmap-next` and `/ship:roadmap-init` need to know about this repository. The schema,
with every key and section, is `~/.claude/skills/ship/reference/contract.md`. Keep every `##` heading
exactly as written: the skills look sections up by name. A missing section, or one whose body is `none`,
turns that part off. A diff that touches this file is always Tier 2.

## Settings

```
contract:       1.2
repo:           tinof/rtx-vsr-modal
base:           main
merge:          squash
branch:         <type>/<id>-<slug>
roadmap:        ROADMAP.md
roadmap-log:    docs/ROADMAP_LOG.md
roadmap-memory: off
push-roadmap:   no
docs-prefix:    "docs(roadmap): "
gate:           uvx ruff check --select F,E9 . && uv lock --check
gate-fallback:  none
gate-skip:      'README.md' 'CLAUDE.md' 'ROADMAP.md' 'docs/**' 'LICENSE' 'NOTICE'
ci:             pr-checks
logic:          '*.py'
tests:          'tests/**'
self-check:     none
review-surfaces: exec
```

- With `merge: squash` the PR body becomes the commit body: follow the PR body format in `~/.claude/CLAUDE.md` (no headings or checklists, no hard wrap).

## Gates

none

## Tier 2 paths

```
.github/**
.claude/**
.agents/**
AGENTS.md
CLAUDE.md
pyproject.toml
uv.lock
```

CI workflows and Dependabot decide what merges unchecked. The agent files steer every reviewer and builder. `pyproject.toml` and `uv.lock` pin the local client's dependencies; the container dependencies are pinned in `modal_app.py` and need a GPU round-trip before they change.

## Routing

default

## Testing rules

- There is no test suite. `gate` checks syntax and undefined names only, and CLAUDE.md says static checks are not evidence that the service works. A change to the GPU path, the web tier or the image is verified only by CLAUDE.md's "Verification protocol" on a bench deploy (`MODAL_APP_NAME=rtx-bench-<x>`), never over production. A reviewer who cannot run it marks a runtime claim NEEDS-HUMAN.
- `.venv/bin/python -c "import modal_app"` works on this host (modal is a client dependency). It validates decorator arguments only, but pure helpers (`parse_hdr_settings`, `_normalize_quality`, `hdr10_sei_nals`, `insert_sei_before_irap_slice`, `patch_cll_sei`, `_hevc_level`) can be called this way without a GPU.
- The probes in `scripts/` are standalone by design (own image, no `import modal_app`). Do not make one import the app.
- Compare GPU output bytes only between runs on the same host driver, ideally in one warm container.

## Size limits

- `python`: files '*.py'; file 800; function none; item E1.
- `claude-md`: files 'CLAUDE.md'; file 24000 bytes; item E1.
- `roadmap`: files 'ROADMAP.md'; file 48000 bytes; item E1.

## Probes

- `modal-auth: modal app list >/dev/null 2>&1 && echo "modal token works"` The Modal CLI is installed and its token is accepted. It spends nothing; a bench deploy and a GPU job do.

## Tags

- `modal`: needs probe `modal-auth`

## Readiness rules

- A step that deploys a bench app or runs a GPU job spends Modal credit and needs the owner present, with tags `owner` and `modal` (probe `modal-auth`).

## Material

none

## Branching

- Types: `feat/`, `fix/`, `docs/`, `chore/`, `refactor/`, `test/`.
- Sharing rule: steps never share a branch.

## Release

none

## Unattended

none

## Codex notes

- The reviewer's sandbox cannot run Modal or a GPU. Claims about NVENC, NVDEC, nvvfx or NGX behaviour come from probes on the real hardware (CLAUDE.md), not from SDK documentation.
- Known false alarms: the `.clone()` after `torch.from_dlpack()`, `torch.cuda.synchronize()` before `Encode()`, holding each P010 surface until the next device-wide sync, the hybrid resize path above 15360 px and `_check_frame_integrity` are all required (CLAUDE.md hard constraints 4 and the video-path traps). Flagging them as dead code or as a slowdown is wrong.
- Pins that look stale on purpose: `FFMPEG_URL` (a month-end BtbN build, not `latest`), `DRIVER_VERSION` and `torch==2.13.0` (CLAUDE.md hard constraints 1-3).

## Reminders

- Test on a bench deploy (`MODAL_APP_NAME=rtx-bench-<x> modal deploy modal_app.py`, then `modal app stop rtx-bench-<x> -y`), never over production `rtx-media-upscaler`.
- A merged `modal_app.py` change whose `[modal]` evidence step is still open is unverified. Deploy production only from a commit whose evidence step passed.

## Notes

The gate is the CI job (`.github/workflows/ci.yml`): ruff F,E9 and the lock check, run over the whole repo (`.`) so modules that a `modal_app.py` split creates stay linted. Ruff's rule set is passed on the command line because the host's ruff picks up rules from outside the repo when none are given. The GPU code cannot run in CI or on this host, so no `## Gates` entry runs the verification protocol: it costs Modal credit and stays an `owner` step. The repo has no deploy command that meets `## Unattended`'s `deploy` contract, so no step is `live` and batch never deploys. A `modal_app.py` split is not a pure move: new modules need `add_local_python_source` in both images, so there are no `move-steps`.

Hard by this repo's criterion: any change to the video path (`_upscale_video*`, the segmented workers, decoder and encoder setup) or to the web tier (`api`, the job and Volume handoff) in `modal_app.py`. The ordering traps in CLAUDE.md make a plausible-looking change produce corrupt output that passes every static check, and only a paid GPU run shows it. Such steps need a written plan and are never routed to batch.

The owner declined `self-review` on 2026-11-26: every PR, batch ones included, gets `/ship:land`'s cross-family review. `roadmap-memory` is `off` (owner, 2026-11-26): the roadmap is small, so every run ranks afresh.
