# Troubleshooting

## `docker compose up` fails: address already in use (port 443 or 80)

Something else on your machine — a VPN client, another web server, a tool
like Tailscale — is already listening on that port. Docker's error
message doesn't say what, only that it's taken. Find out with:

```sh
lsof -iTCP:443 -sTCP:LISTEN      # macOS / Linux
# or
netstat -anv -p tcp | grep LISTEN | grep 443
```

Then either stop that process, or use a different port for VPM. Set
`HOST_HTTPS_PORT` (and `HOST_HTTP_PORT` if 80 is also taken) in your
`.env` to a free port, e.g. `HOST_HTTPS_PORT=8443`, and browse to
`https://localhost:8443` instead. There is nothing VPM- or
Caddy-specific about 443; any free port works.

## Certificate warning on localhost

Expected, and only for the local `tls internal` variant of the Caddyfile.
Caddy is minting its own private certificate authority just for your
machine, which no browser trusts by default — that's a feature (no
internet-facing ACME challenge needed to try VPM locally), not a bug.
Click through the warning to continue. If you're testing with `curl`, use
`curl -k`.

This warning should **not** appear once you switch to the real-domain
variant in the Caddyfile with a real DNS name — if it still does after
that switch, something is still using the local block; check which site
block is active in your Caddyfile.

## `Invalid request.` on first login

This happens if you load the login page (which hands your browser a CSRF
token) *before* running
`vpm issue-admin-temporary-password`. That command deletes every existing
session, including the one that handed out the token you're about to
submit. Reload `https://localhost/login` fresh, after minting the
temporary password, and submit the form from that fresh load. See
[first-run.md](first-run.md#1-create-your-admin-login).

## `account_identity_mismatch` (403) when adding your Vast API key

`VPM_EXPECTED_ACCOUNT_ID` is unset, or set to the wrong account. This
error fires regardless of whether the key itself is valid — VPM checks
account identity before it looks at anything else. See
[first-run.md](first-run.md#2-find-your-vast-account-id), set the
variable, and restart the stack.

## Host header / `TrustedHost` errors, or the healthcheck never turns healthy

VPM only accepts requests whose `Host` header exactly matches
`VPM_ALLOWED_HOSTS` — including its own internal healthcheck, which uses
the *first* entry in that list. If you change the hostname you browse to
(for example, moving from `localhost` to a real domain), you must update
**both**:

- `VPM_ALLOWED_HOSTS` in `.env` (or directly in `compose.yml`), and
- the site block in `Caddyfile`

to the same exact hostname, then restart. A mismatch between these two is
the most common cause of a container that starts but never reports
healthy, and of `400` responses that otherwise look inexplicable.

## Docker Desktop vs. a bare Linux host

This recipe's core trick — Caddy attached to VPM's network namespace via
`network_mode: "service:vpm"`, and the master key delivered as a
file-based Compose secret — are both plain Docker Engine features, not
Docker Desktop extensions, so they work the same way on Linux. Two things
that can differ across platforms:

- **File ownership through Docker Desktop's file-sharing layer.** On
  Docker Desktop for Mac/Windows, host-mounted files sometimes need an
  extra beat to settle into the ownership the container expects. If the
  container reports it can't read `master.key`, restart the stack once
  before assuming something is actually wrong.
- **`--network host`.** Don't reach for it as a shortcut on Linux instead
  of the Caddy sidecar pattern above — it isn't available at all on
  Docker Desktop, and VPM itself refuses to bind anywhere except loopback
  in standalone mode regardless of platform, so `-p` publishing straight
  onto VPM's own port doesn't work either way. Use `compose.yml` as
  shipped.

## VPM is up but `/readyz` keeps saying `read_sync_stale`

Normal until your first successful sync. Run:

```sh
docker compose exec vpm vpm sync
```

If it keeps happening after that, check that your Vast API key was
accepted (Credentials page shows no error) and that
`VPM_EXPECTED_ACCOUNT_ID` matches the account the key belongs to.
