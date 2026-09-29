# Appliance setup: run, open, follow the browser steps

This is the default 0.4.0 installation path. Its image is pinned to the
published release digest below. Verify its signature through
[RELEASES.md](../RELEASES.md); do not use a floating tag.

## 1. Create your container

Use the appliance image with two persistent volumes and one HTTPS port:

```sh
docker run -d --name vpm --restart unless-stopped --stop-timeout 60 \
  -p 8443:8088 \
  -v vpm_data:/data \
  -v vpm_keys:/keys \
  ghcr.io/cryptolabsza/vast-price-manager:0.4.0@sha256:20c1e3a4b66ddd723ea1e3c635e0b7dec3f8cc88fe29f4e27e54f1686954eead
```

Docker Compose users can use `compose.wizard.yml`. No account settings, API
keys, `.env` file, or runtime secret are required at container creation.

The container prepares its database and encryption key automatically. Keep both
volumes when restarting or replacing it.

## 2. Open the dashboard

Open **`https://YOUR-SERVER:8443`**. On the same computer, use
`https://localhost:8443`. If you map another host port, use that port in the URL.
The generated local certificate causes a browser warning on first access. You can
later supply your own certificate or use your existing HTTPS proxy.

Read the initial username and generated password from the logs:

```sh
docker logs vpm
```

Sign in as `admin` within 15 minutes, choose your own username and permanent
password, then sign in again. If the one-time password expires, run:

```sh
docker exec -it vpm vpm issue-admin-temporary-password --emit-secret-on-stdout
```

## 3. Follow the setup wizard

1. **Connect Vast:** create an API key in Vast, paste it, and confirm with your
   VPM password. VPM checks access to your profile, machines, and billing.
2. **Confirm the account:** check the detected account ID before saving the key.
   You do not need to look up the ID or edit `.env`. The key is stored encrypted.
3. **Sync machines:** click Sync. The page shows progress and offers a retry if
   the connection fails. An empty inventory is explained, not treated as a crash.
4. **Set prices:** open a discovered machine and configure fixed prices or
   automatic pricing. Review each machine and enable live Vast changes in
   Settings only when ready; setup does not change any listing.

Confirmed steps survive restarts. A confirmation still waiting for your approval
expires after five minutes; paste the key again if needed. Once configured, login
opens the dashboard directly.

## Prefer manual configuration?

Keep using `.env`, Docker environment variables, mounted secrets and CLI commands.
The wizard does not edit your `.env` or replace existing credentials.

- `VPM_EXPECTED_ACCOUNT_ID` overrides discovery: a key must match the configured
  account. With no value, the wizard discovers and binds the account from the
  accepted key.
- `VPM_WRITES_ENABLED` overrides the appliance browser setting. Set it to
  `false` to keep live changes disabled or `true` to supply an explicit
  process-level allowance; changing it requires a container restart. When it is
  omitted, the Settings control persists the appliance choice.
- The appliance creates `/keys/master.key` on first start. An explicitly mounted
  `VPM_CREDENTIAL_MASTER_KEY_FILE` is reused; never discard that key while
  retaining encrypted API keys.
- `VPM_ALLOWED_HOSTS` is optional in appliance mode. Set it before first start
  when you want the generated local certificate to include the hostname you use;
  existing generated TLS files are retained.
- Set both `VPM_TLS_CERT_FILE` and `VPM_TLS_KEY_FILE` to use mounted TLS material.
- `VPM_SETUP_ENABLED=false` disables guided setup. Local-auth standalone can
  also use guided setup when it is enabled, or follow the manual account-ID path.
- Existing `standalone` and `container_proxy` deployments remain supported.

Pass explicit overrides with `docker run --env-file .env ...` only when you need
them. For Compose, `.env` supplies interpolation values; add runtime overrides
to the service's `environment` or `env_file`. The existing `compose.yml` remains
the [manual Caddy layout](first-run.md); the appliance recipe does not replace it.

The wizard fills missing settings and skips completed steps. A fully configured
manual installation proceeds straight to the dashboard.

## Backups and stopping

Back up both `/data` and `/keys`. The database schema is 14 and migrations do not
go backwards: restore a matching backup instead of starting an older image against
upgraded data. The container refuses to replace a missing master key when encrypted
credentials already exist. Restore that key from backup.
Allow 60 seconds for container shutdown with the default provider timeout; increase
the stop grace if you increase the provider timeout. The container handles its own
five-minute sync, hourly pricing and daily horizon tasks.
