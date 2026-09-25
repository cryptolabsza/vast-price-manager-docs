# Security policy

## Reporting a vulnerability

VPM's source code lives in a private repository, so this public repo is
the intended contact point. Please report suspected vulnerabilities —
whether in the published container image, the deployment files in this
repo (`compose.yml`, `Caddyfile`), or the documented setup steps — using
GitHub's private vulnerability reporting on this repository:

**Security tab → "Report a vulnerability"** (or go directly to this
repo's `/security/advisories/new` page).

This opens a private draft advisory visible only to you and the
maintainers, so details don't become public before a fix is ready. Please
do not open a public issue for a suspected vulnerability.

We don't publish a contact email; private vulnerability reporting is the
only channel we ask you to use.

## What to include

- What you found and where (an endpoint, a config file, a step in the
  docs, the image itself).
- Steps to reproduce, if you have them.
- What you'd expect to happen instead.

## Scope

In scope: the published VPM container image, the files in this repo, and
the setup this repo documents. Out of scope: Vast.ai's own platform and
API (report those to Vast.ai directly) and any other, separate project
that can also deploy VPM.
