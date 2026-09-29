# Safety model

VPM defaults to doing nothing to your live Vast.ai listings. Everything
below exists to keep it that way until you explicitly say otherwise.

## Nothing is live by default

The container starts with writes disabled. It only reads your account and
shows you what it would do.

## Turning writes on is deliberate

In the appliance default, Settings stores the global write choice after a
current-password confirmation. An explicit `VPM_WRITES_ENABLED` environment
value overrides that browser control and needs a restart to change. The manual
standalone/Caddy route uses that explicit process-level setting before its
in-app confirmation. In every deployment, a machine still needs its own
reviewed management opt-in before VPM can change it.

## Per-machine controls are opt-in and independent

Enabling pricing or availability management on one machine needs a fresh
password check and fresh proof you own that machine. Turning either off
does not.

## Uncertain state means "held," not "guessed"

Any missing or contradictory fact about a machine's ownership, listing,
price, or freshness makes VPM leave it alone rather than act on
incomplete information.

## Your Vast API key is encrypted at rest

It's stored encrypted. The appliance keeps its master key in `/keys` and state
in `/data`; the manual recipe keeps `master.key` outside its data volume.
See [upgrade-and-backup.md](upgrade-and-backup.md) for how to back up both
pieces correctly.

## Login is HTTPS-only

Session cookies are marked `Secure`. The appliance serves HTTPS itself with a
generated local certificate or mounted TLS material. The separate manual
standalone recipe uses Caddy as its TLS terminator.
