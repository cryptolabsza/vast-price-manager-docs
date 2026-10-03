# Releases

Every row is a signed, tag-triggered build. Verify a signature before you pull
an image — see [docs/verify-image.md](docs/verify-image.md). Never use
`:latest`; there is no such tag on this image on purpose (see
[docs/upgrade-and-backup.md](docs/upgrade-and-backup.md)).

| Version | Pinned image reference | Published | Notes |
|---|---|---|---|
| 0.4.4 | `ghcr.io/cryptolabsza/vast-price-manager:0.4.4@sha256:51811d3388fc6142212eb6d21746fc884e482975dd218a1710d632c0d1f61dd5` | 2026-10-01 | A machine VPM cannot manage now says why and what to do, instead of a bare error code. |
| 0.4.3 | `ghcr.io/cryptolabsza/vast-price-manager:0.4.3@sha256:99a9cb17cc167d536d8a2caedbddf2c7dbef3cec0ed82c426f55472b1d99c197` | 2026-09-30 | The per-machine write guard is 5 minutes, not 1 hour, so hourly price runs no longer skip every other hour. |
| 0.4.2 | `ghcr.io/cryptolabsza/vast-price-manager:0.4.2@sha256:12fbc9e05e120e3e6b6de9f93e9f95c08b6f1d0f314038dbbac65952e7411de2` | 2026-09-30 | Shows each machine's on-demand, interruptible and idle GPUs, and a "Safe to unlist" tag. |
| 0.4.1 | `ghcr.io/cryptolabsza/vast-price-manager:0.4.1@sha256:b45cfb5df7e9f0c20f523f4fa132b30a2ee4e5ea94a52a4c0d8c79fe6ec5749b` | 2026-09-30 | Stays inside Vast's daily search quota (20,000 rows per account): one market search per GPU type per 30 minutes. **Upgrade from 0.4.0** — 0.4.0 can spend the quota by mid-morning. |
| 0.4.0 | `ghcr.io/cryptolabsza/vast-price-manager:0.4.0@sha256:20c1e3a4b66ddd723ea1e3c635e0b7dec3f8cc88fe29f4e27e54f1686954eead` | 2026-09-29 | Appliance setup: HTTPS on `8088` in the container, persistent `/data` and `/keys`, and browser-guided first run. |
| 0.3.2 | `ghcr.io/cryptolabsza/vast-price-manager:0.3.2@sha256:b99d1c88723e0182ca297725219ed65e603ed3fa939d1d8caa61b1417940e927` | 2026-09-26 | A confirmed relist now clears the "needs attention" failure warning straight away. |
| 0.3.1 | `ghcr.io/cryptolabsza/vast-price-manager:0.3.1@sha256:413e30b52cc3e1250c2e7f08a02e7af168446321a75be63e34718dffd6fc1c56` | 2026-09-26 | Fails loudly when Vast rejects a price: refuses a download price above $0.03, never sends a price Vast will reject, and shows Vast's reason on the dashboard. |
| 0.3.0 | `ghcr.io/cryptolabsza/vast-price-manager:0.3.0@sha256:5080eaf420f997b8943496a036c16153208fa2661cf6d25b45f89dc2a9e63984` | 2026-09-25 | First public release. Signed with cosign; see [docs/verify-image.md](docs/verify-image.md). |

This table, not the private source repository, is the authoritative
version → image mapping for anyone using VPM standalone.
