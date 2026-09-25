# First run

Picks up after the [README](../README.md) quickstart — container up,
healthy, `vpm init-db` done. Everything below is browser-based, except
the one command that mints your first password.

## 1. Create your admin login

VPM has one admin account (`admin`). There's no command to set its
password directly — you mint a random, 15-minute, one-time password and
change it immediately.

```sh
docker compose exec vpm vpm issue-admin-temporary-password --emit-secret-on-stdout
```

This prints the password and signs out every other session. Use it right
away — it expires in 15 minutes and only works once.

Then, in your browser:

1. Open `https://localhost` (or your `HOST_HTTPS_PORT`). Click through
   the certificate warning
   ([why](troubleshooting.md#certificate-warning-on-localhost)).
2. Sign in as `admin` with the temporary password.
3. On the forced **change password** page, set a real password.
4. Sign in again with your new password.

You should land on the Overview page. Got `Invalid request.` instead?
See [troubleshooting.md](troubleshooting.md#invalid-request-on-first-login).

## 2. Find your Vast account ID

VPM only accepts a Vast API key that matches one expected account — this
stops the wrong account's key from being installed by mistake.

Set `VPM_EXPECTED_ACCOUNT_ID` in `.env` to your account's numeric ID, then
restart with `docker compose up -d`.

Find the ID:

- Vast.ai console → Account page, or
- the official CLI: `vastai show user`.

Leave it empty for now if you just want to look around first. VPM works
fine without it — it simply won't store a key yet (`/readyz` shows
`expected_account_not_configured`).

## 3. Create a Vast API key with the right scopes

In the Vast.ai console, create a new API key with all four scopes (VPM
rejects a key missing any):

- `user_read`
- `machine_read`
- `billing_read`
- `misc`

One key is enough — VPM stores it internally as both a "reader" and a
"writer" slot.

## 4. Add the key to VPM

1. Open **Credentials** in the dashboard.
2. Paste the API key and re-enter your password.
3. Submit.

VPM checks the key's account against `VPM_EXPECTED_ACCOUNT_ID` before
anything else. A mismatch means check that setting, not the key.

Once accepted, sync:

```sh
docker compose exec vpm vpm sync
```

The Credentials page then lists every machine VPM can prove you own, each
with **Manage GPU pricing** / **Manage rolling availability** checkboxes —
both off. An empty list just means nothing has synced yet.

## 5. What "writes disabled" means

With writes off (the default), VPM keeps syncing, drafts pricing and
end-date decisions, and lets you review them. It never touches a live
listing.

- `GET /healthz` — is the process alive? (`{"ok": true}` once up.)
- `GET /readyz` — is VPM fully configured and synced? Reasons like
  `expected_account_not_configured` or `read_sync_stale` clear one by one
  as you finish each step above — expected, not an error.

## 6. Enabling writes safely

Two deliberate steps, both required:

1. **Process-level switch.** Set `VPM_WRITES_ENABLED=true` in `.env` and
   restart (`docker compose up -d`). Alone, this changes nothing — it
   just unlocks the in-app switch.
2. **In-app confirmation.** In **Settings**, re-enter your password and
   type the exact confirmation phrase. Only then can you check a
   per-machine box.
3. **Per-machine opt-in.** On a machine's page, check **Manage GPU
   pricing** / **Manage rolling availability**, re-entering your password
   each time. Unchecking never needs a password.

Start with writes off, watch the dry-run decisions for a while, and
enable only machines you've reviewed. Turn the process switch back off
any time — no password needed, takes effect immediately.
