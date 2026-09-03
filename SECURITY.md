# Security Policy

## Reporting a vulnerability

Do not open a public issue containing exploit details, credentials, private
addresses, or logs. Use GitHub's private vulnerability-reporting feature for
this repository. If private reporting is unavailable, open a minimal issue
asking the maintainer to establish a private channel; omit technical details.

Include the affected version/commit, prerequisites, impact, a minimal
reproduction, and suggested remediation. Remove tokens, API keys, cookies,
and private network details.

Vulnerabilities in DynGlance's upstream (github.com/glanceapp/glance) that
are not specific to this fork's changes should also be reported upstream,
since this fork does not control that codebase's release cadence.

## Response targets

These are project targets, not an SLA: acknowledge critical/high reports in
three business days, establish severity and containment in seven, and publish
a coordinated fix/advisory as soon as safely validated. Lower-severity issues
are prioritized by exploitability and impact.

## Supported version

Only the latest published release and the default branch receive security
fixes. Operators should keep Home Assistant and the DynGlance app updated
and retain a tested rollback.

## Security boundaries

DynGlance is a self-hosted dashboard, not a sandbox. Running it as a Home
Assistant app grants it, by default, read-only access to the Home Assistant
config directory (for `$include` references) and read-only access to the
share directory; it has no write access to Home Assistant's own config.
Enabling `docker_api` for the docker-containers or docker-controller widgets
grants the container access to the Docker socket, which is equivalent to
root on the host, and unconditionally forces the app's Home Assistant
security rating to 1 (see `docs/docs/home-assistant.md`); that option is
off by default and its consequence is not cosmetic.

The optional Home Assistant long-lived access token (`home_assistant_token`)
is passed to the container as an environment variable and is only readable
server-side by `custom-api` widgets; it is never sent to the browser. The
Config Upload feature, when enabled, gates config replacement behind a
passphrase (minimum 12 characters) rather than requiring no authentication;
it is off by default. `custom-api` and `dynawidgets` widgets make outbound
HTTP requests to whatever URL is configured, including internal network
addresses if DynGlance itself can reach them; there is no built-in
allowlist, so anyone who can edit `dynglance.yml` can direct the app's
network access.

These findings and controls are evidence for an operator's own risk
assessment; they are not a certification or a claim that no vulnerability
exists.
