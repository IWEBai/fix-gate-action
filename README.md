# Fix Gate

**Was that security fix actually a fix?**

Fix Gate is a free GitHub Action that verifies a security-fix pull request, whoever or whatever wrote it: GitHub Copilot Autofix, Snyk Agent Fix, an editor assistant, or a person. It answers five questions and posts them as a check run named **Fix Gate**:

| Check | How |
|---|---|
| Targeted finding gone | Railo's rules run over the base and head versions of every changed file. The finding reported at base, inside the changed lines, must no longer match at head. |
| Nothing new in the changed lines | A finding at head with no counterpart at base fails, whether it starts inside the changed lines or on a line the change made a rule match. A pre-existing finding that only shifted lines does not. A finding in a test, fixture or mock path (`tests/`, `__tests__/`, `spec/`, `test_*.py`, `*_test.go`, `*.test.ts`, `*Test.java` and the like) is listed under "In test files" and not judged, unless `file`, `alert` or `sarif_base` names that file. |
| Changed files still parse | Python, JavaScript, TypeScript, Java and PHP are parse-checked. A file that already failed to parse at base is not held against the change. |
| Tests did not newly fail | Taken from your own test job's result (`tests_result`). Without it, this row says *not run*. |
| Guard holds against known bypasses | The validator function the change adds (a predicate such as `is_safe_path`, or a sanitizer such as `clean_redirect`) is extracted and called on a fixed list of known bypass payloads for its weakness class: path traversal, open redirect, SSRF, argument injection. A fail quotes the payloads it accepted. A change that adds no standalone validator says *nothing to check*. |

Each row is **pass**, **fail**, **nothing to check** or **not run**. A check that could not run is shown as not run. It is never counted as a pass. When the run names a target (`rule`, `file`, `line`, `alert`, `sarif_base`) or sets `require_resolved`, a finding or new row that could not run fails the gate instead of leaving it neutral (`fail_on_not_run`, below).

It is rule-based: the verdict comes from running the same rules twice and comparing, plus a parser. No language model reads your code. The scans run inside your own GitHub Actions job; no file content leaves it. The ignore file (`.railoignore`) is read from the commit the pull request branched from, never from the pull request's own checkout; a pull request that changes the ignore file has its finding and new rows reported as not run.

The guard row is the one place code from the pull request runs: that one function, with the literal constants and helpers it uses. What the sandbox is: a static allowlist at extraction (imports limited to `re`, `os.path`, `posixpath`, `pathlib`, `urllib.parse`, `ipaddress`, `string`, `unicodedata`, `html`, `typing`, `shlex`, `fnmatch`; no double-underscore names or attributes, no `getattr`/`setattr`/`type`/`super`/`object`/`vars`/`globals`/`locals`, no class bodies; a function that uses any of these is reported as not run), then a separate Python interpreter started with no environment, no token and no path into the repository, a restricted set of builtins without file, process, network or reflection access, memory, CPU, file-size, open-file and process-count limits, and, when the Action runs as root, switched to the `nobody` user before the function loads (a run that cannot drop privileges is reported as not run, never run as root). What it is not: a container or virtual-machine boundary. It is a separate unprivileged process on the same runner, in the same job, and should be read as such. It runs only on `pull_request` events, never on `pull_request_target`, and `guard_bypass: "false"` switches it off. The list is finite and ships as plain YAML in the image; a pass means the function rejected every entry on it, nothing more.

**Status: beta (v0.x).** The rules are the ones behind Railo's published [failure-rate study](https://www.railo.dev/research/autofix-11-repositories), and they will miss things. The check run always shows the rule and line behind a fail, so a reviewer can judge it.

## Why

How often AI-written security patches are wrong is disputed. In August 2026, 1Password reported that 26% of 6,080 patches from two frontier models were fully correct ([their write-up](https://1password.com/blog/why-ai-generated-patches-still-require-human-review)). Trail of Bits reanalysed the same data, keeping only the trials where the models could run code and were not told to apply the wrong fix: 86% blocked the supplied exploit, which they note does not prove a complete fix ([their reanalysis](https://blog.trailofbits.com/2026/09/15/1passwords-ai-patching-benchmark-is-misleading/)). Their own records add that one in eight first fixes by developers did not fully resolve the issue.

Whoever writes a fix, it needs checking. GitHub counts an Autofix as successful only when the alert is gone, no new alert appeared, the file still parses and the tests still pass. Fix Gate applies that same bar to every fix pull request and reports each part separately.

## Install

Commit this as `.github/workflows/fix-gate.yml`:

```yaml
name: Fix Gate

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  checks: write

jobs:
  fix-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09  # v5.1.0
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          fetch-depth: 0
          persist-credentials: false
      - uses: IWEBai/fix-gate-action@v0
```

`fetch-depth: 0` matters: the gate compares against the commit the pull request branched from.

To also get the verdict as a pull request comment, add `pull-requests: write` under `permissions` and `with: comment: true` to the Fix Gate step.

To fill the tests row, run your tests in a job named `test` first, then:

```yaml
  fix-gate:
    needs: [test]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09  # v5.1.0
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
| `fail_on_not_run` | see note | `true` makes the conclusion failure, not neutral, when the finding or new rows could not run: the scan timed out or crashed, more than 200 files changed, the base commit was not found, the ignore file changed, or an external finding has no Railo rule match. Defaults to `true` when `rule`, `file`, `line`, `alert` or `sarif_base` is given or `require_resolved` is true, otherwise `false`. Rows not run by design (tests without `tests_result`, a language without a parser, `guard_bypass` off) never fail the gate. |
| `guard_bypass` | `true` | Runs the validator the pull request adds against the bypass list, in a separate unprivileged subprocess, on `pull_request` events only. `false` leaves that row as not run. |
| `base_branch` | | Branch the pull request targets; defaults to the pull request's base. |
| `github_token` | `github.token` | Token used to create the check run. |

Outputs: `conclusion` (`success`, `failure`, `neutral`) and one of `pass`, `fail`, `neutral`, `not_run` for each of `finding`, `new`, `parse`, `tests`, `guard`.

## What it reports as *not run*

- Findings from other tools (CodeQL, Snyk, a pentest) that Railo's rules do not also report. With `sarif_base`, a finding is matched to a Railo finding in the same file by CWE or line and confirmed gone only through that match; an unmatched one is not run. Without `sarif_base`, other tools' findings are not considered at all.
- Go, Ruby and C# files in the parse check.
- The tests row, unless `tests_result` is passed.
- A scan that timed out, or more than 200 changed files.
- A pull request that changes `.railoignore` (the finding and new rows). The patterns applied are the base commit's.
- A finding in a test, fixture or mock path: reported with a notice annotation, not judged, so a test's mock token or a test that writes outside the root on purpose does not fail the gate. Name the file as the target to have it judged.
- The guard row, when the validator needs something outside the sandbox (a framework setting, a network call, a class), uses a construct the sandbox refuses (a double-underscore name, `getattr`, `type`, and the like), takes more than one argument, is a method, takes too long on a payload, or when the event is not `pull_request`. The row names the reason. A fix that guards inline, inside the function that does the work, is *nothing to check*, and the row names that function.
- A finding whose file the pull request deleted is *nothing to check* ("file removed, not fixed") when a target was named or `require_resolved` is set; otherwise it counts as gone and the summary says the file was deleted.

With `fail_on_not_run` on, the first two of these fail the gate instead of leaving it neutral.

## Pull requests from forks

The token on a fork pull request is read-only, so the check run cannot be created. The verdict is written to the job summary instead, and the job's exit code still reflects it.

## Pull requests from AI tools

Fix Gate runs on any pull request, whoever wrote it: Copilot Autofix, Copilot's coding agent, Snyk, an editor assistant, or a person.

- **Copilot's coding agent.** GitHub does not start workflows on a pull request the agent pushes to until someone clicks **Approve and run workflows** on it. That is GitHub's rule for agent pull requests, not Fix Gate's; the check appears once you approve.
- **A fix for another tool's finding.** A Copilot Autofix pull request usually fixes a CodeQL alert. When Railo's rules report the same finding, Fix Gate confirms it is gone. When they do not, the first row reads *nothing to check* and the check reads *Fix not confirmed*, while the other rows are still checked and the summary lists what held. To judge the alert itself, pass its number as `alert` (with `security-events: read`) or the base scan as `sarif_base`.

## Permissions

`contents: read` to read the repository, `checks: write` to create the check run, `pull-requests: write` only if you ask for a comment. `security-events: read` only if you pass `alert`. Fix Gate never pushes and never writes under `.github/workflows`.

## Telemetry

One usage record per run: the organization and repository name, the plan, the mode (`fix-gate`), the Action version, the number of targeted and resolved findings, and the scan duration. Never file content. Set `RAILO_TELEMETRY: "0"` in the step's `env` and nothing is sent to Railo.

## Source

This repository holds the Action definition. The engine runs from Railo's image, `ghcr.io/iwebai/railo`, pinned by digest in `action.yml`, so a given tag of this Action always runs the same image. A comment next to the digest names the engine version.

The engine is not open source. Its Python modules are compiled in the image, so the gate logic cannot be read there. What you can inspect:

- **The rules.** They ship as plain YAML under `/app/rules` in the image, and the check run names the rule and line behind every fail.
- **The bypass list.** One YAML file per weakness class under `/app/guard_bypass/payloads` in the image, each entry with its technique and whether it is unsafe under any base or host; the check run quotes the entries a validator accepted.
- **What leaves the job.** The usage record described under Telemetry, and nothing at all to Railo when `RAILO_TELEMETRY` is `"0"`. No file content is sent to Railo either way.
- **The verdict.** Each row says pass, fail, nothing to check or not run, and why.

The same checks are how Railo verifies the fixes it writes itself in a [Fix Sprint](https://www.railo.dev/pentest-remediation).

## Security

To report a vulnerability in this Action or the engine image, see [SECURITY.md](SECURITY.md).

## Licence

The files in this repository are MIT licensed. The engine image is Railo's and is free to use through this Action.
