# Upgrade and backup

VPM 0.4.0 uses database schema 14. Migrations move forward only. Do not start
an older image against a volume that a newer image has migrated; roll back by
restoring the matching backup instead.

## Why you pin by digest, not by tag

VPM's database migrations only ever go forward. There is no supported way
to move a database back to an older schema. If you ran on a floating tag
like `:latest`, a routine pull could silently jump you past a schema
change with no way back. That's why every image reference in this repo is
a full `image:version@sha256:digest` pin — see [RELEASES.md](../RELEASES.md)
for the current one, and [verify-image.md](verify-image.md) to check its
signature before you run it.

Treat every upgrade as one-way. Back up before you do it, not after.

## Appliance: back up both named volumes

The appliance keeps its database, encrypted credential slots, and generated TLS
material in `vpm_data`, and its credential master key in `vpm_keys`. Back up
both volumes together before an upgrade. Stop the container first so the SQLite
database is captured consistently:

```sh
docker stop vpm
docker run --rm -v vpm_data:/data:ro -v "$PWD":/backup \
  alpine tar czf /backup/vpm-data-$(date +%Y%m%d).tar.gz -C / data
docker run --rm -v vpm_keys:/keys:ro -v "$PWD":/backup \
  alpine tar czf /backup/vpm-keys-$(date +%Y%m%d).tar.gz -C / keys
docker start vpm
```

Those volume names apply to the `docker run` quickstart. Compose may prefix
them with its project name; use `docker volume ls` to identify the two matching
volumes before copying the commands. Store the two archives securely. Together,
they can recover encrypted credentials.

To upgrade the appliance, back up both volumes, replace its image reference with
the new verified digest, then recreate the container using the same `vpm_data`
and `vpm_keys` mounts from [setup-wizard.md](setup-wizard.md). Docker will run
the migration at startup. Check `docker logs vpm` and `https://YOUR-SERVER:8443/readyz`
afterward. If you need to return to an earlier release, create fresh volumes and
restore the matching two archives before starting the earlier image; do not
blindly point that image at migrated data.

## Manual standalone/Caddy: what to back up

Two things, together, every time — before you touch anything:

1. **The `vpm_data` volume.** This holds the SQLite database and your
   Vast API key, encrypted.
2. **`master.key`**, the file next to your `compose.yml`.

The key is deliberately kept outside the data volume. Back it up
separately, and store the two backups somewhere other than side-by-side —
either one alone is useless to an attacker, but together they decrypt
your stored Vast API key.

If you installed the optional Vast CLI (see [docs/self-test.md](self-test.md)),
its `vast-cli` folder lives inside the `vpm_data` volume too, so the
backup command below already includes it — but it's safe to leave out of
a smaller backup if you want one, since it's just a downloaded copy: a
missing `vast-cli` folder after a restore simply shows as "not installed"
again, and the Install button puts it back.

Stop VPM before creating the archive. This is a plain file tar, not a SQLite
backup API, so a live SQLite database and its WAL files do not provide a
consistent snapshot. Compose prefixes the volume name with your project
directory's name, so find the exact name first:

```sh
docker compose stop vpm
VOL=$(docker volume ls -q --filter name=vpm_data)
STAMP=$(date +%Y%m%d)
docker run --rm \
  -v "$VOL":/data:ro \
  -v "$PWD":/backup \
  alpine tar czf /backup/vpm-data-$STAMP.tar.gz -C / data
sudo cp master.key "master.key.$STAMP.bak"
sudo chown "$(id -u):$(id -g)" "master.key.$STAMP.bak"
chmod 600 "master.key.$STAMP.bak"
docker compose start vpm
```

`master.key` is owned by UID 999 — see the manual first-run guide for why —
so its copy needs `sudo`; the `chown` immediately returns the backup copy to
your account. Keep the volume archive and matching key backup together.

## Manual standalone/Caddy: upgrading

1. Back up both pieces above.
2. Look up the new version's pinned reference in [RELEASES.md](../RELEASES.md)
   and verify it with [verify-image.md](verify-image.md).
3. Update `VPM_IMAGE` in your `.env` to the new pinned reference.
4. Pull and restart:

   ```sh
   docker compose pull vpm
   docker compose up -d
   ```

5. Check `docker compose ps` (both services should be healthy) and
   `curl -k https://localhost/readyz` to confirm VPM came back up
   configured the way you left it.

`docker compose down` (without `-v`) between steps 3 and 4 is safe and
does not touch the `vpm_data` volume or `master.key` — only
`docker compose down -v` deletes the volume, which you should not run
during a normal upgrade.

## Manual standalone/Caddy: restoring from a backup

If an upgrade goes wrong, or you're moving to a new host:

```sh
docker compose down
VOL=$(docker volume ls -q --filter name=vpm_data)   # or docker volume create one on a new host
docker run --rm \
  -v "$VOL":/data \
  -v "$PWD":/backup \
  alpine sh -c "cd / && tar xzf /backup/vpm-data-YYYYMMDD.tar.gz"
sudo cp master.key.YYYYMMDD.bak master.key
sudo chown 999:999 master.key && sudo chmod 400 master.key
docker compose up -d
```

The `chown`/`chmod` here are the same, and just as necessary, as the ones
in the quickstart — a restored key is a fresh file on this host and
starts out owned by whoever ran `cp`, not by VPM's container user.

Restore the database backup and the matching master-key backup together —
a database from one date paired with a key from another will not decrypt
your stored Vast API key, and you'll need to re-enter it from scratch via
[first-run.md](first-run.md).
