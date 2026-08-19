# Addon CI, Security, and Upstream PR Pipeline

**Reproduction target:** [`yacketrj/dune-ops-observability-addon`](https://github.com/yacketrj/dune-ops-observability-addon) @ `2227b67` | **Status:** Current

This document is a reproduction guide, not a tutorial on writing an addon.
It walks through the **exact, real CI workflows, validation scripts,
security scanners, and packaging/PR automation** already committed to the
reference addon repository above, so that another addon author can run
the same checks, produce the same kind of evidence, and submit a
correctly-formed catalog PR to this repository.

Every command, script path, and workflow name below is copied from the
real repository as of commit `2227b67` (2026-08-17). Nothing here is
invented or generalized — if a script takes an argument, the argument
format shown is the argument format the real script requires. Adapt the
paths (`web/`, `addon.json`, script names) to your own addon's layout if
it differs, but the pipeline shape — validate, test, scan, package,
verify, submit — is the one to reproduce.

Sections: [Repository layout](#1-repository-layout) ·
[GitHub Actions workflows](#2-the-9-github-actions-workflows) ·
[Validation scripts](#3-validation-scripts) ·
[Local security gates](#4-local-security-gates) ·
[Packaging](#5-packaging) ·
[Release validation gate](#6-release-validation-gate) ·
[Verify release checksum](#7-verify-the-release-asset-checksum) ·
[Submit the catalog PR](#8-submitting-the-catalog-pr-to-this-repository) ·
[End-to-end sequence](#9-end-to-end-command-sequence) ·
[Troubleshooting](#troubleshooting)

## 1. Repository layout

The reference addon repository has three directories relevant to this
pipeline:

```text
dune-ops-observability-addon/
├── .github/workflows/       9 CI workflow files (see §2)
├── .pre-commit-config.yaml  local + CI pre-commit hook definitions
├── .gitleaksignore          gitleaks false-positive allowlist
├── .semgrepignore           semgrep exclusion list
├── .trivyignore             trivy exclusion list
├── addon.json               the addon manifest (id, version, permissions, downloadUrl, sha256)
├── package.json             devDependency-only; "test": "node --test test/*.test.js"
├── scripts/                 validation, packaging, release, checksum, and catalog-PR scripts (§3, §5-8)
├── ops-observability/
│   └── dev-tools/           local security-gate wrapper scripts (§4)
├── test/                    node --test unit + governance tests
└── web/                     the addon's shipped UI code (validated and packaged, not built)
```

Nothing here requires a bundler or build step. The addon ships as plain
HTML/CSS/JS. All of the tooling below is either a shell script calling a
real CLI tool (git, node, zip, sha256sum, gh) or a small dependency-free
Node script.

## 2. The 9 GitHub Actions workflows

All third-party actions in every workflow below are pinned to a full
commit SHA with a version comment (e.g.
`actions/checkout@9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0 # v7.0.0`),
not a mutable tag — copy this pattern into your own workflows.

### `ci.yml` — the main gate

Triggers: `pull_request`, `push` to `main`, `workflow_dispatch`.

Five jobs run in parallel, then a `ci-gate` job requires all five to have
succeeded before the workflow reports success:

| Job | Command(s) |
|---|---|
| `shellcheck` | `shellcheck --severity=warning $(find . -name "*.sh" -not -path "./node_modules/*" -not -path "./.git/*")` |
| `validate-json` | `find . -name "*.json" -not -path "./node_modules/*" -not -path "./.git/*" -exec python3 -c "import json,sys; json.load(open(sys.argv[1]))" {} \;` |
| `npm-audit` | `npm ci --ignore-scripts \|\| true` then `npm audit --audit-level=moderate` |
| `unit-tests` | Full-history checkout (`fetch-depth: 0`, required because `check-version-consistency.js` does ancestor lookups against every tag), then in order: `node scripts/validate.js`, `node scripts/check-bridge-action-drift.js`, `node scripts/check-version-consistency.js`, `node scripts/check-sri-integrity.js`, `npm test` |
| `security` | `gitleaks/gitleaks-action@v3`, then `semgrep scan --config p/default --config p/secrets --error`, then `aquasecurity/trivy-action@v0.36.0` with `scanners: vuln,misconfig,secret`, `severity: CRITICAL,HIGH`, `exit-code: 1` |

The `ci-gate` job runs `if: ${{ always() }}` and fails the workflow if
any of the five upstream jobs did not report `success`.

### `validate.yml` — minimal manifest check

Triggers: `pull_request`, `push` to `main`, `workflow_dispatch`.

One job, one step: `node scripts/validate.js`. This is the smallest
possible required check — every other workflow either wraps or extends
this same script.

### `pre-commit.yml`

Triggers: `pull_request`, `push` to `main`, `workflow_dispatch`.

Installs `pre-commit`, `semgrep==1.170.0`, Trivy (via
`aquasecurity/setup-trivy`), and `npm ci` (needed because the local
`unit-tests` pre-commit hook runs `npm test`, which needs `jsdom`), then
runs:

```bash
pre-commit run --all-files --show-diff-on-failure
```

### `secret-scan.yml`

Triggers: `pull_request`, `push` to `main`, `workflow_dispatch`. Full
history checkout (`fetch-depth: 0`), then `gitleaks/gitleaks-action@v3`.

### `sast.yml`

Triggers: `pull_request`, `push` to `main`, `workflow_dispatch`.

```bash
python3 -m pip install semgrep==1.170.0
semgrep scan --config p/default --config p/secrets --error
```

### `filesystem-scan.yml`

Triggers: `pull_request`, `push` to `main`, `workflow_dispatch`.

`aquasecurity/trivy-action@v0.36.0` with `scan-type: fs`, `scanners:
vuln,misconfig,secret`, `severity: CRITICAL,HIGH`, `exit-code: 1`.

### `security-gates.yml` — the scheduled superset

Triggers: `pull_request`, `push` to `main`, weekly cron (`17 9 * * 1`,
Monday 09:17 UTC), `workflow_dispatch`.

Five jobs: `dependency-audit` (npm audit), `dependency-review` (PR-only,
`actions/dependency-review-action@v5`, `fail-on-severity: moderate`),
`semgrep-sast`, `secret-scan` (gitleaks), `trivy-filesystem`. This is a
broader, scheduled version of the checks already in `ci.yml` and the
standalone files above — it exists so a dependency going bad *between*
PRs (not just at PR time) still gets caught weekly.

### `semgrep.yml` — Semgrep AppSec Platform integration

Triggers: `workflow_dispatch`, `pull_request`, `push` to `main`/`master`
(path-filtered to itself only), daily cron (`47 0 * * *`).

Runs inside the official `semgrep/semgrep` container:

```bash
semgrep ci
```

This uses a `SEMGREP_APP_TOKEN` repository secret to report to the
Semgrep AppSec Platform, separate from the local `semgrep scan` calls
used elsewhere. If you don't have a Semgrep AppSec Platform account,
skip this workflow — the `sast.yml`/`ci.yml` semgrep jobs above don't
need the token.

### `release.yml` — tag-triggered release automation

Trigger: `push` of a tag matching `v*`.

> This workflow calls `scripts/package.sh` (detailed in §5) but does
> **not** run `scripts/validate-release.sh` (§6) automatically — that
> gate is a manual step you run yourself before tagging, not part of
> CI. See §5 and §6 for what those two scripts actually do.

Steps, in order:

1. **Full-history checkout** (`fetch-depth: 0`).
2. **Ancestor guard** — refuses to proceed unless the tagged commit is
   reachable from `origin/main`:
   ```bash
   git fetch origin main
   git merge-base --is-ancestor "$GITHUB_SHA" origin/main
   ```
   This exists because of a real, documented incident: a tag can be
   created and pushed from a commit that was never merged to `main`,
   producing a published GitHub Release with no real corresponding
   point in the project's history. Reproduce this guard in your own
   `release.yml` — it is a one-command, high-value check.
3. **Validate**: `node scripts/validate.js`.
4. **Tag/manifest version match check**:
   ```bash
   ADDON_VERSION="$(node -e "process.stdout.write(require('./addon.json').version)")"
   TAG_VERSION="${GITHUB_REF_NAME#v}"
   [ "$ADDON_VERSION" = "$TAG_VERSION" ]
   ```
5. **Package**: `bash scripts/package.sh` (§5).
6. **Generate SBOM** (Software Bill of Materials — a machine-readable
   dependency inventory, in CycloneDX JSON format). This step can
   silently no-op (missing `package.json` or a `cyclonedx` install
   failure both fall through to `|| true`):
   ```bash
   npm install --ignore-scripts || true
   npx @cyclonedx/cyclonedx-npm --output-file dist/sbom.json --output-format json || true
   ```
7. **Create the GitHub Release**, attaching the zip and its `.sha256`
   sidecar unconditionally, and `dist/sbom.json` **only if that file
   was actually produced** in step 6 — if SBOM generation silently
   no-op'd, the release ships with just the zip and its checksum:
   ```bash
   RELEASE_ARGS="$PACKAGE_PATH ${PACKAGE_PATH}.sha256"
   [ -f dist/sbom.json ] && RELEASE_ARGS="$RELEASE_ARGS dist/sbom.json"
   gh release create "$GITHUB_REF_NAME" $RELEASE_ARGS \
     --title "$ADDON_NAME v$ADDON_VERSION" \
     --notes "Addon package, SHA-256 checksum, and CycloneDX SBOM."
   ```

## 3. Validation scripts

These are the scripts every workflow above ultimately calls. Reproduce
them directly — they have no dependency on this specific addon beyond
the file paths noted.

### `scripts/validate.js` — manifest and shipped-code validator

Run: `node scripts/validate.js`

Checks, in order:

- `addon.json` parses as JSON and has all required fields (`id`, `name`,
  `description`, `author`, `version`, `type`)
- `schemaVersion === 1`, `type === "ui"`
- `version` matches `^\d+\.\d+\.\d+$` (plain semver, no `v` prefix)
- `entry.path` is set and the file it points to exists on disk
- **every array entry under `permissions.<scope>` equals `"read"`**
  (e.g. `"ops": ["read"]`) — any non-`read` action anywhere in any
  scope's array is a hard failure, enforcing a read-only-by-default
  policy at validation time rather than by convention alone
- every `src="..."` reference inside the entry HTML file resolves to a
  real file on disk (cache-busting query strings like `?v=0.5.1` are
  stripped before the check; `data-src="..."` attributes are
  deliberately excluded via a negative-lookbehind regex, since this
  addon uses `data-src` for a lazy-loaded external iframe URL that has
  no local file to check)
- every `.js` file under `web/` at least parses (`new Function(content)`)
- the version string embedded in the entry HTML file matches
  `addon.json`'s `version`

Exit code is non-zero with a `FAIL:` line per violation if anything
fails; otherwise prints `Addon manifest is valid: <id> v<version>`.

### `scripts/check-version-consistency.js` — three-way version agreement

Run: `node scripts/check-version-consistency.js`

Hard-fails if `addon.json.version`, `package.json.version`, and the
version string parsed out of the shipped HTML (`web/index.html`) don't
all agree. This exists because a version bump that touches some but not
all three files is always a real bug — these should never legitimately
disagree.

Separately (informational only, never fails the build): compares the
current version against the **latest release tag that is both a valid
semver tag and an ancestor of `main`** — deliberately not just the
highest-numbered or most-recently-created tag (see
`scripts/governance-lib.js` below). Being ahead of the latest real
release during active development is normal and expected.

### `scripts/governance-lib.js` — shared ancestor-aware tag logic

Not run directly; imported by `check-version-consistency.js`. Provides:

- `isAncestorOfMain(commitish)` — wraps `git merge-base --is-ancestor`
- `listVersionTags()` — lists all `v*` tags with their peeled commit SHAs
- `findLatestRealReleaseTag()` — the core anti-fabricated-release
  mechanism: filters tags to those matching `vMAJOR.MINOR.PATCH` **and**
  reachable from `main`, then returns the highest version among those.
  This exists specifically because a naive "highest-numbered tag" or
  "most-recently-created tag" approach is exactly what let previous
  fabricated releases pass unnoticed — a tag can look newer while never
  having been merged to `main`.

If you maintain your own addon, copy this file as-is; it has no
addon-specific logic.

### `scripts/check-sri-integrity.js` — Subresource Integrity drift check

Run: `node scripts/check-sri-integrity.js`

Parses every `<script src="..." integrity="sha384-...">` tag in the
entry HTML file, recomputes the real SHA-384 of the file each tag
references, and fails if any declared hash doesn't match the file's
actual current content.

This exists because of a real, documented incident (GitHub issue #119):
SRI hashes were hand-maintained, drifted out of sync with the actual
script content across nine commits, and because SRI failures are
enforced silently by the browser (no console error, the script simply
never executes), the addon shipped completely non-functional for over a
week before anyone noticed. If your addon uses SRI hashes on its
`<script>` tags at all, reproduce this check and run it in CI — it is a
zero-false-positive, purely mechanical comparison.

### `scripts/update-sri.js` — regenerates the hashes this check verifies

Run: `node scripts/update-sri.js`

Recomputes the real SHA-384 of every script referenced by an
`integrity="sha384-..."` tag in the entry HTML, rewrites the tag with
the correct hash, and bumps every cache-busting `?v=<timestamp>` query
string (including the stylesheet `<link>` tag) to the same "now"
timestamp so every asset invalidates together. Run this any time you
edit a file that a `<script integrity="...">` tag references, then
re-run `check-sri-integrity.js` (or `npm test`) to confirm.

### `scripts/check-bridge-action-drift.js` — README-vs-code drift check

Run: `node scripts/check-bridge-action-drift.js`

If your addon documents a list of backend API/bridge actions it calls
(this addon has a "Current bridge-backed actions" table in `README.md`),
this script extracts every action string actually called in the source
(`web/data-providers.js`, matching both direct call patterns and wrapper
call patterns) and fails if the two sets — documented vs. actually
called — disagree in either direction.

The README-parsing half handles a shorthand table format: a row like
`` `ops.health.summary.v2` / `.players` / `.farms` `` means three
actions, not one — every backtick-quoted segment after the first is a
suffix applied to the base action's 2-segment namespace prefix (here,
`ops.health`), not a full action string on its own. The script throws a
hard error (not just a check failure) if a shorthand row's base action
has fewer than 2 dot-segments, or if a suffix segment doesn't start with
`.` — both indicate the table itself is malformed, not that code and
docs merely disagree.

Adapt the file paths if your addon's documentation/code layout differs;
the pattern (parse both sources, diff the sets, fail on any mismatch) is
what to reproduce.

## 4. Local security gates

These wrap the same underlying tools the CI workflows use, but run
locally with concise `PASS:`/`FAIL:` output instead of full scanner
noise — full diagnostics print automatically only on failure. All live
under `ops-observability/dev-tools/`.

> **Note on "gate" terminology:** the scripts in this section
> (`precommit-gate.sh`, `pr-gate.sh`, `release-gate.sh`, etc.) are
> unrelated to `ci.yml`'s `ci-gate` job (§2) and to
> `create-upstream-addon-pr.sh`'s internal "Gate 0–4" stage numbers
> (§8) — all three use the word "gate" independently for "a check that
> must pass before proceeding."

### `toolchain-bootstrap.sh` — installs whatever is missing

Run: `bash ops-observability/dev-tools/toolchain-bootstrap.sh [tool ...]`

With no arguments, ensures `git node npm python3 pipx pre-commit
gitleaks semgrep trivy zip` are all installed, attempting apt → pipx/pip
→ Go → Homebrew → the tool's own upstream installer script, in that
order, per tool. Every other gate script below calls this first (unless
`DUNE_TOOLCHAIN_BOOTSTRAP_DONE=1` is already set in the environment).

Disable auto-install and just get a clear failure listing what's
missing:

```bash
DUNE_AUTO_INSTALL_TOOLS=0 bash ops-observability/dev-tools/toolchain-bootstrap.sh
```

### `precommit-gate.sh`

Run: `bash ops-observability/dev-tools/precommit-gate.sh`

```bash
pre-commit run --all-files --show-diff-on-failure
```

with output filtered to just the interesting lines on failure (hook
names, "Failed", modified-files notices) instead of the full pre-commit
transcript.

### `gitleaks-gate.sh`

Run: `bash ops-observability/dev-tools/gitleaks-gate.sh`

```bash
gitleaks detect --source . --no-git --report-format json --report-path <tmp>
```

On failure, parses the JSON report and prints each finding's file, line,
and rule ID.

### `semgrep-gate.sh`

Run: `bash ops-observability/dev-tools/semgrep-gate.sh`

```bash
semgrep scan --error --json-output <tmp>
```

On failure, parses the JSON and prints each finding's file, line,
severity, rule ID, and first line of the message.

### `trivy-gate.sh`

Run: `bash ops-observability/dev-tools/trivy-gate.sh`

```bash
trivy fs --exit-code 1 --severity UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL --format json --output <tmp> .
```

On failure, parses the JSON and prints vulnerabilities, secrets, and
misconfigurations found, each with target/severity/ID.

### `security-shift-left.sh` — runs all of the above in one call

Run: `bash ops-observability/dev-tools/security-shift-left.sh`

In order: toolchain bootstrap, `git diff --check` (merge-conflict-marker
and whitespace-error scan), `node scripts/validate.js`,
`precommit-gate.sh`, `gitleaks-gate.sh`, `semgrep-gate.sh`,
`trivy-gate.sh`. Reports a final `PASS: shift-left security gate` or
`FAIL: shift-left security gate (<n> failure(s))` summarizing every
step. **This is the single command to run locally before opening any
PR.**

### `pr-gate.sh` — adds branch/tree hygiene checks on top

Run: `bash ops-observability/dev-tools/pr-gate.sh`

Adds, before running `security-shift-left.sh`: `git fetch origin
--prune`, a clean-working-tree check, a check that the current branch
actually has commits over `origin/main` (or `$BASE_REF`), a check that
there are changed files at all, and `git diff --check` scoped to the
branch diff. Use this instead of `security-shift-left.sh` directly when
you're about to open a PR, not just checking your working tree.

### `release-gate.sh` — adds packaging and evidence-directory checks

Run: `bash ops-observability/dev-tools/release-gate.sh`

Adds `bash scripts/package.sh` and requires an evidence directory to
exist at `ops-observability/evidence/releases/$RELEASE_ID/` with
`testing/`, `security/`, `sbom/`, and `controls/` subdirectories present
(`RELEASE_ID` defaults to `unreleased`). If you don't maintain a formal
evidence-bundle convention, skip the evidence-directory checks and keep
the rest.

### `scripts/pre-release-security.sh` — full evidence-producing scan

Run: `bash scripts/pre-release-security.sh <version> [--ci]`

A heavier, evidence-producing counterpart to the gate scripts above.
Runs gitleaks, trivy (`--scanners secret,misconfig`), semgrep, `npm
audit` (if `console/api/package-lock.json` exists — not applicable to
every addon layout), and shellcheck against every `.sh` file under
`scripts/`, `pipeline/`, and `ops-observability/`, writing each tool's
raw report to
`ops-observability/evidence/releases/<version>/security/`. The `--ci`
flag skips an API DAST step that assumes a locally running Console
instance — pass it in any CI context. This is the script to run if you
need an auditable, on-disk evidence bundle rather than just a
pass/fail terminal result.

### `scripts/validate-and-install-local-console.sh` — private local install

Run: `bash scripts/validate-and-install-local-console.sh`
(reads `ADDON_REPO` and `CONSOLE_DIR` environment variables, both
hardcoded to specific local paths by default in the reference script —
override both for your own machine)

Syncs your addon repo to `origin/main`, runs `pre-commit run
--all-files`, `node scripts/validate.js`, and `bash scripts/package.sh`,
then syncs a local Dune Docker Console checkout to `upstream/main`,
copies your addon's `addon.json` and `web/` into that Console's
`runtime/addons/installed/<addon-id>/`, and writes an enabled/
approved-permissions entry into the Console's `runtime/addons/
state.json`. This is environment-specific (it assumes you have a local
Console checkout to test against) and not part of CI — reproduce it
only if you have such a checkout available for manual pre-release
smoke testing.

## 5. Packaging

### `scripts/package.sh`

Run: `bash scripts/package.sh`

```bash
node scripts/validate.js

ADDON_ID="$(node -e "process.stdout.write(require('./addon.json').id)")"
ADDON_VERSION="$(node -e "process.stdout.write(require('./addon.json').version)")"
PACKAGE_NAME="${ADDON_ID}-${ADDON_VERSION}.zip"

rm -rf dist
mkdir -p dist
zip -r "dist/${PACKAGE_NAME}" addon.json web -x "*.DS_Store"

sha256sum "dist/${PACKAGE_NAME}" | tee "dist/${PACKAGE_NAME}.sha256"
```

Validates first, then zips exactly `addon.json` plus the `web/`
directory (adjust to your own addon's shipped-file list), and writes a
`.sha256` sidecar next to the archive. Requires `node` and `zip` on
`PATH`; fails immediately with a clear message if either is missing.

## 6. Release validation gate

### `scripts/validate-release.sh`

Run: `bash scripts/validate-release.sh <version>` (bare version, e.g.
`0.5.1`, no `v` prefix — must match `addon.json`'s `version` field)

Five checks, each printed as `PASS:`/`FAIL:`, with a final pass/fail
count:

1. **No merge conflict markers** anywhere in `web/`, `scripts/`, or
   `addon.json` (`grep -rn '<<<<<<<\|=======\|>>>>>>>'`)
2. **`addon.json`'s version matches** the version argument, and
   `node scripts/validate.js` passes
3. **No stale version labels** in the shipped HTML. Note: the real
   script's pattern here is a literal, hardcoded string
   (`grep -nE 'v0\.3\.0|Release 0\.3[^.]'` — a leftover from an old
   release), not a dynamically computed "previous version" — reproduce
   this as a pattern you manually update after each release, not as a
   self-updating check.
4. **The most recently built zip's SHA-256 matches** `addon.json`'s
   recorded `sha256`, the unzipped contents have no conflict markers,
   and the version string appears at least once in the packaged HTML
5. **`SKIP="trivy,semgrep" pre-commit run --all-files`** passes
   (trivy/semgrep are deliberately skipped here — they're already
   covered by the security gates in §4)

Run this after `scripts/package.sh` and before tagging. It exits
non-zero with "Release validation FAILED" if any check fails.

## 7. Verify the release asset checksum

### `scripts/verify-release-asset-checksum.sh`

Run: `bash scripts/verify-release-asset-checksum.sh <version>
[expected-sha256]` (bare version, no `v` prefix)

```bash
TAG="v$VERSION"
ASSET="$ADDON_ID-$VERSION.zip"
DOWNLOAD_URL="https://github.com/$REPO/releases/download/$TAG/$ASSET"

curl --fail --location --silent --show-error --output "$WORK_DIR/$ASSET" "$DOWNLOAD_URL"
ACTUAL_SHA="$(sha256sum "$WORK_DIR/$ASSET" | awk '{print $1}')"
```

Downloads the **actual uploaded GitHub Release asset** (not your local
`dist/` copy — local zip contents can legitimately differ from what
GitHub serves) and computes its SHA-256. If you pass a second argument,
it's compared against the downloaded file's real checksum and the
script exits non-zero on any mismatch. It always prints the exact
`version` / `downloadUrl` / `sha256` fields to paste into a catalog
manifest:

```text
Community manifest fields:
  "version": "0.5.1",
  "downloadUrl": "https://github.com/<you>/<addon-repo>/releases/download/v0.5.1/<id>-0.5.1.zip",
  "sha256": "<the real, verified checksum>"
```

**This is the only checksum value that should ever go into your
`addon.json` or a catalog manifest.** Never hand-copy a checksum from a
local build.

## 8. Submitting the catalog PR to this repository

### `scripts/create-upstream-addon-pr.sh`

Run: `bash scripts/create-upstream-addon-pr.sh <version> [--draft]`
(bare version, no `v` prefix)

This is the script that opens the actual pull request against this
catalog repository (`Red-Blink/dune-docker-addons`). It requires you to
have already forked this repository and cloned that fork locally. Its
internal stages are numbered "Gate 0" through "Gate 4" (five stages,
zero-indexed) and run in order:

**Gate 0 — Preflight** (runs in your addon repo): reads `addon.json`,
confirms its `version` matches the argument you passed, extracts
`downloadUrl` and `sha256`.

**Gate 1 — Validation** (runs in your addon repo):
```bash
node scripts/validate.js
SKIP="trivy,semgrep" pre-commit run --all-files
bash scripts/verify-release-asset-checksum.sh "$VERSION"
```
Note: `verify-release-asset-checksum.sh` is called here with **no
expected-checksum argument**, so this step only confirms the release
asset exists and downloads successfully — it does **not** compare the
downloaded asset against `addon.json`'s recorded `sha256`. That
comparison only happens if you separately run
`verify-release-asset-checksum.sh <version> <expected-sha>` yourself
(§7). trivy/semgrep are skipped here since they're already covered by
the security gates in §4.

**Gate 2 — Catalog branch** (runs in your local clone of your fork of
this repository):
```bash
git fetch upstream main --quiet
git checkout upstream/main --detach --quiet
git branch -D "catalog-${VERSION}" 2>/dev/null || true
git checkout -b "catalog-${VERSION}" --quiet
```
(the `branch -D` line makes branch creation idempotent if you re-run
this for the same version), then copies your `addon.json` to
`addons/<your-addon-id>.json`, sets its `version` and `sha256` to the
verified values from Gate 0, and updates your addon's short entry in
`index.json` (`version`, `description` — read live from your real
`addon.json`, never hardcoded).

**Gate 3 — Commit and push**:
```bash
git add addons/<your-addon-id>.json index.json
git commit -m "catalog: update <Your Addon Name> to $VERSION" --no-verify
git push origin "catalog-${VERSION}" --force
```
(`--no-verify` because this catalog repository has no
`.pre-commit-config.yaml` of its own — Gate 1 already ran your addon
repo's real checks before any of this.)

**Gate 4 — Create the PR**:
```bash
gh pr create --repo Red-Blink/dune-docker-addons \
  --head "<your-github-username>:catalog-${VERSION}" \
  --base main \
  --title "<Your Addon Name> ${VERSION}" \
  --body "<structured PR body — see below>"
  # --draft is appended to this command only if you passed --draft
  # as the script's second argument
```

The real script generates the PR body below. Several fields are **not**
dynamic per-release content — reproduce these limitations faithfully,
or improve on them in your own copy:

```markdown
## Summary
Updates <Your Addon Name> to <version> in the community addon index.

## Why is it needed?
This release updates the addon catalog entry for version <version> with the verified release asset checksum.

## Release package
- Source repository: https://github.com/<you>/<addon-repo>
- Release tag: <version>
- Package asset: <pinned release asset URL, from addon.json's downloadUrl>
- SHA-256: <sha256, from addon.json>

## Test output
<the last 5 lines of this run's verify-release-asset-checksum.sh output — nothing else>

## Security output
<the first 10 lines of .security-reports/secret-keyword-review.txt, only if that file exists — otherwise empty>

## Permissions requested
<the addon.json permissions object, verbatim>

## Review notes
- No write permissions requested.
- No direct localhost/browser API calls.
- Data access goes through the Console bridge.
- Release URL is pinned, not floating "latest".
- SHA-256 checksum is for the exact uploaded release asset.
```

Notes on the fields above, cross-checked against the real script:

- **"Why is it needed?"** is always the fixed boilerplate sentence
  shown — the script never generates real per-release changelog
  content here. Edit it by hand after the script runs if you want the
  PR to actually explain what changed.
- **"Release tag"** uses the bare version you passed as the CLI
  argument (no `v` prefix) — this does not match the real git tag name
  (`v<version>`) and is a known inconsistency in the reference script.
- **"Test output"** is not general test-suite output; it is
  specifically `verify-release-asset-checksum.sh`'s last 5 lines.
- **"Security output"** is not an aggregation of pre-commit/gitleaks/
  semgrep/trivy results; it only reads a `.security-reports/
  secret-keyword-review.txt` file if one happens to exist at that path.
- This PR body structure is not required by `docs/addon-submission.md`
  in this repository (which has no PR-template requirement of its own)
  — it is simply what the reference script happens to produce.

### Reproducing this for your own addon

Copy `scripts/create-upstream-addon-pr.sh`, then change only:

- `CATALOG_REPO="${HOME}/dune-docker-addons"` → your local clone path of
  your fork of this repository
- `UPSTREAM="Red-Blink/dune-docker-addons"` → stays the same if
  submitting here
- every `dune-ops-observability` string → your own addon's `id`
- the PR body's addon name and description → read live from your own
  `addon.json`, never hardcoded (a hardcoded description silently
  drifts from reality the first time you edit your real `addon.json`
  without also editing the script — this happened in the reference
  implementation and was a real, found bug)

Two path pitfalls confirmed from the reference implementation's own
history, worth checking in your copy:

- If your script lives at `<repo-root>/scripts/`, the repo root is
  `$SCRIPT_DIR/..` (one level up), not two — a one-directory-too-far bug
  here silently fails Gate 0 looking for `addon.json` in the wrong
  place.
- Give your catalog fork clone a stable, permanent path outside any
  directory you might later clean up as scratch space — losing the
  clone this way happened once in the reference implementation.

### Tracking PR status after submission

`ops-observability/dev-tools/check-upstream-prs.sh` lists open/merged
status for every PR the reference maintainer has open across their
addon, Core fork, and catalog-fork repositories, using `gh pr list
--author <you> --state open`. It's a convenience wrapper, not part of
the submission pipeline itself — use `gh pr list --repo
Red-Blink/dune-docker-addons --author <your-github-username>` directly
if you don't want to reproduce its multi-repo tracking/Discord-
notification logic.

## 9. End-to-end command sequence

Run in order, from your addon repository's root, after you've bumped
`addon.json`'s `version` and are ready to cut a release. In the real
`ci.yml`, `check-sri-integrity.js` and `check-bridge-action-drift.js`
run unconditionally on every PR/push — include them as shown below if
you reproduce this pipeline as-is; omit those two lines only if your
addon genuinely has no SRI hashes to check or no bridge/API-action
table to keep in sync (see §3 for what each one actually checks):

```bash
# One-time per machine: install/verify the toolchain
bash ops-observability/dev-tools/toolchain-bootstrap.sh

# 1. Validate the manifest and all shipped code
node scripts/validate.js
node scripts/check-version-consistency.js
node scripts/check-sri-integrity.js
node scripts/check-bridge-action-drift.js
npm test

# 2. Run the full local security gate (pre-commit + gitleaks + semgrep + trivy)
bash ops-observability/dev-tools/security-shift-left.sh

# 3. Build the package
bash scripts/package.sh

# 4. Run the release-validation gate
bash scripts/validate-release.sh <version>          # bare version, e.g. 0.5.1

# 5. Tag and push — this triggers release.yml
git tag -a v<version> -m "<Your Addon> v<version> — <summary>"
git push origin v<version>

# 6. Wait for the release workflow, then verify the uploaded asset for real
gh run watch $(gh run list --workflow=release.yml --limit 1 --json databaseId -q '.[0].databaseId')
bash scripts/verify-release-asset-checksum.sh <version>

# 7. Record the verified checksum in addon.json, commit it
git add addon.json
git commit -m "chore: record verified release checksum for v<version>"
git push origin main

# 8. Submit the catalog PR to this repository
bash scripts/create-upstream-addon-pr.sh <version>
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `release.yml` refuses to run | Tagged commit isn't an ancestor of `main` | Re-tag from a commit that's actually on `main` |
| `check-sri-integrity.js` fails | You edited a script/stylesheet without regenerating its declared hash | `node scripts/update-sri.js`, then re-run the check |
| `check-version-consistency.js` fails | `addon.json`, `package.json`, and the shipped HTML disagree on version | Update whichever file(s) you missed during the version bump |
| `check-bridge-action-drift.js` fails | Your docs list an action your code doesn't call, or vice versa | Update whichever side is stale — this check has no false positives, both sides must genuinely match |
| `create-upstream-addon-pr.sh` fails at Gate 0 | `addon.json` version doesn't match the argument, or you passed a `v`-prefixed version where a bare one is expected | Pass the bare version (`0.5.1`, not `v0.5.1`) matching `addon.json` exactly |
| `create-upstream-addon-pr.sh`'s commit step needs `--no-verify` | This catalog repository has no `.pre-commit-config.yaml` of its own | Expected — your addon repo's real checks already ran in Gate 1 |
| `verify-release-asset-checksum.sh` reports a mismatch | You compared against a local zip instead of the real uploaded asset | Always run it against the real download URL; never trust `sha256sum dist/*.zip` alone as your final checksum |

## Related documentation

- [Addon Submission](addon-submission.md) — what this catalog requires
  and reviews for
- [Addon Manifest](addon-manifest.md) — the exact manifest fields this
  catalog validates against, and the separate lifecycle metadata that
  lives in `index.json`
