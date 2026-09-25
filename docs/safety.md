# Safety model

VPM defaults to doing nothing to your live Vast.ai listings. Everything
below exists to keep it that way until you explicitly say otherwise.

## Nothing is live by default

The container starts with writes disabled. It only reads your account and
shows you what it would do.

## Turning writes on takes two deliberate steps

First you flip a process-level switch (an environment variable, which
needs a restart). Then, inside the dashboard, you type an exact
confirmation phrase after re-entering your password. See
[first-run.md](first-run.md#6-enabling-writes-safely).

## Per-machine controls are opt-in and independent

Enabling pricing or availability management on one machine needs a fresh
password check and fresh proof you own that machine. Turning either off
does not.

## Uncertain state means "held," not "guessed"

Any missing or contradictory fact about a machine's ownership, listing,
price, or freshness makes VPM leave it alone rather than act on
incomplete information.

## Your Vast API key is encrypted at rest

It's stored encrypted, using a separate master key file kept outside the
data volume on purpose. See [first-run.md](first-run.md) and
[upgrade-and-backup.md](upgrade-and-backup.md) for why that split matters
and how to back both pieces up correctly.

## Login is HTTPS-only

Session cookies are marked `Secure`, so VPM always needs a
TLS-terminating proxy in front of it. This repo ships one (Caddy),
already configured for you.
