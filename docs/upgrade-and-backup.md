# Upgrade and backup

## Why you pin by digest, not by tag

VPM's database migrations only ever go forward. There is no supported way
to move a database back to an older schema. If you ran on a floating tag
like `:latest`, a routine pull could silently jump you past a schema
change with no way back. That's why every image reference in this repo is
a full `image:version@sha256:digest` pin — see [RELEASES.md](../RELEASES.md)
for the current one, and [verify-image.md](verify-image.md) to check its
signature before you run it.

Treat every upgrade as one-way. Back up before you do it, not after.

## What to back up, and why both pieces matter

Two things, together, every time — before you touch anything:

1. **The `vpm_data` volume.** This holds the SQLite database and your
   Vast API key, encrypted.
2. **`master.key`**, the file next to your `compose.yml`.

The key is deliberately kept outside the data volume. Back it up
separately, and store the two backups somewhere other than side-by-side —
either one alone is useless to an attacker, but together they decrypt
your stored Vast API key.

Back up the volume with a short-lived helper container so you don't have
to stop VPM to read it. Compose prefixes the volume name with your
project directory's name, so find the exact name first:

```sh
VOL=$(docker volume ls -q --filter name=vpm_data)
docker run --rm \
  -v "$VOL":/data:ro \
  -v "$PWD":/backup \
  alpine tar czf /backup/vpm-data-$(date +%Y%m%d).tar.gz -C / data
```

Back up the key at the same time. `master.key` is owned by UID 999 —
see the quickstart in [README.md](../README.md) for why — so reading it
back needs `sudo`; hand the copy back to yourself right after so it's a
normal file you can move around:

```sh
STAMP=$(date +%Y%m%d)
sudo cp master.key "master.key.$STAMP.bak"
sudo chown "$(id -u):$(id -g)" "master.key.$STAMP.bak"
chmod 600 "master.key.$STAMP.bak"
```

## Upgrading

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

## Restoring from a backup

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
