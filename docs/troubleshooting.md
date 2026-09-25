# Troubleshooting

## Master key not readable

VPM starts, `/healthz` says `{"ok": true}`, and everything looks fine —
until you try to add your Vast API key (or you check `/readyz`) and get
`credential_master_unavailable`. Inside the container this shows up as a
plain `PermissionError` reading `/run/secrets/vpm_master_key` if you go
looking for it.

The cause: Compose's file-based secret is a **bind mount**, not a copy, so
the container sees `master.key`'s exact host owner and permission bits.
VPM's container always runs as UID 999. If `master.key` is owned by your
own host user (the default result of just `chmod 600 master.key`), VPM's
own user cannot read it — on a native Linux Docker host this fails
immediately; on Docker Desktop, the file-sharing layer sometimes
translates ownership in a way that hides the problem, so it can work on
your laptop and fail the same host you deployed it to.

Fix it by handing the file to UID 999 directly:

```sh
sudo chown 999:999 master.key
sudo chmod 400 master.key
```

Do this once, right after you generate `master.key` (the quickstart in
[README.md](../README.md) already includes it) and again any time you
restore it from a backup (see
[upgrade-and-backup.md](upgrade-and-backup.md)) — a restored copy starts
out owned by whoever ran the restore, not by VPM.

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

## Self-test fails, or times out reaching a progress/status endpoint

If a machine self-test fails with a connection timeout, or a message
about being unable to reach the instance's progress or status endpoint,
the most likely cause is not VPM or the Vast CLI — it's that VPM is
running on the same local network as the machine you're testing. The
test has to connect to that machine's own public IP address, and most
routers can't route traffic from inside their own network back out to
their own public IP ("NAT hairpinning"). See
[docs/self-test.md](self-test.md#the-same-network-limitation-read-this-before-you-self-test)
for the full explanation. Run VPM from a different network than the
machine you're self-testing and try again.

If it keeps happening after that, check that your Vast API key was
accepted (Credentials page shows no error) and that
`VPM_EXPECTED_ACCOUNT_ID` matches the account the key belongs to.
