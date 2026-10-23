# Install the optional local print worker

This guide is for a hosting administrator, using an unprivileged worker account. It describes staged release candidate **1.9.0**. Release URLs may not work until the release is published. The current signing key still needs independent official-channel authentication; do not bypass that requirement.

## 1. Prepare the host

Use the PHP CLI interpreter that will run the worker. It needs:

- PHP 8.0 or later, 64-bit, little-endian.
- FFI with dynamic access allowed for the worker, PCNTL, DOM, and the process-control functions checked by `preflight`, including `proc_open`.
- A supported native profile and compatible system ABI. Linux needs glibc >= 2.34, libstdc++ with `GLIBCXX_3.4.29`, and libgcc_s with `GCC_4.2.0`. Alpine/musl is unsupported by these binaries.
- Read access to the shop's WordPress files and `wp-config.php`, and the same database, encryption and storage configuration used by that shop.
- Write access to a persistent private runtime/TMP and the shop's required private storage, plus an executable filesystem for the native libraries.
- Access to the account's own `crontab` and an already running server cron service. The installer looks for `crontab` at `/usr/bin/crontab` or `/usr/sbin/crontab`.

For downloading and checking release files, use curl with TLS, OpenSSL 3, unzip, and the selected PHP CLI. WP-CLI, Node and Python are not worker dependencies. S3 file streaming, if the shop uses it, additionally needs WordPress Core's cURL transport with TLS; installation itself makes no S3 request.

FFI is required by this optional worker, not by the PHP process serving WordPress pages. The installer does not install extensions, run sudo, start a cron daemon, or change hosting security policy. If the host cannot provide these capabilities, use manual export in the Alter Product tool.

Choose a private persistent location outside both WordPress and the public document root. It must not be served by an HTTP alias. Do not use `/tmp`, container tmpfs, the plugin directory or uploads for runtime/configuration secrets. Docker deployments need a persistent volume and cron started as part of the service lifecycle.

The following paths are examples. Replace `/home/worker` with the actual account's canonical private home and replace the WordPress path and shop URL. Paths must be absolute, without symlinks or trailing slashes. Do not grant storefront PHP write access to worker sources or installation files.

## 2. Download and authenticate the release

First obtain the trusted publisher key through an independently authenticated official Alter Product channel. Save it at `/trusted/alter-print-release-key.pem`, or replace that path in the commands. The PEM beside a release or in this repository is only a convenience copy.

Prepare a new private directory as the worker account:

```bash
set -euo pipefail
umask 077
mkdir -p /home/worker/.local/share/alter-print-engine/downloads
mkdir /home/worker/.local/share/alter-print-engine/downloads/1.9.0
cd /home/worker/.local/share/alter-print-engine/downloads/1.9.0
alter_release_url='https://github.com/alterproduct/alter-product-print-engine/releases/download/v1.9.0'
for alter_asset in alter-print-engine-worker-1.9.0.zip alter-print-engine-resources-1.9.0.zip SBOM.spdx.json OPERATOR-README.txt release-manifest.json release-manifest.sig SHA256SUMS SHA256SUMS.sig; do
  curl --fail --location --proto '=https' --proto-redir '=https' --tlsv1.2 --connect-timeout 20 --max-time 600 --max-redirs 5 --output "$alter_asset" "$alter_release_url/$alter_asset"
done
```

Public release downloads do not require a GitHub token or customer account. A curl error stops this block. Do not use `-k`, `curl | php`, `curl | sh`, or execute a verifier downloaded from an unverified archive.

**Before extracting or executing anything, complete every required step in [VERIFY.md](VERIFY.md).** That guide verifies both signatures and the sizes/hashes of the four downloaded artifacts. It deliberately does not require downloading the 123.85 MB source kit for normal installation.

## 3. Extract the verified packages and run preflight

Only continue after signature and artifact checks succeeded. Create fresh source directories; do not overwrite an active installation. ZIP contents have no additional enclosing directory.

```bash
set -euo pipefail
umask 077
mkdir -p /home/worker/.local/share/alter-print-engine/worker-source
mkdir -p /home/worker/.local/share/alter-print-engine/resources
mkdir -p /home/worker/.local/share/alter-print-engine/sites
mkdir /home/worker/.local/share/alter-print-engine/worker-source/1.9.0
mkdir /home/worker/.local/share/alter-print-engine/resources/1.9.0
unzip -q alter-print-engine-worker-1.9.0.zip -d /home/worker/.local/share/alter-print-engine/worker-source/1.9.0
unzip -q alter-print-engine-resources-1.9.0.zip -d /home/worker/.local/share/alter-print-engine/resources/1.9.0
php /home/worker/.local/share/alter-print-engine/worker-source/1.9.0/install.php preflight
```

`php` must select the intended CLI interpreter; use its absolute binary path if multiple versions are installed. Expect `ok=true`, `missingCapabilities=[]`, and the correct CPU profile. This checks PHP capabilities only, not native system-library compatibility or scheduling.

The resources directory must directly contain `resource-manifest.json`, `native/` and its other original files. Keep the exact tree: do not put downloaded ZIPs, checksums, extra README files or symlinks inside it. Sources must not be writable by other users.

If PHP fails before preflight because its configured `auto_prepend_file` is missing, correct that CLI PHP configuration first. That is not an FFI result. Do not copy historical local test-container settings into a production host.

## 4. Obtain a fresh connection code and install

After packages and PHP are ready, ask the shop administrator to open **Alter Product > Settings > Print jobs and order automations > Print preparation worker > Connection / Connect worker** and generate the hosting instructions and a new connection code. The code is shown once and expires after ten minutes. Transfer it privately; never publish it, a WordPress password, database password or AWS key.

Run as the worker account. Replace all example paths, shop URL, the code and the manifest checksum. `--root` is the persistent private installed runtime, not the extracted source directory; its parent must already exist. The example `sites` parent was created above.

```bash
php /home/worker/.local/share/alter-print-engine/worker-source/1.9.0/install.php install \
  --wordpress=/var/www/shop --site=https://shop.example/ \
  --root=/home/worker/.local/share/alter-print-engine/sites/shop \
  --bundle=/home/worker/.local/share/alter-print-engine/resources/1.9.0 \
  --bundle-sha256=VERIFIED_RESOURCE_MANIFEST_SHA256 \
  --pairing-code=NEW_ONE_USE_CODE --schedule=cron
```

Use the exact `home_url` of the shop, including any subdirectory. `--bundle` points to the extracted local directory, not a ZIP or URL. Use `RESOURCE_MANIFEST_SHA256` printed by the verified manifest step; for this candidate it is `9a9e12cfc6373018047178fb411dea3416e901bb3d7fc737bab6eb4a06ddd6fa`.

Do not use `--development-package`, `--allow-root` or `--diagnostics-only` for this installation. `--schedule=manual` is a diagnostic/manual mode, not verification of an automatic scheduler; other schedulers require explicit integration.

Success reports `ok=true`, `distributionReady=true`, and `state=awaiting_scheduled_test`. The installer verifies resources, copies a versioned private runtime, runs a bounded native PNG/SVG/PDF test and adds only its marked entry to this account's crontab. It does not download code, start cron, submit a customer order, contact S3 or enable POD submission.

## 5. Verify actual scheduled operation

In WordPress choose **Check operation**, wait for the real server cron, then **Refresh status**. The installed entry runs once per minute; allow an active job to finish. Expect a recent worker contact and **Passed: PNG, SVG and PDF**.

Opening Settings/Status or requesting the test does not start an operating-system process. A manual `--scheduled` invocation does not verify the server scheduler. Empty queues are normal. Check scheduling again after a server/container restart.

File preparation and POD order submission remain separate explicit choices. You can generate a saved order's print files manually without enabling automatic preparation or supplier submission.

## Updates, rollback and retention

Updates are operator-managed. Pause preparation, let active work finish and verify a compatible new version before repeating installation with the same private `--root` and site. An unchanged live resource binding needs no new connection code; changed resources or a revoked connection need a new one. Reinstallation is designed to reuse an identical verified release and preserve unrelated crontab entries.

The worker also loads PHP from the active WordPress plugin, so coordinate plugin and worker versions. Keep the previous release for rollback. Never rewrite saved job pins or remove inputs needed by active/failed/retry work. Successful publication can retire supported per-job local inputs; original projects and the installed worker/resources remain.

See the authenticated release's `OPERATOR-README.txt` and the worker's `README.txt` for rollback/uninstall details. No instruction here authorizes an unattended external updater or WordPress.org publication.
