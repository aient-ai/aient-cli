# Aient CLI

The `aient` command runs a local workspace in an isolated Aient sandbox. This
repository is the customer-facing binary distribution channel; it intentionally
does not contain the private CLI source.

The current invited-preview release is `0.8.0` for macOS and Linux on Intel and
Arm. Guidance for `0.8.1` is staged below so it can be reviewed with the release
candidate. Until `aient-cli-v0.8.1` is published, treat those sections as
pending and use the installed binary's `--help` as the authority.

Each published release includes:

- one static `aient` archive for each supported platform;
- SHA-256 checksums;
- SPDX JSON SBOMs;
- Sigstore verification bundles; and
- SLSA provenance tied to Aient's private build workflow.

Git 2.45 or newer enables retained-workspace incremental synchronization for
partial/promisor clones without lazy object fetching. Ordinary repositories
also use incremental synchronization on older Git clients; an older partial
clone uses the complete-snapshot transfer path.

> GitHub automatically adds “Source code (zip)” and “Source code (tar.gz)” to
> every release. Those archives contain only this public documentation skeleton.
> They are not Aient CLI source, are not listed in `checksums.txt`, and are not
> verified release artifacts.

## Install on macOS or Linux

Set the published release version and select the archive for your machine:

```sh
VERSION=0.8.0
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
RELEASE_DIR="$(mktemp -d)" || exit 1
cleanup_release() {
  rm -rf -- "${RELEASE_DIR}"
}
trap cleanup_release EXIT HUP INT TERM

curl -fsSL "${BASE}/${ARCHIVE}" \
  -o "${RELEASE_DIR}/${ARCHIVE}" || exit 1
curl -fsSL "${BASE}/checksums.txt" \
  -o "${RELEASE_DIR}/checksums.txt" || exit 1
curl -fsSL "${BASE}/checksums.txt.sigstore.json" \
  -o "${RELEASE_DIR}/checksums.txt.sigstore.json" || exit 1
```

`curl` is deliberate on macOS: it avoids the browser quarantine attribute that
would otherwise block this currently unnotarised preview binary. Verify the
signed checksum manifest, require exactly one well-formed record for the
selected archive, and then verify that record:

```sh
cd "${RELEASE_DIR}" || exit 1
cosign verify-blob \
  --bundle checksums.txt.sigstore.json \
  --certificate-identity \
    "https://github.com/haf/glimt/.github/workflows/aient-cli-release.yml@refs/tags/${TAG}" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  checksums.txt || exit 1

awk -v archive="${ARCHIVE}" \
  'NF == 2 && $2 == archive && length($1) == 64 && $1 ~ /^[[:xdigit:]]+$/ { print }' \
  checksums.txt > archive.checksum || exit 1
[ "$(wc -l < archive.checksum | tr -d '[:space:]')" = "1" ] || {
  echo "Expected exactly one checksum for ${ARCHIVE}" >&2
  exit 1
}

if command -v sha256sum >/dev/null 2>&1; then
  sha256sum -c archive.checksum || exit 1
else
  shasum -a 256 -c archive.checksum || exit 1
fi
```

The signed release archives contain exactly one regular file named `aient`.
Reject unexpected paths, extra entries, and symlinks before extraction:

```sh
tar -tzf "${ARCHIVE}" > archive.members || exit 1
[ "$(wc -l < archive.members | tr -d '[:space:]')" = "1" ] &&
  [ "$(sed -n '1p' archive.members)" = "aient" ] || {
    echo "Unexpected archive contents" >&2
    exit 1
  }

tar -tvzf "${ARCHIVE}" > archive.details || exit 1
[ "$(sed -n '1{s/^\(.\).*/\1/p;}' archive.details)" = "-" ] || {
  echo "The aient archive entry is not a regular file" >&2
  exit 1
}

mkdir extract || exit 1
tar -xzf "${ARCHIVE}" -C extract aient || exit 1
[ -f extract/aient ] && [ ! -L extract/aient ] || {
  echo "The extracted aient binary is not a regular file" >&2
  exit 1
}
```

Install atomically without executing the downloaded candidate before it reaches
the final path:

```sh
mkdir -p "${HOME}/bin" || exit 1
install -m 0755 extract/aient "${HOME}/bin/.aient.new" || exit 1
mv -f "${HOME}/bin/.aient.new" "${HOME}/bin/aient" || exit 1
"${HOME}/bin/aient" version || exit 1
```

Ensure `${HOME}/bin` is on `PATH`. Upgrade by repeating the download,
verification, archive inspection, and atomic install with a newer published
version. Published release assets are immutable; Aient fixes a bad release
with a new version.

## Install the agent skill

Install the customer CLI workflow skill in the current agent project:

```sh
npx skills add aient-ai/aient-cli --skill use-aient-cli
```

Install it globally for Codex:

```sh
npx skills add aient-ai/aient-cli \
  --skill use-aient-cli \
  --agent codex \
  --global \
  --yes
```

The skill covers customer OAuth profiles, folder-specific project selection,
exact workspace synchronization, retained execution recovery, environment
secrets, and current public command boundaries. Native Aient Harness agents
already have sandbox tools and should not create nested CLI sandboxes.

## Verify SLSA provenance

For the full build-provenance check, also download `multiple.intoto.jsonl` and
run:

```sh
curl -fsSL "${BASE}/multiple.intoto.jsonl" \
  -o "${RELEASE_DIR}/multiple.intoto.jsonl" || exit 1
cd "${RELEASE_DIR}" || exit 1
slsa-verifier verify-artifact "${ARCHIVE}" \
  --provenance-path multiple.intoto.jsonl \
  --source-uri github.com/haf/glimt \
  --source-tag "${TAG}" || exit 1
```

The verified source and workflow identity is `haf/glimt`, while this public
repository is only the byte-for-byte release host. That distinction is
intentional; see [Security](SECURITY.md).

## Start a customer session

The default profile uses customer OAuth and never asks for an operator API key.
For a disposable workspace and command:

```sh
aient auth login
aient auth status
aient sandbox run \
  --environment development \
  --size medium \
  -- go test ./...
aient auth logout
```

Omit `--size` to use the environment's configured default. The allowed customer
presets are `small`, `medium`, and `large`; the environment policy decides which
are available. Raw template, CPU, memory, metadata, and named-volume flags
shown in shared help are operator-only. Customers cannot choose scheduler or
storage fields directly.

For a checkout that must remain available for later commands, retain the
environment-bound sandbox, preserve the printed execution UUID, and always
delete the sandbox explicitly. The lifecycle-read commands in this example are
staged for `0.8.1`:

```sh
SANDBOX="retained-$(date +%Y%m%d%H%M%S)-$$"
cleanup_sandbox() {
  aient sandbox delete "${SANDBOX}" >/dev/null 2>&1 || true
}
trap cleanup_sandbox EXIT

aient sandbox run \
  --name "${SANDBOX}" \
  --keep \
  --environment development \
  -- true || exit 1

aient sandbox status "${SANDBOX}" --environment development
aient sandbox wait "${SANDBOX}" --environment development
aient --timeout 15m sandbox exec "${SANDBOX}" \
  --environment development \
  --workdir /workspace/repo \
  -- go test ./...
aient sandbox logs "${SANDBOX}" \
  --environment development \
  --tail-lines 200

aient sandbox delete "${SANDBOX}" || exit 1
trap - EXIT
```

In staged `0.8.1`, the lifecycle reads `status`, `wait`, and `logs` require the
same `--environment` context for an environment-bound sandbox:

- `sandbox status` returns the current sandbox lifecycle snapshot.
- `sandbox wait` polls until the sandbox is ready.
- `sandbox logs` returns a bounded recent sandbox log snapshot; it is not
  retained command stdout/stderr.
- `sandbox execution wait SANDBOX EXECUTION` waits for one supervised command's
  content-free terminal outcome.

Use `sandbox execution attach|status|wait|cancel` with the execution UUID after
transport loss; do not replay the command. Repeat the original `--environment`
and any `--repository` selector for those execution operations.

Active foreground operations automatically receive bounded lease protection.
This prevents a healthy upload, command, shell, download, or bootstrap from
being reclaimed merely because its initial lease is short. It does not lengthen
the default lease, cross the server-owned hard expiry, survive client loss
indefinitely, or make `--keep` permanent. Delete retained sandboxes when done.

## Select identity and secrets

Use one named profile per organisation and a nearest-ancestor
`.aient/config.yaml` per checkout or worktree. This keeps the selected customer
context stable without putting credentials in the project:

```sh
aient --profile acme auth login
aient --profile acme auth status
aient auth profiles default acme
```

Organisation owners and administrators can set or replace an encrypted
environment secret after consenting to the dedicated write capability:

```sh
printf '%s' "${STRIPE_SECRET_KEY}" |
  aient --profile acme environment secrets set \
    --environment development \
    STRIPE_SECRET_KEY \
    --stdin
```

Prefer `--stdin`; a positional secret can leak through shell history or process
inspection. An older login may need browser re-consent for
`aient.environment.secrets.write`. Setting a secret neither enables customer
CLI workloads nor makes that secret sandbox-exportable; environment capability
policy controls export separately. Secret values are never returned by the
command.

## Select files deliberately

With no `--include` or `--exclude`, Git synchronization transfers exact tracked
state and no non-Git files. In staged `0.8.1`, an `--exclude` without any
`--include` first selects **all non-Git paths**, including ignored files, then
subtracts the exclusions. This broad mode can upload `.env`, `.npmrc`, cloud
credentials, access-token files, and other local secrets.

Prefer narrow, repeatable `--include` globs. Use exclude-only selection only
after auditing the complete non-Git tree and explicitly excluding every
credential source. Exclusions never remove tracked Git files. Keep portable
access-token files outside the uploaded workspace and never upload them, a hard
link to them, profile refresh state, or an operating-system credential store.

`auth login` opens the Aient consent flow in your browser. An administrator
must first enable customer CLI development on the selected environment. A
configured Git remote is only a repository selector; the server verifies
authority. For help, contact
[support@aient.ai](mailto:support@aient.ai).
