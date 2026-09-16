# Gribble GitHub Action

Tiny bugs that find your bugs before you ship. This action runs `gribble audit --ci` in your repository and turns the report into a **Gribble** check run with annotations, pull request comments that update in place, a SARIF/JUnit pair, and a downloadable report artifact.

```yaml
- uses: gribble-dev/action@v1
```

This repository is a publish mirror. The source lives in the [gribble monorepo](https://github.com/gribble-dev/gribble) under [`packages/action`](https://github.com/gribble-dev/gribble/tree/main/packages/action); issues and pull requests go there. Full documentation: [gribble.dev/docs/ci-github-action](https://gribble.dev/docs/ci-github-action).

## Requirements

- **`gribble` installed as a devDependency** of the audited repository. The action runs the version in your lockfile. If it cannot find one it falls back to npx, logs a warning, and you get whatever the latest release is.
- **`@earendil-works/pi-ai` and `@earendil-works/pi-coding-agent`** as devDependencies too, but only for `mode: review` or `mode: all`. Gribble declares the AI review runtime as optional peer dependencies, which no package manager installs on its own; `mode: gate` needs neither. The npx fallback fetches them itself, and the [Docker image](#docker-image) ships them.
- **A `.gribble/` directory** created with `gribble init` and committed, including `.gribble/baseline/` once you have run the first audit (see [Baseline](#baseline-write-back)).
- **A Chromium build for Playwright**: run `gribble install` in the job, or use the [Docker image](#docker-image) that ships one.
- **Node.js 22 or newer** on the runner for the CLI. The action itself declares the `node24` runtime.

## Usage

Two jobs. Pull requests get audited and commented on; pushes to the default branch refresh the baseline.

```yaml
# .github/workflows/gribble.yml
name: Gribble

on:
  pull_request:
  push:
    branches: [main]

jobs:
  audit:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      checks: write
      security-events: write # only for the SARIF upload below
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0 # baseline comparison needs history

      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v7
        with:
          node-version: 22
          cache: pnpm

      - run: pnpm install --frozen-lockfile
      - run: pnpm exec gribble install # Playwright Chromium

      - uses: gribble-dev/action@v1
        id: gribble
        with:
          mode: all
          fail-on: error
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: ${{ steps.gribble.outputs.sarif-path }}

  baseline:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v7
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec gribble install

      - uses: gribble-dev/action@v1
        with:
          mode: gate
          update-baseline: true
          comment: false
```

`mode: gate` runs only the deterministic checks and needs no model credential, which is why the `baseline` job passes none. `review` and `all` read the model provider key from the environment (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, ... as configured in `gribble.yaml`); pass it as `env`, never as an input. Use an API key, not a subscription login.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `mode` | `all` | `gate` (deterministic checks and flow replay), `review` (AI review) or `all`. |
| `working-directory` | `.` | Directory containing the audited app's `.gribble/`, relative to the repository root. |
| `target` | | Monorepo target directory relative to `working-directory` (`gribble audit --target`). |
| `all` | `false` | Audit every `.gribble/` under `working-directory` (`gribble audit --all`). |
| `environment` | | Named environment from `gribble.yaml` (`gribble audit --env`). |
| `update-baseline` | `false` | On pushes to the default branch, run with `--update-baseline` and write `.gribble/baseline/` back. |
| `comment` | `true` | Post and maintain pull request comments. |
| `max-comments` | `5` | Maximum number of new per-finding review comments per run; the rest are listed in the summary comment. |
| `fail-on` | `error` | Fail the step when a **new** finding has at least this severity: `critical`, `error`, `warn` or `none`. |
| `github-token` | `${{ github.token }}` | Token for the check run, pull request comments and baseline write-back. |
| `report-artifact` | `true` | Upload the run directory and the SARIF/JUnit files as the `gribble-report` artifact. |

`fail-on: none` is the right setting while you roll Gribble out: everything is still reported, commented on and uploaded, but nobody's build breaks. Tighten to `error` once the baseline is honest.

## Outputs

| Output | Description |
| --- | --- |
| `gate` | `pass` or `fail` as reported by gribble, independent of `fail-on`. |
| `new-findings` | Number of new findings relative to the baseline (summed over targets). |
| `report-path` | Absolute path of the report JSON (`.gribble/runs/latest.json`). |
| `sarif-path` | Absolute path of the generated `gribble.sarif`; feed it to `github/codeql-action/upload-sarif` to see findings in the Security tab. |

## Permissions

Grant the least that covers what you enabled.

| Permission | Needed for |
| --- | --- |
| `contents: read` | Always; checking out the repository. |
| `contents: write` | `update-baseline: true` with `baseline.update: commit`. |
| `pull-requests: write` | PR comments, and `baseline.update: pr`. |
| `checks: write` | The check run and its annotations. |
| `security-events: write` | Uploading the SARIF file to code scanning. |

Without `pull-requests: write` the action logs a warning and continues; without `checks: write` annotations degrade to workflow log annotations. `GITHUB_TOKEN` is enough; no GitHub App is required.

**Forked pull requests** get a read-only token, so comments and the check run will not post. The audit still runs and the artifact still uploads. If you need comments on fork PRs, use `pull_request_target` and understand the security trade-off.

**Workflow approval.** Pull requests from first-time contributors, and workflows on branches pushed by bots, can sit in "workflow awaiting approval" until a maintainer approves the run. Nothing posts until then.

## What one run does

1. Runs `gribble audit --ci --mode <mode> [--target <dir>] [--all] [--env <name>] [--update-baseline]` in `working-directory`. Progress streams to the job log; the report is read from `.gribble/runs/latest.json` (one per target with `all: true`).
2. Writes `gribble.sarif` and `gribble-junit.xml` next to the report and sets the outputs.
3. Uploads the run directory (report, screenshots, aria snapshots, traces) and the SARIF/JUnit files as the `gribble-report` artifact.
4. Creates the **Gribble** check run with one annotation per new finding that names a source file. Gribble records symbol names rather than line numbers, so the line is found by searching for the symbol (line 1 when it is not found).
5. On pull requests, maintains the comments (below).
6. On pushes to the default branch with `update-baseline: true`, writes `.gribble/baseline/` back.
7. Fails the step according to `fail-on`.

## Pull request comments

- One summary comment marked `<!-- gribble:summary -->`, updated in place on every run.
- One review comment per new finding that maps to a file in the diff, marked `<!-- gribble:fp:<fingerprint> -->`. Re-runs update the comment and keep the thread; a fingerprint that disappears gets `patched ✅` prepended and its thread resolved.
- New findings that do not map into the diff, or exceed `max-comments`, are listed in the summary comment.
- Findings already recorded in `.gribble/baseline/findings.json` are counted, never commented. `info` findings stay in the report.
- `max-comments` applies to new findings only, ordered by severity, then confidence, deterministic rules first.
- Bootstrap runs (no baseline committed yet) post no comments.

## Baseline write-back

A pull request never writes the baseline, otherwise it could bless its own regressions. With `update-baseline: true` on a push to the default branch the audit runs with `--update-baseline`, then the action follows `baseline.update` in `gribble.yaml`:

- `commit` (default): commits `.gribble/baseline/` as `github-actions[bot]` with `chore(gribble): update baseline [skip ci]` and pushes to the branch. Needs `contents: write`.
- `pr`: pushes a `gribble/baseline-<sha7>` branch and opens a chore pull request. For protected branches; needs `pull-requests: write`.
- `manual`: leaves the working tree alone; run `gribble baseline update` locally and commit.

Pushes made with `GITHUB_TOKEN` do not trigger other workflows, and the baseline commit carries `[skip ci]` anyway.

## Monorepos

Set `all: true` (or `target: apps/web`) and the action audits every app with its own `.gribble/` in one job. The summary comment gets one section per app, fingerprints include the app name, and `max-comments` applies across all apps.

## Docker image

If installing a browser in CI is inconvenient, run the job in the prebuilt image. It is based on the official Playwright image and ships Node 22, Gribble and Chromium, so the `gribble install` step goes away.

```yaml
jobs:
  audit:
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/gribble-dev/gribble:0
    permissions:
      contents: read
      pull-requests: write
      checks: write
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - run: pnpm install --frozen-lockfile
      - uses: gribble-dev/action@v1
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Tags: `latest`, the exact version (`0.2.1`) and the moving major (`0` while the CLI is 0.x, `1` from 1.0.0). Pin the major rather than `latest` so a release cannot change your CI behaviour without a commit. The action still runs the `gribble` from your lockfile; the image only provides Node and the browser.

## Versions and runtime

- `gribble-dev/action@v1` follows every release. `gribble-dev/action@v0.2.1` pins one. Version tags equal the `gribble` CLI version they were built from; `v1` is the moving tag for every 0.x and 1.x release.
- The action declares `runs.using: node24`. GitHub runners run JavaScript actions on Node 24 by default since June 16, 2026 and remove Node 20 on September 23, 2026, so declaring `node20` would not have changed anything.
- The bundle inlines `@actions/*` and `yaml`. `@gribble/core` is resolved at runtime from the audited repository's `node_modules` (it depends on Playwright and Lighthouse); when it is missing, built-in formatters produce the summary, SARIF and JUnit output.

## Source and releases

The action is developed in [`gribble-dev/gribble`](https://github.com/gribble-dev/gribble) as the private workspace package `@gribble/action`. `dist/` is git-ignored there, so a release workflow ([`mirror-action.yml`](https://github.com/gribble-dev/gribble/blob/main/.github/workflows/mirror-action.yml)) builds it with tsdown on every `gribble@X.Y.Z` tag and commits `action.yml`, `dist/index.js`, `README.md`, `CHANGELOG.md` and `LICENSE` to this repository as `vX.Y.Z`, moving `v1` along. Do not open pull requests against the mirror; they would be overwritten by the next release.

To build it yourself:

```bash
git clone https://github.com/gribble-dev/gribble
cd gribble && pnpm install && pnpm build
ls packages/action/dist/index.js
```

## License

[MIT](./LICENSE)
