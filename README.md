# workflows

Reusable GitHub Actions workflows, so the same checks run everywhere without a
copy per repo.

## Why this exists

Most checks fire when you push. But some problems appear **without your code
changing**:

- a CVE is published against a dependency you last touched months ago
- a link in your README rots because someone else moved a page
- an action tag gets repointed by whoever controls that repo

Nothing commit-triggered can catch those — there is no commit to trigger on. They
need a clock. That's what `audit.yml` is: four checks on a schedule, and a failure
emails **one specific person** — see [Who gets the email](#who-gets-the-email),
because it is probably not who you assume.

## Use it

Add this to any repo as `.github/workflows/audit.yml`:

```yaml
name: Audit

on:
  schedule:
    - cron: "0 13 * * 1" # Mondays 09:00 ET (cron is UTC; 13:00 UTC = 09:00 EDT)
  workflow_dispatch: # so you can also run it by hand

permissions:
  contents: read

jobs:
  audit:
    # ⬇ Copy the exact line from the latest release — see Versioning below.
    uses: hawkesj12/workflows/.github/workflows/audit.yml@<SHA> # <version>
    permissions:
      # The UNION of what every called job declares — see Notes. Omitting any of
      # these is a startup_failure with no log.
      contents: read
      id-token: write # Scorecard — required even when scorecard: false
    with:
      docs: "README.md CHANGELOG.md docs/"
      scorecard: true # public repos only — see below
```

That's the whole integration. Two things to get right, both covered in Notes:
commit a **lockfile** so the CVE scan has a target, and grant the **full
permissions block** above.

> **The `<SHA>` is a placeholder on purpose.** A hardcoded pin here is stale one
> release later and nothing enforces it — which is drift, in the repo whose whole
> premise is not drifting. So the exact line lives in
> [Releases](https://github.com/hawkesj12/workflows/releases) instead, where it is
> written at release time and cannot go out of date. The latest release body opens
> with a paste-ready `uses:` line.
>
> Pasting `<SHA>` literally is caught by this audit's own first job: zizmor's
> `unpinned-uses` flags it High and exits non-zero, so a wrong paste is loud on the
> very first run rather than silently a version behind. (`actionlint` does **not**
> catch it — measured, exit 0.) And once a repo is onboarded this stops mattering
> entirely: Dependabot bumps the caller's pin from then on.

## Versioning

Pin a **commit SHA with the version in a trailing comment**, as above — not
`@main`, and not a bare tag.

Not pinning `@main` isn't about trusting this repo; it's mine. It's that `@main`
has no staging channel: every caller's next scheduled run executes whatever was
last pushed here, and you learn from failure emails instead of a diff. A pin makes
`main` the place changes get tried — gated by `self-test.yml`, which runs the audit
against this repo on every PR — and the pinned commit the thing the fleet actually
runs.

The SHA rather than the tag is so callers pass their own `zizmor` audit without an
exception file. zizmor's `unpinned-uses` defaults to a blanket hash-pin policy, and
a `ref-pin` exception for `hawkesj12/*` would accept `@main` too — an exception that
doesn't enforce what it claims. A SHA needs no exception.

Dependabot maintains the pin either way — its `github-actions` ecosystem
[covers reusable-workflow references](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot),
so each caller gets its own PR on a new release, runs its own CI against it, and
converges on your merge.

## What it runs

| Job                 | Tool                                                         | Catches                                                  |
| ------------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| `workflow-security` | [zizmor](https://docs.zizmor.sh/)                            | template injection, credential leakage, unpinned actions |
| `links`             | [lychee](https://lychee.cli.rs/)                             | dead links in your docs                                  |
| `vulnerabilities`   | [osv-scanner](https://google.github.io/osv-scanner/)         | known CVEs in your dependencies                          |
| `posture`           | [OpenSSF Scorecard](https://openssf.org/projects/scorecard/) | branch protection, pinned deps, signed releases (opt-in) |

Jobs are independent — one failing never hides another.

## Inputs

| Input       | Default     | Notes                                                                                                                                   |
| ----------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `docs`      | `README.md` | space-separated files/globs for lychee                                                                                                  |
| `scorecard` | `false`     | **public repos only** — it needs `id-token: write` and publishes to the public OpenSSF API. Runs on the default branch only (see Notes) |
| `deps`      | `auto`      | `auto` scans; osv-scanner fails on its own if it finds no packages. `none` declares the repo dependency-free and **skips** the CVE job  |

## Notes

**Actions are pinned to SHAs, not tags.** A tag like `@v4` can be repointed by
whoever controls that repo, so it isn't a fixed thing you audited — it's whatever
they publish tomorrow. Dependabot is configured to bump these pins, which is what
makes SHA-pinning maintainable instead of rot.

**A check that examined nothing must not report success.** This is the design rule
the audit is built around, and it has its own failure mode distinct from a broken
check. A broken check has something wrong with it. A check aimed at empty space has
nothing wrong with it at all — the code is fine, the config is fine, and it passes.
A green shield then reads as "no CVEs" when it means "no scan."

Two of the four checks are exposed to this, and they are handled differently:

- **The links lane fails closed — but the guard is the action's, not lychee's.**
  Measured: the `lychee` binary given a link-free file exits **0**. The red comes
  from `lychee-action`'s `failIfEmpty`, which greps its own markdown summary for
  `Total | 0`. That is a third-party wrapper default doing load-bearing work, which
  is exactly what the CVE lane refuses wrappers for — so this audit **asserts**
  `failIfEmpty: true` rather than inheriting it. Two ways it would otherwise stop
  guarding silently: the default flips on a version bump, or a caller puts
  `--format json` in `docs` and the grep has no markdown to match. When it does
  fire, take it as a real signal — either the glob is wrong, or the doc should be
  linking things it isn't. (This is how it caught its own repo's unlinked README.)
- **`osv-scanner` also fails closed — but only if you run the binary.** It exits
  128 with "No package sources found", and both of Google's wrappers throw that
  away. The reusable workflow runs the scanner with `continue-on-error: true` and
  reports from a `results.json` the failed scan never wrote. The action alone
  downgrades exit 128 to `##[warning]No lockfiles found` and passes — its own log
  calls that deprecated. So this audit downloads the pinned, checksum-verified
  binary and runs it, which is the only path where the exit code survives. If a
  repo genuinely has no dependencies, say so with `deps: none` and the CVE job is
  **skipped**.

There is deliberately no setting that produces a green CVE job having scanned
**nothing**. It scans something, it skips, or it fails — and `deps: none` is a claim
written in your workflow file that a reviewer can disagree with, not an absence
nobody notices.

### What a green CVE job does and does not mean

That guarantee is narrower than it first reads, and the difference matters because
it is the same distinction the rest of this document polices.

**Exit 0 means: at least one package source was found, scanned, and was clean.** It
does **not** mean every dependency in the repo was examined. osv-scanner skips
manifest formats it does not support, silently, and a single recognized source is
enough to produce a green for the whole repo. Measured:

```
package-lock.json  (clean, 1 package)   ← recognized, scanned
environment.yml    (requests==2.19.1)   ← NOT a supported format, skipped
→ exit 0, "No issues found"
```

`requests==2.19.1` carries multiple published advisories. The job is green anyway.

Three consequences worth internalizing:

- **Check that your real dependency manifest is one osv-scanner reads.** The log
  names every file it scanned — `Scanned .../uv.lock file and found 33 packages`.
  If your actual manifest is not in that list, the lane is not covering it,
  whatever color the badge is. (That list exists because this workflow leaves
  `--verbosity` at its default of `info`; at `warn` or `error` the `Scanned` lines
  disappear entirely — verified.)
- **A `.gitignore`d lockfile is skipped too**, but that case alone fails closed: a
  repo whose only lockfile is gitignored produces "No package sources found" and
  exit 128, which is red. The risk is the _combination_ — a gitignored lockfile
  beside a recognized one goes green while skipping the first.
- **An `osv-scanner.toml` in your repo can suppress findings into a green.**
  `[[IgnoredVulns]]` entries are honored, and this audit does not pass
  `--no-config`. Verified: a `package-lock.json` pinning `lodash@4.17.15` exits 1
  with 6 advisories; adding an `osv-scanner.toml` listing those six IDs prints
  "No issues found" and exits **0**. That is a defensible mechanism — it is the
  same shape as `deps: none`, a claim written down that a reviewer can argue with
  — but it is a third way to reach green, so read it as part of the repo's audit
  surface rather than an implementation detail. If you want the audit to refuse
  caller-side suppression, add `--no-config` to your own fork of the scan step.

`--no-ignore` is deliberately **not** enabled, and the reason is narrower than it
first looks. It does **not** address the gap above: an unsupported format is skipped
whether or not it is gitignored, so the flag changes nothing about that case
(verified — exit 0 either way).

What it actually does is pull in gitignored _declared manifests_ — a gitignored
`vendor/requirements.txt` goes from skipped to scanned. It does **not** scan an
installed virtualenv: `scan source` has no installed-artifact extractor, so a
gitignored `.venv` full of `dist-info` is invisible with or without the flag
(verified — exit 128 both ways). Leave it off unless you deliberately gitignore a
real manifest, in which case turn it on per-repo.

### What the scan walks (monorepos and vendored trees)

The scan is `--recursive ./` from the repo root, so **every** committed manifest in
the tree is scanned, not just the one at the top. Measured on a fixture with three
lockfiles:

```
svc-a/package-lock.json              ← scanned
vendor/thirdparty/package-lock.json  ← scanned  (vulnerable → whole audit red)
node_modules/x/package-lock.json     ← NOT scanned (matched .gitignore)
```

Two things follow, and neither is obvious:

- **A monorepo gets full coverage for free.** Every service's lockfile is audited by
  the one job. No per-package configuration.
- **A committed vendored dependency turns your audit red for code you did not
  write.** If you check in a third-party tree that carries its own lockfile, its
  CVEs are your red. That is arguably correct — you are shipping that code — but it
  is a surprise the first time. Gitignoring the vendored tree excludes it, since the
  walk respects `.gitignore` by default (which is also why a large `node_modules`
  costs nothing here).

### The CVE lane depends on two network services

A green needs both `api.osv.dev` (the vulnerability query) and, for _manifest_ files
rather than lockfiles, `deps.dev` (transitive dependency resolution). Either being
unreachable is **exit 127** — a red caused by someone else's outage, on a repo that
did nothing wrong. Verified: a clean `requirements.txt` with the network blocked
gives `max retries exceeded ... api.osv.dev/v1/querybatch` and exit 127, never 0.

That is the honest tradeoff for refusing wrappers: this lane fails closed on
_everything_, including infrastructure. It is the right default — an unavailable
scan is not a clean scan — but if a flaky resolver becomes noise for a
manifest-based repo, `--no-resolve` drops the `deps.dev` dependency. Know what it
costs: on `requirements.txt` pinning `requests`, resolution found 2 packages and 4
advisories; `--no-resolve` found 1 package and 2, missing the transitive `idna`
finding entirely. Resilience bought with coverage — usually the wrong trade, which
is why it is off.

The general lesson, which cost two wrong fixes to learn: **check what your tooling
does with the exit code before building anything around it.** v1.1.0 added a
lockfile-detection job that covered exactly one of the many ways a scan can fail
silently. The first attempt at v1.2.0 dropped that job for Google's action, on the
assumption it preserved the exit code — it does not. Only running the binary does,
and that was settled by pushing each version at a repo with no lockfile and reading
the result, not by reasoning about it.

**A red that always fires is the same disease as a meaningless green.** Both teach
you to stop reading the signal. That's why Scorecard runs on the **default branch
only** (v1.2.1): it exits `validating options: only default branch is supported` on
any other ref, so a caller using `workflow_dispatch` from a branch used to get a
guaranteed red for a reason unrelated to its repo. It now **skips** there instead —
"not applicable here" rather than a lie.

**A caller must grant the union of permissions every called job declares.**
Permissions only narrow down a reusable-workflow chain. Granting less than the
callee asks for is a `startup_failure` with no jobs, no log, and no annotation —
the hardest failure here to diagnose. Note that a job's `if:` does not exempt it:
the chain is validated before any condition is evaluated, so the Scorecard job's
`id-token: write` is required even when `scorecard: false`.

```yaml
permissions:
  contents: read
  id-token: write # Scorecard — required even when scorecard: false
```

Callers upgrading from v1.1.0 can drop `actions: read` and
`security-events: write`. Those were Google's reusable workflow's requirements;
v1.2.0 no longer uses it.

### Who gets the email

A scheduled run notifies **one user, not the repo's watchers and not a team**. Per
GitHub's
[notification docs](https://docs.github.com/en/actions/concepts/workflows-and-actions/notifications-for-workflow-runs):
notifications go to "the user who initially created the workflow"; if someone else
"updates the cron syntax, in the `schedule` event in the workflow file, subsequent
notifications will be sent to that user instead"; and if the workflow is disabled and
re-enabled, they go to whoever re-enabled it.

So the alert follows **whoever last touched the `cron:` line** — an attribute of the
file's history, not a role, not the repo, not a rota. Three consequences:

- **On an org repo, the audit can go unwatched while still running.** The engineer
  who onboarded the repo leaves; the emails keep going to a dead address; every job
  stays green-or-red on a dashboard nobody opens.
- **Reassigning it is a one-character commit.** Edit the `cron:` line and the
  notifications follow you.
- **If it matters, don't rely on the email.** Route the signal somewhere durable —
  a failure step that posts to a channel, or a scheduled check of the runs API.

An audit whose alert is addressed to a person who left is the same failure as a
green that scanned nothing: the machinery works, nobody hears it.

**A quiet public repo loses its audit silently.** GitHub
[disables scheduled workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
in a **public** repository after 60 days with no repository activity. A finished
project that sits for two months stops being audited, and the mechanism that was
supposed to tell you things stops telling you. Private repos are unaffected. If you
onboard a repo you expect to go quiet, either touch it occasionally or accept that
its audit is dormant — don't assume green means checked.

## Releases

[**Releases**](https://github.com/hawkesj12/workflows/releases) — what changed in each
version, and the paste-ready pin for the latest.

There is deliberately no changelog table here. A hand-maintained copy of the release
list is the same defect as a hardcoded pin: it is written once, nothing checks it, and
it is wrong by the next release. This one went stale three times in a day before it
was deleted. The release bodies are written at release time and cannot drift from
themselves.
