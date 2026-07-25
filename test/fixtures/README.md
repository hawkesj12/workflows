# CVE-lane fixture

`package-lock.json` here exists so the audit's CVE lane actually **runs** in this
repo instead of skipping. Without it this repo has no dependencies, so `deps: none`
was the honest setting — and the consequence was that the whole path

    download the pinned binary → verify its SHA-256 → scan → exit 0

had never once concluded green in the repo that owns it. Its only real evidence
lived in a caller. A change here could break it and `self-test` would still pass.

## Why `ms@2.1.3`

Tiny, dependency-free, and mature — the least likely thing to acquire an advisory
and turn this repo red for a reason unrelated to the change under test.

**If it does go red on the fixture:** that is not a false alarm to suppress. A real
advisory landed against a version pinned in this repo, and the audit told you. Bump
the pin here to a clean version and move on — it should take under a minute.

## What this does and does not prove

**Proves:** the binary downloads, the checksum matches, the scanner executes, and a
clean scan exits 0 — the four steps a wrapper used to swallow.

**Does not prove detection.** A vulnerable fixture would, but then the job is red by
design and you need `continue-on-error` plus an inverted assertion — reintroducing
the exact flag v1.2.0 exists to remove. Detection is proven in `repolens` instead: a
planted vulnerable manifest produced 11 advisories and a red job.
