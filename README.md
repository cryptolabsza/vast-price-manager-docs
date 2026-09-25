# Vast Price Manager (VPM)

VPM is a self-hosted dashboard for people who rent out GPU or CPU machines
on [Vast.ai](https://vast.ai). It watches your machines, the current market
price for your GPU type, and your earnings, and it can adjust your listing
price and rental end-date for you — but only for machines you explicitly
tell it to manage, and only once you turn that on yourself.

This repo is the public quickstart for running VPM as a standalone Docker
container on your own machine. It contains documentation and deployment
files only — no application source code. The VPM source stays private; you
run it as a signed, published container image.

There is also a fuller self-hosted monitoring suite ("dc-overview") that
can deploy and manage VPM alongside other datacenter tooling. This repo
does not need it and does not assume you have it — everything here works
standalone.

## Who this is for

Anyone running one or more machines on Vast.ai who wants:

- a dashboard that keeps a synced, read-only view of their fleet and
  earnings without needing anything installed on the machines themselves;
- pricing decisions they can review before anything goes live; and
- a tool that defaults to doing nothing to their live listings until they
  say otherwise.

It is not a general Vast.ai account manager and it does not replace the
Vast.ai console — it is a narrow, careful layer on top of one account.

## What it does

- Syncs your machine inventory, ownership, and earnings from your Vast.ai
  account (read-only).
- Tracks the current market price for each of your GPU types.
- Produces dry-run pricing and end-date decisions you can inspect before
  anything is applied.
- Lets you opt individual machines into two independent controls —
  "Manage GPU pricing" and "Manage rolling availability" — each off by
  default.
- Supports scheduled maintenance windows, with a preview step before you
  commit to one.
- Keeps an audit log of every decision and command it runs.
- Has exactly one administrator account, protected by a password that is
  re-checked before every sensitive action (adding a key, turning writes
  on, changing what a machine can do).

## Safety model, in plain words

- **Nothing is live by default.** The container starts with writes
  disabled. It only reads your account and shows you what it would do.
- **Turning writes on is a deliberate, two-step act.** First you flip a
  process-level switch (an environment variable, which needs a restart);
  then, inside the dashboard, you type an exact confirmation phrase after
  re-entering your password. See
  [docs/first-run.md](docs/first-run.md#enabling-writes-safely).
- **Per-machine controls are opt-in and independent.** Enabling pricing or
  availability management on one machine needs a fresh password check and
  fresh proof you own that machine; turning either off does not.
- **If VPM isn't sure, it treats a machine as held, not as safe to
  change.** Any missing or contradictory fact about a machine's ownership,
  listing, price, or freshness makes VPM leave it alone rather than guess.
- **Your Vast API key never leaves the container in the clear.** It is
  stored encrypted at rest, using a separate master key file that is kept
  outside the data volume on purpose — see
  [docs/first-run.md](docs/first-run.md) and
  [docs/upgrade-and-backup.md](docs/upgrade-and-backup.md) for why that
  split matters and how to back both pieces up correctly.
- **Login is HTTPS-only.** Session cookies are marked `Secure`, so VPM
  always needs a TLS-terminating proxy in front of it — this repo ships
  one (Caddy) that is set up for you.

## 10-minute quickstart (try it on your own machine)

Requirements: Docker and Docker Compose. Nothing else to install.

1. Get this repo (clone it, or download `compose.yml`, `Caddyfile`, and
   `.env.example`) and open a terminal in that directory.
2. Copy the environment template and fill in the image reference:

   ```sh
   cp .env.example .env
   ```

   Set `VPM_IMAGE` in `.env` to the current pinned image from
   [RELEASES.md](RELEASES.md), and check its signature first with
   [docs/verify-image.md](docs/verify-image.md). Never use `:latest`.

3. Generate the master key that will encrypt your Vast API key at rest.
   Do this *before* `docker compose up` — Compose needs the file to exist
   to start the `vpm` service at all. This uses the VPM image's own
   Python, so you don't need anything else installed:

   ```sh
   docker run --rm "$(sed -n 's/^VPM_IMAGE=//p' .env)" \
     python -c "from cryptography.fernet import Fernet; import sys; sys.stdout.buffer.write(Fernet.generate_key())" \
     > master.key
   sudo chown 999:999 master.key && sudo chmod 400 master.key
   ```

   The `chown` matters on a native Linux Docker host: VPM always runs as
   UID 999 inside its container, and a plain `chmod 600` alone leaves the
   file owned by *you*, which VPM's own user cannot read — Compose's
   file-based secret is a bind mount, not a copy, so the host file's
   owner and mode are exactly what the container sees. (Docker Desktop's
   file-sharing layer can hide this — see
   [docs/troubleshooting.md](docs/troubleshooting.md#master-key-not-readable) —
   which is why the `chown` is still worth doing there too.)

4. Start the stack and initialize the database:

   ```sh
   docker compose up -d
   docker compose exec vpm vpm init-db
   ```

5. Open `https://localhost` (or `https://localhost:$HOST_HTTPS_PORT` if you
   changed the port). Your browser will warn you the certificate isn't
   trusted — that's expected for the local `tls internal` setup and is
   explained in [docs/first-run.md](docs/first-run.md).

6. Continue in **[docs/first-run.md](docs/first-run.md)** to create your
   admin login, find your Vast account ID, add your Vast API key, and
   (only when you're ready) turn writes on.

## Documentation

- [docs/first-run.md](docs/first-run.md) — admin bootstrap, finding your
  Vast account ID, creating a scoped Vast API key, adding it, and safely
  enabling writes.
- [docs/configuration.md](docs/configuration.md) — every setting VPM
  reads from the environment, its default, and when you'd change it.
- [docs/upgrade-and-backup.md](docs/upgrade-and-backup.md) — pinning by
  digest, backing up the data volume and master key, and restoring.
- [docs/verify-image.md](docs/verify-image.md) — checking the image's
  signature before you run it.
- [docs/troubleshooting.md](docs/troubleshooting.md) — common first-run
  problems and how to read them.
- [RELEASES.md](RELEASES.md) — version → signed image table.
- [SECURITY.md](SECURITY.md) — how to report a vulnerability.
- [LICENSE](LICENSE) — terms of use for the published image.
