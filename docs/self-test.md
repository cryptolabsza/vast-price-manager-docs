# Machine self-test

VPM can run Vast's own machine self-test against one of your machines from
a machine's diagnostics page. This uses Vast's own command-line tool
("the Vast CLI", package name `vastai`) under the hood — and the public
VPM image does **not** ship it.

## Why it isn't already installed

`vastai` pulls in about sixty third-party packages (roughly 220 MB) that
VPM otherwise never touches, including `borb`, which is AGPL-licensed,
and it pins older versions of `cryptography` and `pillow` than VPM itself
uses. Baking all of that into every copy of the public image, for one
optional command, isn't a trade worth making by default. Instead, you
install it yourself, on demand, only if and when you want to self-test a
machine.

## What you'll see

A machine's page shows the Vast CLI's status as one of:

- **not installed** — the default state.
- **installing** — an install is running in the background.
- **installed (version …)** — ready to use.
- **install failed: *reason*** — the last attempt didn't finish; you can
  try again.
- **not supported in this deployment** — this VPM install has no working
  `pip` for its own base Python interpreter, so an install isn't possible
  here.

## Installing it

If you're signed in as the administrator, you'll see an **Install Vast
CLI** button next to the status. Click it and re-enter your dashboard
password (the same re-authentication every sensitive action requires).

Behind that click, VPM:

- downloads a fixed, hash-checked set of packages from PyPI (about 60
  packages, about 220 MB) — every package and version is pinned in
  advance, and the install refuses anything that doesn't match its
  expected hash;
- runs the download as the same unprivileged user the container already
  runs as, never as root;
- writes the result into a folder inside your data volume, and only
  switches it live if the whole download succeeds — a failed or partial
  attempt is simply discarded, never shown as installed.

You need outbound internet access to `pypi.org` for this to work. If your
VPM host has no outbound internet access at all, installation will fail
or show as unsupported.

## The same-network limitation (read this before you self-test)

**Self-test does not work when VPM runs on the same local network as the
machine you're testing.** The test rents a short-lived instance on that
machine and connects to the machine's own public IP address — and most
home and office routers cannot route traffic that starts inside their own
network back out to their own public IP (this is usually called "NAT
hairpinning" or "NAT loopback"). The Vast CLI's own diagnostics report the
same limitation; it isn't specific to VPM.

Run VPM from a different network than the machine you want to self-test —
for example, a different site, or a small always-on box outside the LAN
your machines are on.

## Licensing

The packages the Vast CLI installer downloads are not part of the
published VPM image — you fetch them yourself, directly from PyPI, only
if you choose to. Each of them stays under its own original licence once
installed, including `borb`, which is AGPL-licensed. Installing the Vast
CLI does not change the terms this repo's [LICENSE](../LICENSE) sets for
the VPM image itself.

## Where it lives, and what it doesn't affect

The installed CLI sits entirely inside a `vast-cli` folder in your data
volume — the same volume [docs/upgrade-and-backup.md](upgrade-and-backup.md)
already tells you to back up. A few things follow from that:

- It survives a normal upgrade (a new VPM image version doesn't remove it).
- It's safe to leave out of a backup if you want a smaller backup — if
  it's missing after a restore, the status just goes back to "not
  installed", and you can click the button again.
- The older `cryptography`/`pillow` versions the Vast CLI needs stay
  confined to this one folder. VPM's own web app and everything else it
  does keep using VPM's own, newer `cryptography` — the older copies are
  never loaded outside a self-test run.
