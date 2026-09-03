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
</content>
