# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is `DataCrusade1999/DataCrusade1999` — a GitHub special profile README repo. There is no
application code, no build system, and no tests. `README.md` embeds a set of `metrics.plugin.*.svg`
and `profile-3d-contrib/*.svg` images; every one of those SVGs is generated and committed by
scheduled GitHub Actions, not authored by hand. The only things a human (or Claude) actually edits
here are `README.md` and the workflow files under `.github/workflows/`.

## Workflows (the real "build" of this repo)

- `.github/workflows/metrics.yml` — runs nightly (`30 0 * * *`), on manual dispatch, and on push to
  `main` that touches `metrics.yml` itself (scoped via `paths:` so README/docs-only pushes don't
  trigger a run). It regenerates all `metrics.plugin.*.svg` files (via `lowlighter/metrics@latest`
  and `dkhokhlov/metrics@master`), the dev.to devcard (`dailydotdev/action-devcard`), and the 3D
  contribution graphs under `profile-3d-contrib/` (via `yoshi389111/github-profile-3d-contrib`),
  then commits and pushes the results back to `main`.
- `.github/workflows/codesnippet.yml` — runs yearly, regenerates `metrics.plugin.code.svg`.
- `concurrency: { group: metrics, cancel-in-progress: false }` on `metrics.yml` queues overlapping
  runs instead of letting them race — two concurrent runs committing generated SVGs to `main` used
  to collide (stale-sha rejections, "local changes would be overwritten by merge"). Don't remove
  this without solving that problem another way.
- Jobs in `metrics.yml` are deliberately split into 4 sequential lanes via `needs:` chains, each
  `if: ${{ !cancelled() }}` so one failing plugin doesn't stop the rest of its lane. This keeps
  concurrent GitHub API callers low enough to avoid secondary rate limits (which used to bake
  "Unexpected error" cards into the generated SVGs). Lane 4 specifically isolates the
  slow/unpredictable jobs (`dkhokhlov/metrics@master` cold Docker builds can take 10+ min) so they
  never sit in front of and block a fast lane. When adding a new plugin job, put it in the lane
  that matches its expected runtime and add it to that lane's `needs:` chain — don't create a fifth
  lane or a parallel top-level job unless it's genuinely independent and cheap.
- Most `lowlighter/metrics` steps need `token: ${{ secrets.METRICS_TOKEN }}` (a PAT with broader
  scopes than the default `GITHUB_TOKEN`) for plugins that read private/organization data;
  plugins that only hit public third-party APIs (dev.to, Stack Overflow, XKCD, Twitter) use
  `token: NOT_NEEDED` instead.
- `metrics.yml` starts with a `detect` job that diffs the file's previous vs. current committed
  version, per job block (via `yq`), and every plugin job is gated on `needs.detect.outputs.run_all
  == 'true' || contains(needs.detect.outputs.changed, ' <job-key> ')`. On a push that only edits one
  job's `with:` inputs, only that job runs. It always runs everything for `schedule`/
  `workflow_dispatch`, and falls back to running everything on push whenever it can't safely tell
  what changed (new branch, force-push, a change outside `jobs:` like `concurrency:`/triggers, or
  any error in the diffing script itself) — so a broken diff never silently skips real work. When
  adding a new job, no changes to `detect` are needed; just give the new job its own `contains(...,
  ' new-job-key ')` check like the others.

## Making changes

- To change what a generated SVG shows, edit the corresponding job's `with:` inputs in
  `metrics.yml`/`codesnippet.yml` — see https://github.com/lowlighter/metrics#-documentation for
  the full plugin option reference. Never hand-edit a `metrics.plugin.*.svg` or
  `profile-3d-contrib/*.svg` file directly; it will be overwritten on the next scheduled run.
- To verify a workflow change, trigger it manually (`gh workflow run metrics.yml` or via the
  Actions tab) rather than waiting for the nightly schedule, and check the run for the specific job
  you touched.
- `README.md` layout changes (adding/removing/reordering an embedded image, editing bio text, links)
  are plain hand edits — no generation step involved.
- `patches/` holds one-off patch files that were applied historically; it isn't part of any
  automated process.
