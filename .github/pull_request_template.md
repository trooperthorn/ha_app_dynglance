## Change

Describe the behavior and security impact.

## Verification

- [ ] `go build`/`go vet`/`go test` pass (test.yml)
- [ ] Security workflow passes (CodeQL, gosec, ShellCheck, image scan)
- [ ] No secret, credential, token, private key, or production log was committed
- [ ] New network/outbound-request behavior is documented and failure-bounded
- [ ] `docs/docs/` (user-facing) or `docs/operations.md`/`docs/design.md`/`docs/decisions.md` (maintainer) updated when applicable

## Risk and rollback

State affected trust boundaries, migration steps, and a rollback method.
