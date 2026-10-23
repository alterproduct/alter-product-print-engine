# Alter Product Print Engine

Optional local PHP/FFI worker for preparing PNG, SVG and PDF print files on an Alter Product for WooCommerce shop server.

**Status: staging / release candidate for 1.9.0.** Preparing this repository or a draft release does not publish release assets, authenticate the publisher key, update any shop, or establish WordPress.org approval. The independent official-channel publication of the current signing key is still pending. Do not install until you can authenticate that key independently.

## What runs where

The WordPress plugin manages saved print jobs, settings and the queue. A hosting administrator explicitly installs this separate worker under an unprivileged server account. The worker renders locally using PHP CLI and a server cron schedule; GitHub only hosts release files.

The worker is optional. **FFI is required for the worker**, together with PCNTL, DOM and process control. WordPress web PHP does not need FFI. Installing the WordPress plugin does not download or install this executable code, start operating-system processes from web requests, install extensions, or configure a server cron daemon.

Without a configured local worker, a shop owner can still open the saved design in the Alter Product tool and export print files manually. Automatic file preparation and automatic submission of an order to a POD supplier are separate opt-in settings. Installing this worker does not enable either.

## Install and verify

Read [INSTALL.md](INSTALL.md) and [VERIFY.md](VERIFY.md). Use the actual PHP CLI interpreter for the worker. WP-CLI, Node and Python are not worker dependencies.

The intended versioned release location is:

<https://github.com/alterproduct/alter-product-print-engine/releases/tag/v1.9.0>

Files use this exact base URL:

`https://github.com/alterproduct/alter-product-print-engine/releases/download/v1.9.0/`

These versioned locations may not be available while the release remains a draft. Do not substitute a `latest` download URL or assume a public key downloaded beside a ZIP is trustworthy.

## Release files and complete sources

The release is intended to provide these ten assets together:

- `alter-print-engine-worker-1.9.0.zip` — PHP operator tool.
- `alter-print-engine-resources-1.9.0.zip` — the pinned native CPU libraries and notices.
- `alter-print-engine-sources-1.9.0.tar.gz` — complete prepared source kit, dependency sources, licences and relinking/build instructions; 123,854,734 bytes for this candidate.
- `SBOM.spdx.json` — dependency and artifact inventory.
- `OPERATOR-README.txt` — operator notes included in the signed artifact inventory.
- `release-manifest.json` and `release-manifest.sig` — authenticated version, file hashes and resource-manifest pin.
- `SHA256SUMS` and `SHA256SUMS.sig` — authenticated checksum list.
- `release-public-key.pem` — convenience copy of the public key.

This small repository contains public documentation, licensing information and a convenience public key. It does not contain a private development history, shop configuration, customer data, credentials or test output. The full prepared sources are provided as the custom source-kit release asset. **GitHub's automatically generated “Source code (zip/tar.gz)” of this documentation repository is not that source kit.**

The full matching source kit must remain available beside the native release without a customer account or expiring download link. It is not installed in WordPress or copied into each print job. Fonts and project resources are captured from the shop's own authorized saved job; the native release is not a font collection.

## Supported environment and scope

The resource set contains `linux-arm64`, `linux-amd64` and `darwin-arm64` profiles. A matching CPU alone is insufficient. The Linux set requires glibc 2.34 or later, libstdc++ providing `GLIBCXX_3.4.29`, and libgcc_s providing `GCC_4.2.0` symbols. These binaries do not support Alpine/musl. See [INSTALL.md](INSTALL.md) for PHP, account, storage and scheduler requirements.

The release is not a promise that every design fits every 2 GB host. Existing queue, retry, input-retention and resource limits still apply. Saved jobs keep their original resource identities and exporter profile; this candidate does not silently migrate historical C9 library pins.

Worker maintenance is an explicit operator action. Plugin updates remain separate, and the worker also uses PHP from the active WordPress plugin. Coordinate compatible versions and repeat the scheduled export test after changes; there is no hidden external code updater.

## Licence

Unless a file explicitly states otherwise, Alter Product-authored code and documentation distributed for this project are free software under the GNU General Public License as published by the Free Software Foundation, either version 2 of the License or, at your option, any later version (`GPL-2.0-or-later`). See [LICENSE](LICENSE) and [LICENSE-NOTICE.md](LICENSE-NOTICE.md).

This grant does not relicense third-party native libraries, dependency sources or other third-party material. Their individual licence texts, copyright notices and obligations remain applicable. No GPLv2-only compatibility of the complete native set is claimed.

## Publisher-key status

The candidate key's SHA256 fingerprint over its DER-encoded public key is:

`8013c93569dfe456dc72013ad2a54a79a023ee5c83d790e5b0bc615041573b67`

This repository's statement and its PEM copy are not independent authentication of the publisher. An operator must obtain or confirm the key through an independently authenticated official Alter Product channel. That official-channel publication has not yet been completed for this staging candidate. Follow [VERIFY.md](VERIFY.md) and stop if independent confirmation is unavailable.
