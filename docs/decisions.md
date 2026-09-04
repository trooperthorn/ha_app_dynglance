# Decisions

Dated design decisions pulled out of source comments during the 2026-09-03
comment-to-docs pass, each with the alternative that was rejected and why.
This is a maintainer reference, not part of the published docs site under
`docs/docs/`.

## 2026-09-03: bookmark widget uses raw/processed field pairs instead of pointer dereference in templates

`internal/dynglance/widget-bookmarks.go`, the `SameTabRaw`/`SameTab` and
`HideArrowRaw`/`HideArrow` fields. A `*bool` is needed to tell "not set in
config" apart from "explicitly set to false", but Go's `text/template`
cannot dereference a pointer directly: `{{ if not .SameTab }}` would
evaluate true for any non-nil pointer regardless of what it points to,
because the pointer itself is truthy. Rejected: exposing the raw `*bool`
field to templates and asking template authors to work around the
dereference problem themselves. Chosen instead: parse the raw pointer field
once in Go and duplicate it into a plain `bool` field that templates read
directly, at the cost of one extra field per optional boolean.

## 2026-09-03: custom-api template functions take `(prefix, s)` rather than `(s, prefix)`

`internal/dynglance/widget-custom-api.go`, `trimPrefix` and `trimSuffix`.
Go template pipelines pass the piped value as the *last* argument to the
next function, not the first. Rejected: the more natural argument order
`trimPrefix(s, prefix)`, matching `strings.TrimPrefix`. Chosen instead: the
reversed order `trimPrefix(prefix, s)`, so a template author can chain
calls like `{{ .JSON.String "foo" | trimPrefix "bar" | doSomethingElse }}`
instead of having to nest calls as
`{{ trimPrefix (.JSON.String "foo") "bar" | doSomethingElse }}`.

## 2026-09-03: ZFS disk usage shells out to `zfs list` instead of using statfs

`pkg/sysinfo/sysinfo.go`, `getZFSUsage`. Rejected: relying on `statfs(2)`
for all filesystems uniformly, as is done for non-ZFS mountpoints. That
call returns `Used=0` for a ZFS pool root when all the pool's data lives in
child datasets, which is the normal layout on TrueNAS SCALE and similar
systems, making the sensor read as empty. Chosen instead: shell out to
`zfs list -H -p -o used,available,mountpoint` for ZFS mountpoints and fall
back to the `statfs` values only if the `zfs` binary is unavailable or the
mountpoint is not found.

## 2026-09-03: container disk reporting falls back to `/` instead of requiring a host bind-mount

`pkg/sysinfo/sysinfo.go`, filesystem enumeration. Inside a container,
`disk.Partitions(false)` only returns devices whose source starts with
`/dev/`, which excludes the overlay root filesystem and leaves the widget
with no mountpoints to report. Rejected: requiring the operator to bind-mount
a host path so a `/dev/`-prefixed device shows up. Chosen instead: when no
mountpoints are found, add `/` explicitly, so disk usage is still reported
out of the box without requiring extra container configuration.

## 2026-09-03: Ingress base URL is read from a request header, not only from static config

`internal/dynglance/dynglance.go`, `effectiveBaseURL` and the
`X-Ingress-Path` header handling. Each Home Assistant installation's
Ingress path prefix (`/api/hassio_ingress/<token>`) is random and assigned
at runtime, so it cannot be known when `dynglance.yml` is written. Rejected:
requiring the operator to set a fixed `server.base-url` for Ingress
deployments, which cannot represent a value that changes per installation
and is not known until Supervisor proxies the request. Chosen instead: read
the Supervisor-supplied `X-Ingress-Path` header on each request and prefer
it over the static `server.base-url` when present, falling back to the
static value for manually reverse-proxied setups. Full user-facing detail
is in `docs/docs/home-assistant.md`.

## 2026-09-04: `effectiveBaseURL` validates the `X-Ingress-Path` header instead of trusting it unconditionally

`internal/dynglance/dynglance.go`, `effectiveBaseURL`. Found by triaging
`gosec` G710 (open redirect) findings added by the security workflow: every
login/logout/OIDC-callback redirect is built as
`effectiveBaseURL(r) + "/some/path"`, and the header value was used as-is
with no validation. Inside Home Assistant's Ingress proxy this header is set
safely by Supervisor, but the app also accepts direct port access
(documented in `docs/docs/home-assistant.md`) and ships as a standalone
Docker image with no Supervisor in front of it at all; in either of those
modes a client can set `X-Ingress-Path` to anything, including an absolute
URL like `https://evil.example.com`, turning every one of those redirects
into an open redirect to an attacker's origin. Rejected: suppressing the
`gosec` findings with `#nosec`, since the taint warning was correct and the
header genuinely is attacker-controlled outside the Ingress-proxied
deployment mode. Rejected: writing a new validator, since `auth.go` already
has `isSafeLocalPath` (same-origin-path check, with its own test coverage in
`auth_redirect_test.go`) guarding the post-login redirect-target cookie for
exactly this class of bypass, backslash variants included. Chosen instead:
reuse `isSafeLocalPath` for the Ingress header too, so a forged header
degrades to the configured static `server.base-url` instead of being
followed, and the two same-origin checks in the codebase can't drift apart.

## 2026-09-04: Go toolchain bumped 1.25.14 to 1.26.8, a minor version, not held on 1.25.x

`go.mod`, `Dockerfile`, and `ha-addon/dynglance/Dockerfile`. PR #7's image
scan flagged a stdlib CVE (CVE-2026-46600) fixed only in Go 1.26.6 or
later; no 1.25.x patch carries the fix. Rejected: staying on the latest
1.25.x patch (1.25.14, itself a same-day earlier fix for a different batch
of stdlib CVEs) and treating this one CVE as accepted residual risk, since
a minor version bump carries more compatibility risk than a patch bump.
Checked the Go 1.26 release notes for anything that could break this
codebase before deciding: no language or standard library changes affect
how this app uses `net/http`, `html/template`, `text/template`,
`encoding/json`, `encoding/xml`, or `context`; the only two notices that
touch packages this app actually uses are `http.Client` now scoping
cookies to `Request.Host` (affects `widget-torrenting.go`'s cookie jar
only if a request's `Host` header differs from its connection target,
which this app never sets) and `http.ServeMux` trailing-slash redirects
changing from 301 to 307 (more correct, not a behavior this app or a
browser depends on). Go's own release notes state the Go 1 compatibility
promise holds for 1.26. Chosen: bump to 1.26.8 (the latest patch as of
this session) now, rather than defer past the point of forgetting why the
CVE was accepted.

## 2026-09-04: standalone-image Dockerfiles run `apk upgrade` before installing packages

`Dockerfile`, `Dockerfile.goreleaser`. Bumping the Alpine base from 3.21 to
3.24 (previous decision) fixed every `curl` CVE the image scan had found,
but the very next run surfaced a fresh batch of `libssl3`/`libcrypto3`
CVEs instead, with a patched package version already available upstream.
This is Alpine's ordinary rolling release cycle, not a regression: a base
image tag is a snapshot of whatever packages were current when it was last
published, and new CVEs get disclosed against those snapshotted versions
continuously. Rejected: chasing each new disclosure with another version
bump, which only ever fixes the packages that happen to be behind as of
the last time someone looked. Chosen instead: `apk upgrade --no-cache`
before `apk add`, so every build pulls whatever Alpine's repo currently
has for every package already in the base image, not just the ones this
Dockerfile explicitly installs. This does not fully solve image staleness
(a build today is still frozen at today's packages until the next build),
but it removes the specific failure mode of the base layer alone being
stale relative to the image tag.

Deliberately not applied to `ha-addon/dynglance/Dockerfile`'s runtime
stage: it builds from `${BUILD_FROM}` (`ghcr.io/home-assistant/base` by
default), a Home Assistant-curated and version-pinned base image for
Supervisor compatibility, not raw Alpine. Running `apk upgrade` there
could pull packages from a different repository branch than what HA
built and tested that base image against, and that compatibility can't be
verified from outside a real Home Assistant environment.
</content>

## 2026-09-04: releases come from a merge to `main`, not a hand-pushed tag

`Release` publishes the version already in `ha-addon/dynglance/config.yaml` and calls
`deploy.yml` with it; `Prepare release` writes the next version into both version fields
in a reviewed, auto-merged PR. Rejected: the previous flow, where the maintainer pushed a
`YYYY.MM.DD.V` tag by hand and the app version was bumped separately, which
`operations.md` itself described as "nothing automates that today". Tags stay bare
because the Supervisor shows the app version verbatim and the deploy job tags images
with it.

## 2026-09-04: upstream code comments stay

The comment-to-docs pass covered the workflows and `docs/`. Comments inside the Go
sources and the static JavaScript, which this fork shares with `Panonim/dynacat`, are left
as written so upstream merges stay clean.

