# Roadmap log

Closed steps and items of [`ROADMAP.md`](../ROADMAP.md), newest first. Each closed step goes here
as `## <date>: <ID> <title> (PR #<n>)`, with its full text and the gate evidence. IDs listed here are
never reused.

## 2026-11-27: E1.12 Split `run` in `modal_client.py` (PR #12)

- [E1.12][desk][batch] Split `run` in `modal_client.py` (76 statements) by phase (argument
  handling, submit, poll, download). Gate: `gate` with its marker removed, and
  `.venv/bin/python modal_client.py --help` prints the same text as on `origin/main`. Needs: E1.1.

- **Evidence:** on PR #12's head, `uvx ruff check --select F,E9,PLR0915 . && uv lock --check`
  passed with the marker removed, and CI passed (`check`, GitGuardian). `modal_client.py --help`
  printed the same 58 lines from the main and branch files, both run under the same file name.
  An uncommitted harness with `requests`, `time.sleep` and `time.monotonic` stubbed ran 11 cases
  (submit, resume, upload and job errors, poll retries, deadline, truncated download, argument
  errors) against both files: output, HTTP calls, exits and written files were equal in all 11.
- **Review:** CodeRabbit CLI 0.9.7 (deep requested), LGTM with no findings, Tier 1. Built by an
  Opus builder in `/ship:roadmap-batch`, routed mode.
- **Resolution:** `run()` calls `_build_parser`, `_parse_args`, `_submit`, `_poll` and `_download`
  in the original order; the largest has 20 statements by ruff's count. The real round-trip through
  the client is E1.18.

## 2026-11-27: E1.1 Turn on Ruff PLR0915 (PR #10)

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

- **Evidence:** on PR #10's head, `uvx ruff check --select F,E9,PLR0915 . && uv lock --check`
  passed, and CI passed (`check`, GitGuardian). A temporary 51-statement function in
  `modal_app.py` failed with `modal_app.py:3329:5: PLR0915 Too many statements (51 > 50)`.
  `ast.dump` was identical before and after on all 8 touched `.py` files, so the markers are
  comments only.
- **Review:** CodeRabbit CLI 0.9.7 (deep requested), LGTM with no findings. The PR is Tier 2
  (`pyproject.toml`, `.github/**`); Claude built it, so one family was left and it ran as Tier 1.
  Built by an Opus builder in `/ship:roadmap-batch`, routed mode.
- **Resolution:** CI runs PLR0915 over the whole repository with `max-statements = 50`, and each
  of the 13 current violations carries `# noqa: PLR0915 -- <step ID>` on its `def` line.
- **Note for the splits:** in one shape (an assignment, 49 `t += 1`, a `return`), ruff 0.17.6
  first reported "51 > 50" at 52 physical lines. A split aimed at exactly 50 statements should
  leave a margin.
- **Docs gaps:** `CLAUDE.md:644` (Verification protocol step 1) and the contract `gate` still name
  `--select F,E9`; E1.2 updates both.
