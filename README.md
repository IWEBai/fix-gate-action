# Fix Gate

**Was that security fix actually a fix?**

Fix Gate is a free GitHub Action that verifies a security-fix pull request, whoever or whatever wrote it: GitHub Copilot Autofix, Snyk Agent Fix, an editor assistant, or a person. It answers four questions and posts them as a check run named **Fix Gate**:

| Check | How |
|---|---|
| Targeted finding gone | Railo's rules run over the base and head versions of every changed file. The finding reported at base, inside the changed lines, must no longer match at head. |
| Nothing new in the changed lines | A finding at head with no counterpart at base, inside the changed lines, fails. A pre-existing finding that only shifted lines does not. |
| Changed files still parse | Python, JavaScript, TypeScript, Java and PHP are parse-checked. A file that already failed to parse at base is not held against the change. |
| Tests did not newly fail | Taken from your own test job's result (`tests_result`). Without it, this row says *not run*. |

Each row is **pass**, **fail**, **nothing to check** or **not run**. A check that could not run is shown as not run. It is never counted as a pass.

It is rule-based: the verdict comes from running the same rules twice and comparing, plus a parser. No language model reads your code. The scans run inside your own GitHub Actions job; no file content leaves it.

**Status: beta (v0.x).** The rules are the ones behind Railo's published [failure-rate study](https://www.railo.dev/research/autofix-11-repositories), and they will miss things. The check run always shows the rule and line behind a fail, so a reviewer can judge it.

## Why

In August 2026, 1Password tested 6,080 security patches written by two frontier models: 26% were fully correct and 54% left the vulnerability in place, introduced a new one, or both ([their write-up](https://1password.com/blog/why-ai-generated-patches-still-require-human-review)). GitHub counts an Autofix as successful only when the alert is gone, no new alert appeared, the file still parses and the tests still pass. Fix Gate applies that same bar to every fix pull request and reports each part separately.

## Install

Commit this as `.github/workflows/fix-gate.yml`:

```yaml
name: Fix Gate

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write
  checks: write

jobs:
  fix-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          fetch-depth: 0
          persist-credentials: false
      - uses: IWEBai/fix-gate-action@v0
```

`fetch-depth: 0` matters: the gate compares against the commit the pull request branched from.

To fill the tests row, run your tests in a job named `test` first, then:

```yaml
  fix-gate:
    needs: [test]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          fetch-depth: 0
          persist-credentials: false
      - uses: IWEBai/fix-gate-action@v0
        with:
          tests_result: ${{ needs.test.result }}
```

## Inputs

| Input | Default | What it does |
|---|---|---|
| `rule` | | Rule id of the finding the pull request fixes. With `file`, names the targeted finding instead of inferring it. |
| `file` | | Path of the file the finding is in. |
| `line` | | Line of the finding at base, to pick between several in one file. |
| `alert` | | Number of a code-scanning alert the pull request fixes. Targets its file and line, and its rule when the alert came from Railo; another tool's rule is matched by file and line and Railo's rules decide. Needs `security-events: read`. |
| `sarif_base` | | Path to a SARIF file from another tool's scan of the base commit (CodeQL, Snyk, Semgrep). Each finding in the changed files must be matched by a Railo finding in the same file, by CWE or line, that is gone at head. A finding Railo's rules do not match is reported as not run, never as gone. |
| `comment` | `false` | `true` also posts the verdict as one pull request comment, edited in place on later runs. Needs `pull-requests: write`. |
| `require_resolved` | `false` | `true` fails the gate when no base finding was resolved at all, so a change that fixes nothing cannot pass as a fix. |
| `tests_result` | | Your test job's outcome: `success`, `failure` or `skipped`. |
| `timeout` | `120` | Seconds for the two scans. A timeout is reported as not run. |
| `fail_on_failure` | `true` | `false` reports a failed gate without failing the job. |
| `base_branch` | | Branch the pull request targets; defaults to the pull request's base. |
| `github_token` | `github.token` | Token used to create the check run. |

Outputs: `conclusion` (`success`, `failure`, `neutral`) and one of `pass`, `fail`, `neutral`, `not_run` for each of `finding`, `new`, `parse`, `tests`.

## What it reports as *not run*

- Findings from other tools (CodeQL, Snyk, a pentest) that Railo's rules do not also report. With `sarif_base`, a finding is matched to a Railo finding in the same file by CWE or line and confirmed gone only through that match; an unmatched one is not run. Without `sarif_base`, other tools' findings are not considered at all.
- Go, Ruby and C# files in the parse check.
- The tests row, unless `tests_result` is passed.
- A scan that timed out, or more than 200 changed files.

## Pull requests from forks

The token on a fork pull request is read-only, so the check run cannot be created. The verdict is written to the job summary instead, and the job's exit code still reflects it.

## Permissions

`contents: read` to read the repository, `checks: write` to create the check run, `pull-requests: write` only if you ask for a comment. `security-events: read` only if you pass `alert`. Fix Gate never pushes and never writes under `.github/workflows`.

## Telemetry

One usage ping per run with the repository name, the mode (`fix-gate`), the Action version, and counts. Never file content. Set `RAILO_TELEMETRY: "0"` in the step's `env` to turn it off.

## Source

This repository holds the Action definition. The engine runs from the `ghcr.io/iwebai/railo` image published by [Railo](https://www.railo.dev); the gate logic is `core/fix_gate.py` there. The same checks are how Railo verifies the fixes it writes itself in a [Fix Sprint](https://www.railo.dev/pentest-remediation).

## Licence

The files in this repository are MIT licensed. The engine image is Railo's and is free to use through this Action.
