# Aient CLI

The `aient` command runs a local workspace in an isolated Aient sandbox. This
repository is the customer-facing binary distribution channel; it intentionally
does not contain the private CLI source.

The current invited-preview release is `0.1.1` for macOS and Linux on Intel and
Arm. Each release includes:

- one static `aient` archive for each supported platform;
- SHA-256 checksums;
- SPDX JSON SBOMs;
- Sigstore verification bundles; and
- SLSA provenance tied to Aient's private build workflow.

> GitHub automatically adds “Source code (zip)” and “Source code (tar.gz)” to
> every release. Those archives contain only this public documentation skeleton.
> They are not Aient CLI source, are not listed in `checksums.txt`, and are not
> verified release artifacts.

## Install on macOS or Linux

Set the release version and select the archive for your machine:

```sh
VERSION=0.1.1
case "$(uname -s)-$(uname -m)" in
  Darwin-x86_64) TARGET=darwin_amd64 ;;
  Darwin-arm64) TARGET=darwin_arm64 ;;
  Linux-x86_64) TARGET=linux_amd64 ;;
  Linux-aarch64|Linux-arm64) TARGET=linux_arm64 ;;
  *) echo "Unsupported platform: $(uname -s)-$(uname -m)" >&2; exit 1 ;;
esac

TAG="aient-cli-v${VERSION}"
ARCHIVE="aient_${VERSION}_${TARGET}.tar.gz"
BASE="https://github.com/aient-ai/aient-cli/releases/download/${TAG}"
RELEASE_DIR="$(mktemp -d)"
curl -fsSL "${BASE}/${ARCHIVE}" -o "${RELEASE_DIR}/${ARCHIVE}"
curl -fsSL "${BASE}/checksums.txt" -o "${RELEASE_DIR}/checksums.txt"
curl -fsSL "${BASE}/checksums.txt.sigstore.json" \
  -o "${RELEASE_DIR}/checksums.txt.sigstore.json"
```

`curl` is deliberate on macOS: it avoids the browser quarantine attribute that
would otherwise block this currently unnotarised preview binary. Verify the
signed checksum manifest before trusting the downloaded archive:

```sh
cd "${RELEASE_DIR}"
cosign verify-blob \
  --bundle checksums.txt.sigstore.json \
  --certificate-identity \
    "https://github.com/haf/glimt/.github/workflows/aient-cli-release.yml@refs/tags/${TAG}" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  checksums.txt

if command -v sha256sum >/dev/null 2>&1; then
  grep "  ${ARCHIVE}$" checksums.txt | sha256sum -c -
else
  grep "  ${ARCHIVE}$" checksums.txt | shasum -a 256 -c -
fi
```

Install without requiring Python, Node.js, `uv`, or a package manager:

```sh
tar -xzf "${ARCHIVE}"
mkdir -p "${HOME}/bin"
install -m 0755 aient "${HOME}/bin/.aient.new"
mv "${HOME}/bin/.aient.new" "${HOME}/bin/aient"
"${HOME}/bin/aient" version
```

Ensure `${HOME}/bin` is on `PATH`. Upgrade by repeating the download,
verification, and atomic install with a newer published version. Published
release assets are immutable; Aient fixes a bad release with a new version.

## Verify SLSA provenance

For the full build-provenance check, also download `multiple.intoto.jsonl` and
run:

```sh
curl -fsSL "${BASE}/multiple.intoto.jsonl" \
  -o "${RELEASE_DIR}/multiple.intoto.jsonl"
cd "${RELEASE_DIR}"
slsa-verifier verify-artifact "${ARCHIVE}" \
  --provenance-path multiple.intoto.jsonl \
  --source-uri github.com/haf/glimt \
  --source-tag "${TAG}"
```

The verified source and workflow identity is `haf/glimt`, while this public
repository is only the byte-for-byte release host. That distinction is
intentional; see [Security](SECURITY.md).

## Start a customer session

The default profile uses customer OAuth and never asks for an operator API key:

```sh
aient auth login
aient auth status
aient sandbox run --repository . -- go test ./...
aient auth logout
```

`auth login` opens the Aient consent flow in your browser. The current preview
supports create, sync, run, exec, shell, files, logs, and delete. Buffered
commands and shells cannot detach or reattach yet; do not treat an EOF as a
durable execution result.

For help, contact [support@aient.ai](mailto:support@aient.ai).
