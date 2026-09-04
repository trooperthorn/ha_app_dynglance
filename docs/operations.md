# Operations

This file documents operational knowledge for maintaining this fork (release
process, CI) that doesn't belong in the user-facing DynGlance documentation
under `docs/docs/`.

## Release path

A merge to `main` is the only release path. Nobody edits the version fields or pushes a
tag by hand.

1. `Release` runs on every push to `main` that touches anything outside `docs/`. It calls
   the Test workflow, then reads the version from `ha-addon/dynglance/config.yaml`
   through `.release.json` (the app Dockerfile's `APP_VERSION` default must match). If a
   published release for that version exists it stops. Otherwise it creates the tag,
   drafts the release, calls `deploy.yml` with that version to build and push the
   multi-arch images to ghcr.io and Docker Hub and to attach the Linux binaries, then
   publishes the release as latest.
2. `Prepare release` runs after every successful `Release` on `main`. When the version
   equals the latest published release and any release-bearing path changed since that
   tag (`ha-addon`, `internal`, `pkg`, `main.go`, `go.mod`, `go.sum`, the root
   `Dockerfile`), it runs `scripts/set_version.py --next-from-tags`, which rewrites both
   version fields, pushes the bump to `automation/calver-release` with a GitHub App token,
   opens a PR, and arms squash auto-merge. The merge triggers `Release` again. Changes
   under `docs/` and to workflows do not bump the version.
3. Without the GitHub App credentials the second step fails at its credential check and
   nothing else happens. The repository still releases: run
   `python scripts/set_version.py --next-from-tags` on a branch, open the PR, and the
   merge publishes.

Versions and tags are the bare `YYYY.MM.DD.N` (no `v` prefix), matching the app's
`version:` field, which the Supervisor shows verbatim, and the `type=raw` image tag the
deploy job writes. Older tags with other shapes (`2.4.0` from upstream, `v2026.08.22.0`)
stay but are ignored when the next sequence is counted. `scripts/set_version.py` is the
only writer; `scripts/release_config.py` behind `build_release_artifacts.py
--validate-only` is the independent reader the Test workflow also runs.

`deploy.yml` no longer triggers on tag pushes; it is a callable workflow invoked by
`Release` with the version as input, plus `workflow_dispatch` for a manual image build
(which then tags images with today's date and `.0` and attaches nothing).

### GitHub App for zero-touch version PRs

`Prepare release` needs the release GitHub App (Contents: Read and write, Pull requests:
Read and write) installed on this repository, plus the repository variable
`RELEASE_AUTOMATION_CLIENT_ID` and the Actions secret `RELEASE_AUTOMATION_PRIVATE_KEY`.

## Image scan scope

`security.yml` scans the image built from the root `Dockerfile`, which builds from the
local checkout. The app Dockerfile under `ha-addon/dynglance` clones `DYNGLANCE_REF`
(`main`) from GitHub by design, so the Supervisor can build it locally; scanning that one
in CI would scan `main` rather than the change under review.

## Build-time version injection

`internal/dynglance/main.go`'s `buildVersion` variable is set at build time
via an `-X` ldflag (see `.goreleaser.yaml`, `Dockerfile`,
`ha-addon/dynglance/Dockerfile`, and `.github/workflows/deploy.yml`), using
the same `YYYY.MM.DD.V` scheme as the release tags above. A plain `go
build`/`go run` with no ldflags leaves `buildVersion` empty; `resolveVersion`
falls back to today's date with a `.0` build number in that case, so the
displayed version is always a real date, never a placeholder like "dev" or
"unknown".
