# Configuration reference

Every setting VPM reads from the environment, grouped by how often you will
touch it. The appliance installation normally needs none of them. The manual
standalone Caddy recipe wires its values through `.env` and `compose.yml`.
An explicit environment value always takes precedence over appliance discovery.

## Settings you'll actually set

| Variable | Default | When to set it |
|---|---|---|
| `VPM_CREDENTIAL_MASTER_KEY_FILE` | appliance: `/keys/master.key`; standalone: none (required) | The appliance creates and retains its key in the separate `/keys` volume. The manual Compose recipe mounts `master.key` as a secret. Do not move either key inside `/data`. |
| `VPM_EXPECTED_ACCOUNT_ID` | none | Optional in guided local-auth setup (appliance or standalone): if set, it overrides account discovery and the key must match it. Required before the manual account-ID path accepts a key; see [first-run.md](first-run.md#2-find-your-vast-account-id). |
| `VPM_ALLOWED_HOSTS` | appliance: `*`; standalone: `127.0.0.1,localhost` | In appliance mode, set it before first start to include hostnames in the generated local certificate; existing generated TLS files are retained. In the manual recipe, set the exact hostname that matches Caddy. |
| `VPM_WRITES_ENABLED` | `false` | An explicit value overrides appliance browser control and takes effect when the process is recreated. When omitted in appliance mode, Settings persists the global write choice. The manual Compose recipe exposes this as `.env` interpolation and uses the process-level gate. |
| `VPM_SETUP_ENABLED` | `true` | Available in local-auth appliance and standalone deployments. The manual Compose recipe explicitly defaults it to `false` for the account-ID/credential path; set it to `true` in `.env` only to use guided local-auth setup. |
| `VPM_TLS_CERT_FILE` + `VPM_TLS_KEY_FILE` | generated local certificate | Set both to use mounted TLS material in appliance mode. Supplying only one prevents startup. |

## Settings you'll rarely need

| Variable | Default | When to set it |
|---|---|---|
| `VPM_DATA_DIR` | `/var/lib/vast-price-manager` (the image sets it to `/data`) | Where the SQLite database and encrypted credential slots live. The image already points this at `/data`, matching the `vpm_data` volume in `compose.yml` — don't change it unless you also change that volume mount. |
| `VPM_DATABASE_PATH` | `<VPM_DATA_DIR>/vpm.sqlite3` | Only if you want the database file somewhere other than the default inside `VPM_DATA_DIR`. |
| `VPM_APPROVED_MACHINE_IDS` | empty (no ceiling) | A comma-separated list of Vast machine IDs to restrict discovery to. Leave empty to let VPM discover every machine your account owns. |
| `VPM_APPROVED_READER_FINGERPRINTS` | empty | An optional extra pin on the exact key fingerprint VPM will read with. Not needed for the normal one-key setup in first-run.md. |
| `VPM_APPROVED_WRITER_FINGERPRINTS` | empty | Same idea as above, for the write path. Not needed for the normal setup. |
| `VPM_PROVIDER_BASE_URL` | `https://console.vast.ai` | Only if Vast.ai ever asks you to point at a different API base URL. |
| `VPM_MARKET_TYPE` | `on-demand` | Which Vast market VPM prices against. Leave as-is unless you specifically use the other market type. |
| `VPM_READINESS_MAX_AGE_SECONDS` | `900` | How stale a sync can be before `/readyz` reports `read_sync_stale`. Raise it if you sync deliberately infrequently. |
| `VPM_LEASE_TTL_SECONDS` | `1800` | How long an internal operation lease is held. Leave as-is unless VPM's own docs for your deployment tell you otherwise. |
| `VPM_SESSION_IDLE_SECONDS` | `900` | How long you can be idle in the dashboard before you're signed out. |
| `VPM_SESSION_ABSOLUTE_SECONDS` | `28800` | Hard cap on a session's lifetime regardless of activity. |
| `VPM_ANONYMOUS_SESSION_LIMIT` | `64` | Cap on concurrent pre-login sessions. Only matters under unusual load. |
| `VPM_DECISION_MAX_AGE_SECONDS` | `900` | How fresh the inputs to a pricing/end-date decision must be before VPM will act on it. |
| `VPM_MACHINE_FRESHNESS_SECONDS` | `900` | How fresh a machine's synced state must be before VPM treats it as current. |
| `VPM_OPERATION_DEADLINE_SECONDS` | `1500` | Overall time budget for one sync/cycle/reconcile run. Must stay shorter than `VPM_LEASE_TTL_SECONDS`. |
| `VPM_PROVIDER_TIMEOUT_SECONDS` | `15` | Per-request timeout when calling the Vast.ai API. |
| `VPM_MARKET_SEARCH_LIMIT` | `100` | How many market listings VPM pulls per price check. |
| `VPM_MARKET_REFRESH_SECONDS` | `1800` | How often VPM searches the market for each GPU type, in seconds (300 to 3600). Vast limits offer searches to 20,000 rows per account per day; a shorter interval spends more of it. Since 0.4.1. |
| `VPM_AUTH_FAILURE_LIMIT` | `5` | Failed login attempts (per bucket) before a temporary lockout. |
| `VPM_AUTH_LOCK_SECONDS` | `900` | Length of that lockout. |
| `VPM_ARGON2_CONCURRENCY` | `2` | Password-hashing CPU parallelism. Only worth changing on a very constrained or very large host. |
| `VPM_MARKET_RETENTION_PER_MACHINE` | `96` | How many historical market snapshots VPM keeps per machine. |
| `VPM_SYNC_RUN_RETENTION` | `500` | How many past sync-run records VPM keeps. |
| `VPM_DECISION_RETENTION` | `5000` | How many past pricing/end-date decisions VPM keeps. |
| `VPM_AUDIT_RETENTION` | `10000` | How many audit-log entries VPM keeps. |

## Deployment-shape settings

These settings describe the deployment boundary. The appliance uses its image
default; the manual recipe sets `standalone` explicitly. `container_proxy` is
for an embedded private proxy and needs its additional settings.

| Variable | Default | Notes |
|---|---|---|
| `VPM_DEPLOYMENT_MODE` | image default: `appliance` | `appliance` serves HTTPS on `0.0.0.0:8088` in the container. `standalone` is the manual Caddy recipe and binds loopback HTTP. `container_proxy` is for an embedded private proxy and needs several other settings. |
| `VPM_BASE_PATH` | empty (root) | Only meaningful with `container_proxy` mode above. |
| `VPM_AUTH_MODE` | `standalone` | The other value, `fleet`, defers all login to an external authority and requires `container_proxy` mode. Not used here. |
| `VPM_FLEET_AUTH_URL` | none | Only used with `VPM_AUTH_MODE=fleet`. |
| `VPM_FLEET_AUTH_TIMEOUT_SECONDS` | `5` | Only used with `VPM_AUTH_MODE=fleet`. |

## One setting to never touch here

`VPM_SESSION_INSECURE` exists for local, no-container, no-proxy Python
development only. Setting it turns off the `Secure` flag on session cookies,
which breaks the HTTPS login boundary. Nothing in this repo needs it; leave it
unset.
