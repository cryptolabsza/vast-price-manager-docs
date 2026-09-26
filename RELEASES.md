# Releases

Every row is a signed, tag-triggered build. Verify a signature before you
pull it — see [docs/verify-image.md](docs/verify-image.md). Never use
`:latest`; there is no such tag on this image on purpose (see
[docs/upgrade-and-backup.md](docs/upgrade-and-backup.md)).

| Version | Pinned image reference | Published | Notes |
|---|---|---|---|
| 0.3.2 | `ghcr.io/cryptolabsza/vast-price-manager:0.3.2@sha256:b99d1c88723e0182ca297725219ed65e603ed3fa939d1d8caa61b1417940e927` | 2026-09-26 | A confirmed relist now clears the "needs attention" failure warning straight away. |
| 0.3.1 | `ghcr.io/cryptolabsza/vast-price-manager:0.3.1@sha256:413e30b52cc3e1250c2e7f08a02e7af168446321a75be63e34718dffd6fc1c56` | 2026-09-26 | Fails loudly when Vast rejects a price: refuses a download price above $0.03, never sends a price Vast will reject, and shows Vast's reason on the dashboard. |
| 0.3.0 | `ghcr.io/cryptolabsza/vast-price-manager:0.3.0@sha256:5080eaf420f997b8943496a036c16153208fa2661cf6d25b45f89dc2a9e63984` | 2026-09-25 | First public release. Signed with cosign; see [docs/verify-image.md](docs/verify-image.md). |

This table, not the private source repository, is the authoritative
version → image mapping for anyone using VPM standalone.
