# Configuration reference

Every setting VPM reads from the environment, grouped by how often you'll
actually touch it. Two settings — `compose.yml`'s `VPM_ALLOWED_HOSTS` and
`.env`'s `VPM_EXPECTED_ACCOUNT_ID` — are already wired up for you in this
repo's `compose.yml`; everything else is an environment variable you can
add to the `vpm` service's `environment:` block in `compose.yml` if you
need it.

## Settings you'll actually set

| Variable | Default | When to set it |
|---|---|---|
| `VPM_CREDENTIAL_MASTER_KEY_FILE` | none (required) | Already set in `compose.yml` to the Compose secret file. Don't change this unless you change how the master key is mounted. |
| `VPM_EXPECTED_ACCOUNT_ID` | none | Set to your Vast.ai numeric account ID before you can add your API key. See [first-run.md](first-run.md#2-find-your-vast-account-id). |
| `VPM_ALLOWED_HOSTS` | `127.0.0.1,localhost` | Set to the exact hostname you browse to (must match your Caddyfile site block). Required, non-empty; the first entry is also the Host header VPM's own healthcheck uses. |
| `VPM_WRITES_ENABLED` | `false` | Only after you've read [first-run.md](first-run.md#6-enabling-writes-safely) in full. This is the process-level half of the write gate; it needs a container restart to take effect either way. |

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
| `VPM_AUTH_FAILURE_LIMIT` | `5` | Failed login attempts (per bucket) before a temporary lockout. |
| `VPM_AUTH_LOCK_SECONDS` | `900` | Length of that lockout. |
| `VPM_ARGON2_CONCURRENCY` | `2` | Password-hashing CPU parallelism. Only worth changing on a very constrained or very large host. |
| `VPM_MARKET_RETENTION_PER_MACHINE` | `96` | How many historical market snapshots VPM keeps per machine. |
| `VPM_SYNC_RUN_RETENTION` | `500` | How many past sync-run records VPM keeps. |
| `VPM_DECISION_RETENTION` | `5000` | How many past pricing/end-date decisions VPM keeps. |
| `VPM_AUDIT_RETENTION` | `10000` | How many audit-log entries VPM keeps. |

## Settings that don't apply to this recipe

These exist in VPM for other deployment shapes — for example, being
embedded behind another proxy as part of a larger internal deployment.
Leave them unset for the standalone setup in this repo.

| Variable | Default | Notes |
|---|---|---|
| `VPM_DEPLOYMENT_MODE` | `standalone` | The other value, `container_proxy`, is for being embedded behind another proxy's network and needs several other settings alongside it. Not used here. |
| `VPM_BASE_PATH` | empty (root) | Only meaningful with `container_proxy` mode above. |
| `VPM_AUTH_MODE` | `standalone` | The other value, `fleet`, defers all login to an external authority and requires `container_proxy` mode. Not used here. |
| `VPM_FLEET_AUTH_URL` | none | Only used with `VPM_AUTH_MODE=fleet`. |
| `VPM_FLEET_AUTH_TIMEOUT_SECONDS` | `5` | Only used with `VPM_AUTH_MODE=fleet`. |

## One setting to never touch here

`VPM_SESSION_INSECURE` exists for local, no-container, no-proxy Python
development only. Setting it turns off the `Secure` flag on session
cookies, which breaks the HTTPS-only login boundary this whole recipe is
built around. Nothing in this repo needs it, and it's absent from
`compose.yml` on purpose — leave it that way.
