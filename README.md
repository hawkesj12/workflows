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
emails you.

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
    uses: hawkesj12/workflows/.github/workflows/audit.yml@84b6fa2db9023aaed2f27eb07738437be6cb2f7c # v1.2.2
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

> The SHA above is whatever was current when this section was last edited, and
> nothing enforces that it stays current — check
> [Releases](https://github.com/hawkesj12/workflows/releases) for the latest. Once
> a repo is onboarded this stops mattering: Dependabot bumps the caller's pin, not
> this example.

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

Two consequences worth internalizing:

- **Check that your real dependency manifest is one osv-scanner reads.** The log
  line names every file it scanned — `Scanned .../uv.lock file and found 33
packages`. If your actual manifest is not in that list, the lane is not covering
  it, whatever colour the badge is.
- **A `.gitignore`d lockfile is skipped too**, but that case alone fails closed: a
  repo whose only lockfile is gitignored produces "No package sources found" and
  exit 128, which is red. The risk is the _combination_ — a gitignored lockfile
  beside a recognized one goes green while skipping the first.

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

**A quiet public repo loses its audit silently.** GitHub
[disables scheduled workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
in a **public** repository after 60 days with no repository activity. A finished
project that sits for two months stops being audited, and the mechanism that was
supposed to tell you things stops telling you. Private repos are unaffected. If you
onboard a repo you expect to go quiet, either touch it occasionally or accept that
its audit is dormant — don't assume green means checked.

## Releases

| Version  | Change                                                                                 |
| -------- | -------------------------------------------------------------------------------------- |
| `v1.2.2` | Bound every job with a timeout; narrow the CVE guarantee to what it provides |
| `v1.2.1` | Scorecard runs on the default branch only; skips elsewhere instead of failing          |
| `v1.2.0` | Run the pinned, checksum-verified osv-scanner binary; delete the lockfile precondition |
| `v1.1.0` | (superseded) lockfile precondition for the CVE lane                                    |
| `v1.0.0` | Initial: zizmor, lychee, osv-scanner, Scorecard                                        |
