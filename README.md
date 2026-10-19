# Xalgorix Scan — GitHub Action

[![License](https://img.shields.io/badge/License-Apache_2.0-C51C39)](LICENSE)

Run an autonomous [Xalgorix](https://www.xalgorix.com) pentest from CI against a
deployed target (a staging/preview URL or production host) and optionally fail
the build when findings at or above a severity threshold are discovered.

It's a thin wrapper over the public REST API (`POST /api/public/v1/scans` +
status polling), so it works on any runner with `bash`, `curl`, and `jq`.

> **Free to run in CI.** The Action itself is open source and free — on public
> repos GitHub's runner minutes are free too, so wiring it in costs nothing.
> Each scan it starts uses your Xalgorix credits (single = 1 credit, wildcard =
> 1 per live subdomain), exactly like the web UI and API. No extra fee for using
> the Action.

> Xalgorix is a DAST engine — it scans a **live, reachable URL**, not the PR
> diff. Point `target` at a deployed environment (e.g. your preview deployment).

## Usage

```yaml
- name: Xalgorix security scan
  uses: xalgorix/scan-action@v1
  with:
    api-key: ${{ secrets.XALGORIX_API_KEY }}
    target: https://staging.example.com
    mode: single
    fail-on: high
```

Create an API key under **Dashboard → API keys**. Store it as the
`XALGORIX_API_KEY` repository/organization secret.

## Use it on pull requests

Drop this in `.github/workflows/security.yml` to scan on every PR and block the
merge on proven high/critical findings (a ready-to-copy version lives in
[`examples/security.yml`](examples/security.yml)):

```yaml
name: security
on:
  pull_request:
    branches: [main]
concurrency:
  group: xalgorix-${{ github.ref }}
  cancel-in-progress: true
jobs:
  xalgorix:
    runs-on: ubuntu-latest
    # Fork PRs can't read secrets — only run on branches in this repo.
    if: ${{ github.event.pull_request.head.repo.full_name == github.repository }}
    steps:
      - name: Xalgorix security scan
        uses: xalgorix/scan-action@v1
        with:
          api-key: ${{ secrets.XALGORIX_API_KEY }}
          target: https://staging.example.com
          fail-on: high
```

**Scanning the PR's own preview deploy.** Xalgorix scans a running app, so the
most useful PR setup is to deploy the PR first and pass that preview URL:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.preview_url }}
    steps:
      - id: deploy
        run: echo "preview_url=https://pr-${{ github.event.number }}.staging.example.com" >> "$GITHUB_OUTPUT"
        # ...your actual deploy step sets preview_url...
  xalgorix:
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - uses: xalgorix/scan-action@v1
        with:
          api-key: ${{ secrets.XALGORIX_API_KEY }}
          target: ${{ needs.deploy.outputs.url }}
          fail-on: high
```

### Things to know for PR runs

- **The target must be reachable from the public internet** — the hosted
  Xalgorix scanner connects to it, not the GitHub runner. `localhost` or a
  private-network staging won't work unless it's exposed.
- **Fork PRs don't get secrets.** GitHub withholds repository secrets from PRs
  opened from forks, so the scan won't run there (the `if:` guard above makes
  that explicit instead of a confusing empty-key failure).
- **Only scan targets you own or are authorized to test.** Unauthorized
  scanning breaches the Terms and the law.
- **Pin the Action** to a commit SHA (`@<sha>`) once you've reviewed it, rather
  than the moving `@v1` tag, for supply-chain safety.
- **Scans use credits and take time.** Keep PR runs focused with
  `mode: single` and a sensible `timeout-minutes`; use `fail-on: none` if you
  want the report without blocking the merge.

> No deployed URL to scan on a PR? Use the self-hosted engine's **code-scan**
> mode instead — `xalgorix --source ./ --code-scan review` in CI reviews the
> repository source with no live target. See the
> [engine repo](https://github.com/xalgorix/xalgorix).

## Inputs

| Input             | Default                     | Description |
|-------------------|-----------------------------|-------------|
| `api-key`         | —  (required)               | Xalgorix API key (`xlg_live_…`). |
| `target`          | —  (required)               | Live URL/host to scan. |
| `mode`            | `single`                    | `single` or `wildcard`. |
| `severity-filter` | `low`                       | Minimum severity to report. |
| `fail-on`         | `high`                      | Fail the job on a finding at/above this severity (`none` to never fail). |
| `wait`            | `true`                      | Wait for completion, or return right after queuing. |
| `poll-interval`   | `20`                        | Seconds between status polls. |
| `timeout-minutes` | `60`                        | Max wait before failing. |
| `base-url`        | `https://www.xalgorix.com`  | Override for self-hosted/staging. |

## Outputs

`scan-id`, `status`, `web-url`, and per-severity counts: `critical`, `high`,
`medium`, `low`, `info`.

```yaml
- id: scan
  uses: xalgorix/scan-action@v1
  with:
    api-key: ${{ secrets.XALGORIX_API_KEY }}
    target: https://staging.example.com
    fail-on: none          # don't fail; just report
- run: echo "Found ${{ steps.scan.outputs.critical }} critical, ${{ steps.scan.outputs.high }} high"
```

## Notes

- Each scan consumes credits exactly like the web UI (single = 1 credit;
  wildcard = 1 per live subdomain).
- A per-severity table is written to the job summary, with a link to the full
  report in your dashboard.
- Set `wait: false` for fire-and-forget (e.g. nightly scans whose results you
  review in the dashboard or via a `scan.completed` webhook).
