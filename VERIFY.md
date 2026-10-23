# Authenticate release 1.9.0 before execution

This guide verifies files; it does not execute code from the downloaded worker or native libraries. Use your already installed system OpenSSL 3 and PHP CLI. The commands assume the fresh download directory from [INSTALL.md](INSTALL.md).

**Current staging limitation:** independent official-channel authentication of this candidate signing key is not yet published. Stop until you can obtain or confirm the key through an independently authenticated official Alter Product channel. A PEM and matching fingerprint delivered together in this repository, the release or an unsigned message do not establish publisher identity.

## 1. Authenticate the public key

Keep the independently authenticated key at `/trusted/alter-print-release-key.pem`, or replace that path below. If you received an independently confirmed fingerprint, check the key's DER fingerprint:

```bash
set -euo pipefail
openssl pkey -pubin -in /trusted/alter-print-release-key.pem -outform DER -out release-key.der
openssl dgst -sha256 release-key.der
```

Compare the output exactly with the independently confirmed official fingerprint. The staged candidate key is Ed25519, with DER SHA256:

`8013c93569dfe456dc72013ad2a54a79a023ee5c83d790e5b0bc615041573b67`

This is a hash of the DER public key, not of the PEM text. A mismatch, an unexpected key change, or missing independent confirmation means stop. Do not trust the `signing` field of an unsigned manifest as an authority for a new key.

## 2. Verify both detached signatures

```bash
set -euo pipefail
openssl pkeyutl -verify -pubin -inkey /trusted/alter-print-release-key.pem -rawin -in release-manifest.json -sigfile release-manifest.sig
openssl pkeyutl -verify -pubin -inkey /trusted/alter-print-release-key.pem -rawin -in SHA256SUMS -sigfile SHA256SUMS.sig
```

Both commands must succeed. Ed25519 uses raw input; do not add a `-digest` option. Do not extract or execute the worker if either verification fails.

## 3. Verify the installation artifacts against the authenticated manifest

This is locally supplied verification code for the existing system PHP interpreter. It parses authenticated JSON and hashes files; it does not load anything from the worker ZIP. `hash_file` streams file contents rather than loading the resource ZIP into PHP memory.

```bash
set -euo pipefail
php -r '
$m = json_decode(file_get_contents("release-manifest.json"), true, 64, JSON_THROW_ON_ERROR);
if (($m["schema"] ?? "") !== "alter.print-engine.public-release.v1" || ($m["version"] ?? "") !== "1.9.0" || !is_array($m["artifacts"] ?? null)) {
    fwrite(STDERR, "Wrong release manifest\n"); exit(1);
}
$artifacts = [];
foreach ($m["artifacts"] as $a) {
    if (!is_array($a) || !is_string($a["file"] ?? null) || isset($artifacts[$a["file"]])) {
        fwrite(STDERR, "Invalid or duplicate artifact entry\n"); exit(1);
    }
    $artifacts[$a["file"]] = $a;
}
foreach (["alter-print-engine-worker-1.9.0.zip", "alter-print-engine-resources-1.9.0.zip", "SBOM.spdx.json", "OPERATOR-README.txt"] as $name) {
    $a = $artifacts[$name] ?? null;
    if (!is_array($a) || !is_int($a["bytes"] ?? null) || $a["bytes"] < 1 || !is_string($a["sha256"] ?? null) || preg_match("/^[a-f0-9]{64}$/D", $a["sha256"]) !== 1 || !is_file($name) || filesize($name) !== $a["bytes"] || !hash_equals($a["sha256"], hash_file("sha256", $name))) {
        fwrite(STDERR, "Artifact verification failed: " . $name . "\n"); exit(1);
    }
    echo "VERIFIED ", $name, PHP_EOL;
}
if (!is_string($m["resourceManifestSha256"] ?? null) || preg_match("/^[a-f0-9]{64}$/D", $m["resourceManifestSha256"]) !== 1) {
    fwrite(STDERR, "Invalid resource manifest pin\n"); exit(1);
}
echo "RESOURCE_MANIFEST_SHA256=", $m["resourceManifestSha256"], PHP_EOL;
'
```

Expect four `VERIFIED` lines and, for this candidate:

`RESOURCE_MANIFEST_SHA256=9a9e12cfc6373018047178fb411dea3416e901bb3d7fc737bab6eb4a06ddd6fa`

Only now continue to extraction and preflight in [INSTALL.md](INSTALL.md). Never substitute a hash calculated from an unauthenticated resource manifest for the authenticated `resourceManifestSha256`.

## Optional complete source-kit verification

The installer needs the two ZIPs, not the source archive. The complete source kit remains a required matching public release asset for source/licence availability and for people inspecting, modifying or relinking the software.

To download it for a full audit after the release is public:

```bash
set -euo pipefail
curl --fail --location --proto '=https' --proto-redir '=https' --tlsv1.2 --connect-timeout 20 --max-time 600 --max-redirs 5 --output alter-print-engine-sources-1.9.0.tar.gz https://github.com/alterproduct/alter-product-print-engine/releases/download/v1.9.0/alter-print-engine-sources-1.9.0.tar.gz
sha256sum -c SHA256SUMS
```

On macOS use `shasum -a 256 -c SHA256SUMS` instead. All five entries must report `OK`. The checksum list was already authenticated by its detached signature above.

`SHA256SUMS` includes worker, resources, source kit, SBOM and operator README. Running its full check after downloading only the installation subset will fail because the source archive is absent; that alone does not mean the verified ZIP is corrupt. Do not use a blanket “ignore missing” option to conceal missing required installation files. The manifest procedure above explicitly checks each required artifact and fails if it is missing.

GitHub's automatic “Source code (zip/tar.gz)” archive is a snapshot of this documentation repository and is not a substitute for `alter-print-engine-sources-1.9.0.tar.gz`.

## Verification tools and release identity

Use the exact `v1.9.0` asset URLs, not `releases/latest/download`. Intended publication uses immutable release assets and a fixed tag; a new build requires a new version, hashes and signatures. Do not assume immutability has been enabled merely because a URL has a version in it.

Upstream tool references: [OpenSSL 3 pkeyutl](https://docs.openssl.org/3.0/man1/openssl-pkeyutl/) and [GitHub immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases).
