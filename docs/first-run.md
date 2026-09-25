# First run

This picks up after the quickstart in [README.md](../README.md): the
container is up, healthy, and `vpm init-db` has run. Everything below
happens through the dashboard in your browser, except the one command
that mints your first password.

## 1. Create your admin login

VPM has exactly one administrator account, username `admin`. There is no
command to set its password directly — the only way in is a random,
15-minute, one-time password that you immediately have to change. This is
deliberate: no permanent password ever exists outside your own head.

Mint one:

```sh
docker compose exec vpm vpm issue-admin-temporary-password --emit-secret-on-stdout
```

This prints a temporary password to your terminal and immediately signs
out any other session that might exist. Use it right away — it expires in
15 minutes and only works once.

In your browser:

1. Open `https://localhost` (or your chosen `HOST_HTTPS_PORT`). Click
   through the certificate warning — see
   [troubleshooting.md](troubleshooting.md#certificate-warning-on-localhost)
   if that concerns you.
2. Sign in as `admin` with the temporary password.
3. You'll land on a forced **change password** page. Pick a real password
   and submit it.
4. You're sent back to the login page. Sign in again with your new,
   permanent password.

You should now see the VPM dashboard (the "Overview" page). If you get an
`Invalid request.` error on the very first login attempt, see
[troubleshooting.md](troubleshooting.md#invalid-request-on-first-login).

## 2. Find your Vast account ID

Before VPM will accept your Vast API key, it needs to know which Vast.ai
account you expect that key to belong to — this stops a key for the wrong
account from being installed by mistake. Set `VPM_EXPECTED_ACCOUNT_ID` in
your `.env` file to that account's numeric ID, then restart the stack
(`docker compose up -d`) to pick it up.

The ID VPM checks is the plain numeric `id` your Vast account already has
(the same value the Vast API returns as your user/account ID). You can
get it either way:

- Log in to the [Vast.ai console](https://cloud.vast.ai) and look at your
  Account page — your account/user ID is shown there.
- If you use the official `vastai` command-line tool, `vastai show user`
  prints your account details, including this ID.

Leave `VPM_EXPECTED_ACCOUNT_ID` empty for now if you just want to look
around the dashboard first — VPM runs fine without it, it will just
refuse to store a key (with a clear `expected_account_not_configured`
reason on `/readyz`) until you set it.

## 3. Create a Vast API key with the right scopes

In the Vast.ai console, create a new API key. It must include these
scopes (VPM checks for all four and refuses a key missing any of them):

- `user_read`
- `machine_read`
- `billing_read`
- `misc`

You only need one key. VPM stores the same value internally in two
encrypted roles (a "reader" and a "writer" slot) — you don't create two
keys yourself.

## 4. Add the key to VPM

1. Sign in and open **Credentials** in the dashboard.
2. Paste the API key and re-enter your dashboard password (every
   sensitive action re-checks your password, not just your session).
3. Submit.

VPM validates that the key belongs to the account you configured in step
2, then runs a handful of read-only checks against your account before
storing it. If the account doesn't match, you'll get a plain
"account identity mismatch" error — go back and check
`VPM_EXPECTED_ACCOUNT_ID`, not the key itself.

Once the key is accepted, run a sync so VPM can see your machines:

```sh
docker compose exec vpm vpm sync
```

The Credentials page then lists every machine VPM can prove you own, each
with its own **Manage GPU pricing** and **Manage rolling availability**
checkboxes — both unchecked. An empty list here is normal and valid; it
just means VPM found no machines to manage yet, or hasn't synced.

## 5. What "writes disabled" actually means

With writes off (the default), VPM will:

- keep syncing your inventory, market prices, and earnings;
- show you dry-run pricing and end-date decisions; and
- let you review everything it *would* do.

It will not change a live listing, price, or end-date on Vast.ai. Nothing
you do in steps 1–4 above makes a live change to your account. Two
endpoints reflect this while you're getting set up:

- `GET /healthz` — is the process alive? (`{"ok": true}` once the
  container is up.)
- `GET /readyz` — is VPM fully configured and synced? While you're
  working through this page, expect reasons like
  `expected_account_not_configured` or `read_sync_stale` to disappear one
  at a time as you complete each step. This is expected, not an error.

## 6. Enabling writes safely

Turning writes on is intentionally a two-step, two-place action, and
neither step alone is enough:

1. **Process-level switch.** Set `VPM_WRITES_ENABLED=true` in your `.env`
   and restart the stack (`docker compose up -d`). This alone does not
   let anything happen — it only makes the in-app switch below available.
2. **In-app confirmation.** Open **Settings** in the dashboard, re-enter
   your password, and type the exact confirmation phrase the page asks
   for. Only after both of these are true can you check a per-machine box.
3. **Per-machine opt-in.** On each machine's page, check **Manage GPU
   pricing** and/or **Manage rolling availability** as you want, re-entering
   your password again. Each machine needs fresh proof you own it before
   its checkbox takes effect. Unchecking a box never needs a password.

Start with writes off, watch the dry-run decisions for a while, and only
enable a machine you've actually reviewed. You can turn the process-level
switch back off at any time; doing so does not need a password and takes
effect immediately.
