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
    uses: hawkesj12/workflows/.github/workflows/audit.yml@v1.0.0
    permissions:
      # The UNION of what every called job declares — see Notes. Omitting any of
      # these is a startup_failure with no log.
      contents: read
      actions: read # osv-scanner
      security-events: write # osv-scanner
      id-token: write # Scorecard — required even when scorecard: false
    with:
      docs: "README.md CHANGELOG.md docs/"
      scorecard: true # public repos only — see below
```

That's the whole integration. Two things to get right, both covered in Notes:
commit a **lockfile** so the CVE scan has a target, and grant the **full
permissions block** above.

## Versioning

Pin a **tag**, not `@main`. Not because this repo is untrusted — it's mine — but
because `@main` has no staging channel: every caller's next scheduled run
executes whatever was last pushed here, and you learn from failure emails instead
of a diff. A tag makes `main` the place changes get tried (gated by
`self-test.yml`, which runs the audit against this repo on every PR) and the tag
the thing the fleet actually runs.

Dependabot maintains the pin for you — its `github-actions` ecosystem
[covers reusable-workflow references](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot),
so each caller gets its own PR on a new release and converges on your merge.

## What it runs

| Job                 | Tool                                                         | Catches                                                  |
| ------------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| `workflow-security` | [zizmor](https://docs.zizmor.sh/)                            | template injection, credential leakage, unpinned actions |
| `links`             | [lychee](https://lychee.cli.rs/)                             | dead links in your docs                                  |
| `vulnerabilities`   | [osv-scanner](https://google.github.io/osv-scanner/)         | known CVEs in your dependencies                          |
| `posture`           | [OpenSSF Scorecard](https://openssf.org/projects/scorecard/) | branch protection, pinned deps, signed releases (opt-in) |

Jobs are independent — one failing never hides another.

## Inputs

| Input       | Default     | Notes                                                                                                                         |
| ----------- | ----------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `docs`      | `README.md` | space-separated files/globs for lychee                                                                                        |
| `scorecard` | `false`     | **public repos only** — it needs `id-token: write` and publishes to the public OpenSSF API                                    |
| `deps`      | `auto`      | `auto` requires a lockfile and **fails** if there is none; `none` declares the repo dependency-free and **skips** the CVE job |

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

- **`lychee` fails closed on its own.** Given files with no links it exits non-zero
  with "No links were found. This usually indicates a configuration error." Take it
  as a real signal: either the glob is wrong, or the doc should be linking things it
  isn't. (This is exactly how it caught its own repo's unlinked README.)
- **`osv-scanner` does not**, so `deps` is enforced from the outside. It reads
  lockfiles; a repo with only loose ranges in `pyproject.toml` finds no package
  sources and passes. So a `lockfile present` job runs first and **fails the audit**
  when it finds no tracked lockfile. If a repo genuinely has no dependencies, say so
  with `deps: none` and the CVE job is **skipped** rather than passed.

There is deliberately no setting that produces a green CVE job without a lockfile.
It scans something, it skips, or it fails — and `deps: none` is a claim written in
your workflow file that a reviewer can disagree with, not an absence nobody notices.

**A caller must grant the union of permissions every called job declares.**
Permissions only narrow down a reusable-workflow chain. Granting less than the
callee asks for is a `startup_failure` with no jobs, no log, and no annotation —
the hardest failure here to diagnose. Note that a job's `if:` does not exempt it:
the chain is validated before any condition is evaluated, so the Scorecard job's
`id-token: write` is required even when `scorecard: false`.

```yaml
permissions:
  contents: read
  actions: read # osv-scanner
  security-events: write # osv-scanner
  id-token: write # Scorecard — required even when scorecard: false
```
