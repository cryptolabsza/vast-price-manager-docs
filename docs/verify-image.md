# Verifying the image signature

Every published VPM image is signed keylessly with
[cosign](https://docs.sigstore.dev/cosign/) via GitHub Actions OIDC, and
the signature is attached to the image's digest, not to a mutable tag.
Verify it before you pull the image into anything you care about.

## Install cosign

You need **cosign 3.0 or later**. Images are signed with cosign 3, which
stores the signature in a newer format; cosign 2.x reports
`no signatures found` for them.

Pick one:

```sh
brew install cosign          # macOS
```

or download a release binary from the
[cosign releases page](https://github.com/sigstore/cosign/releases) and
follow its install instructions for your platform.

Or install nothing and run cosign from its official container image:

```sh
docker run --rm gcr.io/projectsigstore/cosign:v3.0.6 verify \
  --certificate-identity https://github.com/cryptolabsza/vast-price-manager/.github/workflows/release-image.yml@refs/tags/v0.3.2 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/cryptolabsza/vast-price-manager@sha256:b99d1c88723e0182ca297725219ed65e603ed3fa939d1d8caa61b1417940e927
```

## Verify

Look up the pinned reference and version for the release you want in
[RELEASES.md](../RELEASES.md), then run:

```sh
cosign verify \
  --certificate-identity https://github.com/cryptolabsza/vast-price-manager/.github/workflows/release-image.yml@refs/tags/v<version> \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  <pinned ref>
```

For example, for version `0.3.2`:

```sh
cosign verify \
  --certificate-identity https://github.com/cryptolabsza/vast-price-manager/.github/workflows/release-image.yml@refs/tags/v0.3.2 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/cryptolabsza/vast-price-manager@sha256:b99d1c88723e0182ca297725219ed65e603ed3fa939d1d8caa61b1417940e927
```

A successful check prints "The cosign claims were validated", confirms the
transparency-log entry and the certificate chain, and then prints the
verified signature entries as JSON. If it fails, do not run that
image — either the reference is wrong, or the image was not built and
signed by the expected release workflow.

## Why the identity looks like this

The `--certificate-identity` value is not a person or an organization —
it's the exact GitHub Actions workflow file and the exact tag that
triggered it. This proves the image was built by VPM's own tag-triggered
release workflow on GitHub-hosted runners, not by anyone with push access
to the container registry, and not by a workflow run against an
arbitrary branch.

## What this doesn't cover

Signature verification proves the image came from VPM's release workflow
for the tag it claims. It does not audit the code inside the image — for
that, VPM's source repository (private) carries its own CI and review
history. This repo publishes only the image and its signature, not the
source.
