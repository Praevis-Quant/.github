# Standard release pipeline: `trusted-publishing.yml`

Every Praevis-Quant repository that publishes packages uses the same release
pipeline: same file, workflow name, job ids, job display names, order and
permissions. A reader of any repository's Actions tab sees the same jobs. The
owner approved this standard for Praevis-Quant on 2026-11-22. It is the
[Rust Commons standard](https://github.com/rustcommons/.github/blob/main/docs/TRUSTED-PUBLISHING.md)
(approved there on 2026-11-21), with the Praevis-Quant additions marked below.

crates.io binds Trusted Publishing to the repository, the workflow **file name**
`.github/workflows/trusted-publishing.yml` and the environment `crates-io`;
PyPI binds it to the same file and the environment `pypi`. Those identities
never change. Job ids and names are not part of either registration, so
adopting this standard needs no crates.io or PyPI change.

## The pipeline

Workflow `name: Trusted publishing`.

| # | Job id | Display name | Runs on | Purpose | Extra permissions |
|---|---|---|---|---|---|
| 1 | `gate` | Release gate | every run | Decide whether this run may publish. | `actions: read` |
| 2 | `build` | Build | every run | Package, test and checksum exactly what will be published. | none |
| 3 | `attest` | Checksums and attestations | tag runs | Sign provenance for the checksummed files. | `id-token: write`, `attestations: write` |
| 4 | `publish` | Publish to crates.io | tag runs, after owner approval | Publish through Trusted Publishing. | `id-token: write` (environment `crates-io`) |
| 4b | `publish-pypi` | Publish to PyPI | tag runs, after `publish` and a second owner approval | **Praevis-Quant addition:** publish the wheels through PyPI Trusted Publishing. | `id-token: write` (environment `pypi`) |
| 5 | `github-release` | GitHub release | tag runs | Create the release from committed notes and attach the files. | `contents: write` |
| 6 | `verify` | Verify publication | tag runs | Prove that what the registries and the release serve is what was built and signed. | none |

`needs`: `build` needs `gate`; `attest` needs `gate` and `build`; `publish`
needs `gate`, `build` and `attest`; `publish-pypi` needs `gate`, `build` and
`publish`; `github-release` needs `gate` and `publish` (and `publish-pypi`
where it exists); `verify` needs `gate` and `github-release`. Jobs 3 to 6 run
only when `needs.gate.outputs.publish == 'true'`; each lists `gate` directly
because a job can read outputs only from the jobs it names in `needs`.

A repository with no Python package has no `publish-pypi` job and no `pypi`
environment.

### 1. Release gate

On a `v*` tag, all of these must hold, or the run fails before anything is built:

- The tag is annotated, and its name is `v` plus the release version.
- The tagged commit is reachable from `main`, and CI succeeded on that commit.
- Every crate whose version equals the release version is in the release set;
  every other workspace crate is already on crates.io at its declared version.
- **`CHANGELOG.md` has a dated `## [X.Y.Z] - YYYY-MM-DD` entry** for the release
  version, and `docs/releases/vX.Y.Z.md` exists. The changelog entry is the
  record the release notes are written from; a release without it is refused.

On pull requests and manual runs the gate reports what it found, sets
`publish=false` and lets the dry run continue. Outputs: `version`, `publish`
and the ordered crate list.

### 2. Build

- `cargo package --locked` for each release crate in dependency order; this also
  builds each crate from its packaged source.
- A CycloneDX SBOM from `cargo-cyclonedx`, installed with `--locked` at a pinned
  version.
- `SHA256SUMS` over the `.crate` files and the SBOM.
- Uploads one artifact, `release-<version>`, holding all of the above.

Repositories with extra release qualification (package audits, candidate
installs) run it here, so a pull request dry run exercises it too.

### 3. Checksums and attestations

Re-checks `SHA256SUMS`, then signs GitHub artifact attestations (Sigstore, SLSA
build provenance) for every `.crate` file and the SBOM. Saves the signed bundle
as `<repo>-<version>.crates.intoto.jsonl` in the release artifact.

### 4. Publish to crates.io

The only job with the `crates-io` environment: the owner's approval click.

- Re-checks the downloaded files against `SHA256SUMS`.
- **Nothing compiles in this job.** It holds `id-token: write`, so any code it
  runs could request its own OIDC token with the claims Trusted Publishing
  trusts. It runs `cargo package --locked --no-verify` for each release crate
  and `cmp`s the result against the attested `.crate`; `build` already built
  the packaged source.
- Only then obtains a short-lived token from the pinned
  `rust-lang/crates-io-auth-action`, and runs `cargo publish --locked
  --no-verify` for each release crate in dependency order, with
  `CARGO_REGISTRY_TOKEN` set on that step alone. `build` has already proved
  each package builds. It skips a version crates.io already has,
  so a re-run after a partial failure is safe.
- **Praevis-Quant addition:** after each crate is published, the job downloads
  it back from `static.crates.io` and requires its SHA-256, and the sparse
  index's checksum, to equal the attested `.crate`, retrying network errors
  with bounded backoff. Any difference fails the job, so nothing after it
  (PyPI, the GitHub release) runs. The toolchain is the exact one that built
  the attested crates (a `build` output), not the current stable, because the
  approval can come after a Rust release.

### 4b. Publish to PyPI (Praevis-Quant addition)

Only in repositories that ship a Python package. The only job with the `pypi`
environment: a **second, separate approval click**, which GitHub requests only
after `publish` succeeded, including its byte-for-byte checks. A crate problem
therefore stops the release before anything reaches PyPI.

- Re-checks the downloaded files against `SHA256SUMS`, and uploads exactly the
  attested wheels with the pinned `pypa/gh-action-pypi-publish` (PyPI
  attestations on, existing files skipped on a re-run). No sdist unless the
  repository ships one deliberately.
- Nothing compiles in this job either.

### 5. GitHub release

Creates release `vX.Y.Z` with `--verify-tag` from the committed
`docs/releases/vX.Y.Z.md`, so the published body equals the reviewed file. The
title is that file's first heading. Attaches the `.crate` files, the SBOM,
`SHA256SUMS` and the provenance bundle. On a re-run it uploads only missing
files and never replaces an existing asset.

### 6. Verify publication

- crates.io's checksum for each published crate equals the attested `.crate`
  file's SHA-256.
- `gh attestation verify` succeeds for every attached file against this
  repository and this workflow.
- docs.rs reports a successful build. A queue that is still pending after the
  bounded wait is reported as pending, not as a pass.
- Crates that ship executables pass a clean `cargo install --locked` into an
  empty root and report the release version.

**Praevis-Quant addition (verify scope):** `verify` runs as a reusable
`verify-publication.yml` with one leg per release target, each on that
target's native runner, and can also be run by hand for any published version
(and on pull requests that change it, against the latest release). Every leg:
- checks that the release has exactly the expected assets, each matching
  GitHub's recorded digest and `SHA256SUMS`;
- verifies every artifact's provenance online and offline (the attached bundle
  and Sigstore's trusted root), and the SBOM attestations, all with
  `--signer-workflow` set to the build workflow and
  `--source-ref refs/tags/v<version>`;
- runs that platform's downloaded executable, with no token in its
  environment.

One leg also checks:
- the release body against the committed notes;
- crates.io (API, sparse index and download) and PyPI against the attested
  files;
- docs.rs;
- a clean `cargo install --locked` and `pip install`, retried while the index
  lags.

Every leg uploads its report and every attestation verification as a proof
artifact, and the publication record cites it.

## Common settings

- Triggers: `push: tags: ['v*']` publishes. `pull_request` (filtered to the
  workflow and release scripts) and `workflow_dispatch` are dry runs that run
  jobs 1 and 2 and never publish.
- **Praevis-Quant addition (owner decision 2026-11-22):** dry runs also run
  `publish-path-check` ("Publish path (dry run)"), with `needs: [gate,
  build]` and `if: needs.gate.outputs.publish != 'true'`, `contents: read`
  only, no environment and no OIDC token. It exercises the publish job's
  checks without publishing: the build's pinned toolchain, the bundle
  against `SHA256SUMS`, `cargo package --no-verify` compared byte for byte
  with the checksummed `.crate` files, and the crates.io download checks
  against a version already published. So the publish path is tested before
  the first tag.
- Top-level `permissions: contents: read`; every extra permission is granted
  only in the job listed above.
- `concurrency: group: trusted-publishing-${{ github.ref }}`, with
  `cancel-in-progress` true only for pull requests. A tag run is never cancelled.
- Runners `ubuntu-24.04`. Actions pinned to full commit SHAs with version
  comments; checkout with `persist-credentials: false`. Tools installed with
  `--locked` at exact versions. No `[skip ci]`.

### GitHub settings (Praevis-Quant)

Each publishing repository has these settings; the workflow cannot enforce
them, so the repository records them as read back from GitHub.

| Setting | Configuration |
|---|---|
| Environment `crates-io` (and `pypi` where it exists) | Required reviewer: `joey-huckabee`. Deployments only from tags matching `v*`. Administrators cannot bypass the protection rules (`can_admins_bypass: false`). |
| Tag ruleset "release tags immutable" | On `refs/tags/v*`: blocks deletion, update and non-fast-forward, with **no bypass actors**, so it applies to administrators too. |
| Tag ruleset "release tags creation" | On `refs/tags/v*`: blocks creation, with one bypass, the repository admin role. Only administrators can create a `v*` tag. |
| Branch ruleset on `main` | Pull request required, the repository's required status checks, linear history, no deletion or force-push, no bypass actors. |

**Who acts, stated honestly.** The development agents work under the
maintainer's own GitHub account (the owner's decision; there is no bot
identity). GitHub therefore enforces that the account approves each
environment and that no one moves a release tag, but it cannot tell an
agent's actions from the maintainer's. That agents never create tags,
publish or approve rests on **process** (each repository's AGENTS.md and the
maintainer's review of each run), **not on a technical control**.

## Repositories that also ship binaries

Repositories that ship prebuilt executables or wheels use a **binary
variant** of the standard, because SLSA Build L3 needs the build and the
signature in the same isolated reusable workflow:

- `build` is `uses: ./.github/workflows/build-release.yml`. That reusable
  workflow builds every binary target and wheel with all qualification gates,
  packages the `.crate` files, generates the SBOMs and one `SHA256SUMS`, and
  then, only on a `v*` tag push, re-verifies and signs its **own** output in
  its nested job "Checksums and attestations". The caller cannot choose what
  is signed.
- There is no top-level `attest` job. The caller's `build` job grants
  `id-token: write` and `attestations: write`, which the nested signer alone
  uses; PR and manual dry runs build everything but never sign.
- Building and signing finish **before** the owner's `crates-io` approval, so
  approval timing does not matter.
- `github-release` creates the release as a draft, checks its tag, title, body
  and state against the committed notes, uploads every asset, checks each by
  name, size and digest, then publishes and checks again. A re-run resumes a
  draft and never modifies a published release.
- Assets: the `.crate` files, every binary archive and wheel, the SBOMs, one
  `SHA256SUMS` and one `<repo>-<version>.intoto.jsonl` bundle covering every
  signed file (the suffix OpenSSF Scorecard counts as provenance), plus any
  SBOM attestation bundles.
- **Praevis-Quant addition:** the `.crate` files are packaged once in the
  build and again on a second runner and architecture with the same
  toolchain, and must be byte-identical before anything is signed.
- Verifiers keep accepting earlier releases' layouts exactly, and require the new
  layout only for later versions.

In Praevis-Quant, praevis-thetadata uses the binary variant: CLI executables
for eight targets and abi3 wheels for eight platforms, SLSA Build Level 3.

## Lessons from the rollout

From Rust Commons:
- **Gate CI lookup:** GitHub's `status=success` filter on workflow runs can
  return no runs for a commit whose CI succeeded. Read every run for the
  commit (`?head_sha=SHA&per_page=100`) and require one with
  `conclusion == 'success'` (cargo-libc#44).
- **Fuzz workspace locks:** a version bump must also update `fuzz/Cargo.lock`
  (and any exact `=X.Y.Z` pin in `fuzz/Cargo.toml`), or the locked fuzz build
  fails. The same holds for any other lockfile in the repository that names
  the workspace's crates by path.
- **Binary repositories:** signing must stay in the reusable workflow that
  built the files; a signer that signs a caller-supplied artifact no longer
  proves the qualified build made it (rust-mirror#76 review). A killed
  `gh release create` leaves a draft that the tags API cannot see, so create
  drafts deliberately and resume them through the releases list.
- **Smoke tests:** don't pipe a program's output into `head` under `pipefail`;
  write it to a file first.

From Praevis-Quant (praevis-thetadata, 0.1.3 to 0.1.5):
- **`cargo package` is reproducible:** two checkouts, runners or
  architectures with the same toolchain produce byte-identical `.crate`
  files, and the files `cargo publish` uploads one crate at a time equal the
  ones packaged together in the build. That is what makes the byte-for-byte
  checks before and after publishing possible.
- **Verify against the tag:** any run that signs from a branch produces
  genuine attestations for byte-identical `.crate` files. Every documented
  `gh attestation verify` command passes `--source-ref refs/tags/v<version>`.
- **Registry reads need retries:** a slow index or a network error after a
  crate is published must not fail the release; retry with bounded backoff.
- **GitHub stores release bodies with a trailing newline:** compare the body
  with the committed notes after trimming trailing whitespace.

## Rollout tracker

| Repository | Before | Adoption |
|---|---|---|
| praevis-thetadata | `check`, `build` (binary variant), `publish-crates`, `publish-pypi`, `github-release`, dry-run `publish-path-check`; `verify` proposed (praevis-thetadata#236) | Adopting: gap list with the owner (2026-11-22) |
| praevis-data, praevis, axiom-ledger, twsapi-mirror, oquants-com, ibkr-rs, .github | publish nothing (checked 2026-11-22: only the line-endings check; praevis-data has no workflows) | Not applicable until a package ships |

Each adoption is one pull request whose dry run passes before merge. The first
real use is the repository's next authorized release; no published version is
re-tagged or re-released. Update this table as each adoption merges.
