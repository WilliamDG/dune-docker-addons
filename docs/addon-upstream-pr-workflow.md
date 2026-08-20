# Addon Release and Catalog PR Workflow

This guide shows one reusable way to validate, scan, package, release,
and submit a Dune Docker addon. It is optional automation, not an
additional catalog requirement. The authoritative requirements remain
[Addon Submission](addon-submission.md) and
[Addon Manifest](addon-manifest.md).

The workflow is informed by
[`yacketrj/dune-ops-observability-addon`](https://github.com/yacketrj/dune-ops-observability-addon)
at commit `2227b67`, but it intentionally does not reproduce that
repository line for line. Its layout, bridge-action checks, evidence
folders, local paths, and PR tracker are specific to that addon.

## Recommended flow

Use this order for every release:

1. Validate the addon manifest and runtime files.
2. Run tests and security checks.
3. Build the install archive once.
4. Publish that exact archive as a versioned release asset.
5. Download the published asset and calculate its SHA-256.
6. Put the pinned asset URL and verified SHA-256 in the catalog
   manifest.
7. Update `index.json`, validate the catalog changes, and open a PR.

Do not rebuild or replace the archive after recording its checksum. Any
change to the archive creates a different release artifact and requires
a new checksum.

## Suggested repository layout

Adapt names to your addon, but keep release tooling separate from
runtime files:

```text
your-addon/
├── .github/workflows/
│   ├── ci.yml
│   └── release.yml
├── addon.json
├── scripts/
│   ├── validate.js
│   ├── package.sh
│   └── verify-release-asset-checksum.sh
├── test/
└── web/
```

The install archive should contain only files needed by the addon, such
as `addon.json`, `web/`, and optional runtime documentation. Do not use
GitHub's automatically generated source archives as install packages.

## 1. Validate and test

At minimum, validate the fields described in
[Addon Manifest](addon-manifest.md):

- required identity and version fields;
- a stable lowercase addon ID;
- a valid runtime entry path;
- a pinned, versioned release URL for catalog submissions;
- a 64-character SHA-256 in the catalog manifest;
- the smallest permissions the addon needs; and
- the existence and syntax of every shipped runtime file.

Run addon-specific tests as well. A plain HTML/CSS/JavaScript addon can
usually use dependency-light commands such as:

```bash
node scripts/validate.js
npm test
```

Checks tied to one implementation should remain optional. For example,
an SRI drift check is useful only when the addon declares Subresource
Integrity hashes, and a bridge-action drift check is useful only when
the addon maintains a documented action inventory.

## 2. Run security checks

Choose checks that match the repository's languages and dependencies.
A typical read-only UI addon can use:

```bash
git diff --check
shellcheck --severity=warning scripts/*.sh
gitleaks detect --source .
semgrep scan --config p/default --config p/secrets --error
trivy fs --scanners vuln,misconfig,secret --severity HIGH,CRITICAL --exit-code 1 .
npm audit --audit-level=moderate
```

Only run commands that apply to the project. For example, omit
`npm audit` when there is no lockfile and omit ShellCheck when there are
no shell scripts. Scanner allowlists should be narrow, documented, and
reviewed like code.

For CI workflows:

- grant the minimum GitHub token permissions;
- pin third-party actions to full commit SHAs;
- use a full-history checkout only for checks that need history;
- fail when dependency installation fails instead of masking the error
  with `|| true`; and
- keep the required status checks small enough that maintainers can
  understand what failed.

## 3. Package once

A minimal packaging script can validate first and then create one
versioned archive:

```bash
#!/usr/bin/env bash
set -euo pipefail

addon_id="$(node -e "process.stdout.write(require('./addon.json').id)")"
version="$(node -e "process.stdout.write(require('./addon.json').version)")"
archive="dist/${addon_id}-${version}.zip"

node scripts/validate.js
rm -rf dist
mkdir -p dist
zip -r "$archive" addon.json web -x '*.DS_Store'
sha256sum "$archive" | tee "${archive}.sha256"
```

If the package includes `addon.json`, do not try to place the package's
own final SHA-256 inside that copy before building. Changing an embedded
checksum changes the archive and therefore changes the checksum again.
The catalog manifest under `addons/` is the external trust record for
the published archive and is updated after the asset exists.

## 4. Publish a pinned release asset

The release workflow should:

1. Confirm the tag matches `addon.json.version`.
2. Run validation and tests.
3. Build the archive exactly once.
4. Generate its `.sha256` sidecar.
5. Upload both files to the versioned GitHub Release.

If the workflow generates an SBOM, attach it only when generation
succeeds. Do not claim that an SBOM was published when the step was
allowed to fail or produced no file.

An ancestor check can prevent releases from unmerged commits when that
matches the repository's release policy:

```bash
git fetch origin main
git merge-base --is-ancestor "$GITHUB_SHA" origin/main
```

## 5. Verify the published asset

Verify what users will actually download, not merely the local file
left in `dist/`:

```bash
version="1.2.3"
asset="your-addon-${version}.zip"
url="https://github.com/YOUR_USER/YOUR_REPO/releases/download/v${version}/${asset}"
expected_sha="REPLACE_WITH_RELEASE_CHECKSUM"

curl --fail --location --output "$asset" "$url"
actual_sha="$(sha256sum "$asset" | awk '{print $1}')"
test "$actual_sha" = "$expected_sha"
printf 'Verified SHA-256: %s\n' "$actual_sha"
```

Obtain `expected_sha` from the release job's checksum output or its
published `.sha256` sidecar, then record the verified value in the
catalog manifest. A verification script must require the expected value
and exit non-zero when it is missing or does not match; merely
downloading the asset is not checksum verification.

The catalog values should now point to the immutable release:

```json
{
  "version": "1.2.3",
  "downloadUrl": "https://github.com/YOUR_USER/YOUR_REPO/releases/download/v1.2.3/your-addon-1.2.3.zip",
  "sha256": "REPLACE_WITH_VERIFIED_64_CHARACTER_SHA256"
}
```

## 6. Update the catalog

Work from the current upstream catalog branch, not an old fork branch:

```bash
git fetch upstream main
git switch --create "catalog-your-addon-1.2.3" upstream/main
```

Then:

1. Add or update `addons/your-addon.json`.
2. Add or update the matching short entry in `index.json`.
3. Keep lifecycle fields in `index.json`, not the addon manifest.
4. Parse every changed JSON file.
5. Run any validation script present in the catalog checkout.
6. Review the diff before committing.

Example JSON validation:

```bash
python3 -m json.tool addons/your-addon.json >/dev/null
python3 -m json.tool index.json >/dev/null
git diff --check
git diff -- addons/your-addon.json index.json
```

Commit and push to your catalog fork:

```bash
git add addons/your-addon.json index.json
git commit -m "catalog: update Your Addon to 1.2.3"
git push --set-upstream origin "catalog-your-addon-1.2.3"
```

Avoid force-pushing by default. Use it only when intentionally replacing
your own existing PR branch and after confirming the exact target.

## 7. Open the PR

The GitHub CLI command is:

```bash
gh pr create \
  --repo Red-Blink/dune-docker-addons \
  --head "YOUR_USER:catalog-your-addon-1.2.3" \
  --base main \
  --title "Your Addon 1.2.3" \
  --body-file /path/to/pr-body.md
```

The PR description should contain real output from the release being
submitted, not fixed boilerplate presented as evidence. Include:

- what changed and why;
- the source repository and release tag;
- the exact pinned package URL;
- the verified SHA-256;
- tests and security checks that actually ran;
- requested permissions and their justification; and
- any known limitation relevant to review.

## Automating the catalog PR safely

If you turn the steps above into a script, make every repository-specific
value explicit instead of copying another addon's constants:

```bash
GITHUB_USER="YOUR_USER"
ADDON_ID="your-addon"
ADDON_NAME="Your Addon"
ADDON_REPO="/absolute/path/to/your-addon"
CATALOG_REPO="/absolute/path/to/your-catalog-fork"
UPSTREAM="Red-Blink/dune-docker-addons"
```

Also audit and parameterize all of these before using a copied script:

- source repository and release asset URLs;
- release tag and archive naming conventions;
- fork owner, branch name, PR title, and PR head;
- manifest and `index.json` IDs;
- permission claims and review notes;
- local evidence and PR-tracker paths; and
- any fallback permission or metadata values.

Require a clean working tree, validate both repositories, require the
expected checksum argument, and print the exact branch and remote before
pushing. Do not use `--force` or `--no-verify` as universal defaults.

## Reference implementation caveats

The reference addon at commit `2227b67` contains useful examples, but
three parts should not be copied unchanged:

1. `validate-release.sh` compares a ZIP checksum with a checksum stored
   inside `addon.json` in that same ZIP. This is self-referential and
   cannot be satisfied by an edit-and-rebuild loop. Keep its other
   package-content checks, but remove that comparison.
2. `create-upstream-addon-pr.sh` calls its checksum script without the
   expected SHA and then reports the checksum as verified. Pass the
   expected SHA and make it mandatory.
3. The PR script contains contributor-specific usernames, repository
   URLs, local paths, addon IDs, review claims, permission fallbacks,
   and tracker updates. Parameterize or remove every one of them.

The reference `validate.js` also checks an HTML version only when its
regular expression finds one; its separate
`check-version-consistency.js` is the check that fails when the version
is absent. Preserve that distinction if adapting those scripts.

These caveats are tracked in the reference repository as issues
[#158](https://github.com/yacketrj/dune-ops-observability-addon/issues/158)
and
[#159](https://github.com/yacketrj/dune-ops-observability-addon/issues/159).
