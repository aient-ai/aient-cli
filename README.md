# Aient CLI

The `aient` command runs a local workspace in an isolated Aient sandbox. This
repository is the customer-facing binary distribution channel; it intentionally
does not contain the private CLI source.

The current release is `0.11.5` for macOS and Linux on Intel and
Arm. Each release includes:

- one static `aient` archive for each supported platform;
- SHA-256 checksums;
- SPDX JSON SBOMs;
- Sigstore verification bundles; and
- SLSA provenance tied to Aient's private build workflow.

Workspace synchronization requires a complete, self-contained local Git
repository. Shallow, partial/promisor, alternate-backed, grafted, or
replace-influenced sources are rejected before upload; obtain a standalone
complete clone first. Compatible retained workspaces still use incremental
synchronization when their remote base is independently certified complete.

Named sandbox creation is replay-safe. The CLI binds one public sandbox name
to one idempotency identity, retries only that same identity across bounded
transport ambiguity, and refuses to execute in or delete a sandbox when the
create response cannot be trusted as the requested name.

Organisation selection is UUID-bound and human-readable. The CLI resolves the
authenticated display name from Aient and shows it beside the canonical
organisation UUID, while treating that display name as render-only. Project,
profile, access-token, refreshed-claim, and authenticated server context UUIDs
must agree. A legacy project name fails before the requested operation and
returns trusted migration guidance with the resolved name and UUID.

Environment-bound sandbox operations carry one authenticated
organisation/environment/repository tuple across create, synchronization,
files, execution, observation, lease refresh, and deletion. New customer
workspace operations retain that tuple in `workspace-operation-bundle-v2` and
resume with the exact authority instead of weakening to owner-only access.
Legacy V1 receipts remain readable under their established compatibility rules.

The conservative supervised execution timeout ceiling is `23h56m18s`.
`aient sandbox exec` rejects larger command budgets locally before contacting
the sandbox service. An upgraded server also rejects unsupported values before
claim creation or provider dispatch and returns the typed
`sandbox_execution_timeout_invalid` boundary. A valid command budget can still
conflict when an older retained sandbox has insufficient remaining hard lease.

`aient auth exec --profile NAME -- COMMAND...` gives a trusted host wrapper one
renewable, access-only Aient credential. The launcher injects or copies no
refresh token, named profile, credential-store entry, shared OAuth lock, or
operator credential into the command. This is credential minimization, not a
local process sandbox: untrusted same-UID descendants require an external
confinement boundary that denies ambient host credentials and grants only the
exact task credential file.

From a configured Aient Git project, `aient auth login` establishes bounded
project access by default. On macOS, the refresh credential remains in Keychain
and a launchd user service rotates only the repository-private, 15-minute access
envelope for up to 90 days. Login returns only after manager-owned renewal is
verified. No foreground terminal or PID owns that renewal. Use
`--project-access=none` to keep only the host login and remove managed access
for the nearest project; use
`aient --profile NAME auth export --store-in-project` to repair or reconcile an
existing managed installation.

On macOS, the protected directory birth time lets the same installation survive
an APFS device-number change across a remount while directory replacement still
fails closed. If a legacy definition predates that evidence, a healthy host
profile can replace it with `--store-in-project` without another login.

Release 0.11.5 retains Agent Thread discovery plus the existing status, events,
messaging, interaction, cancellation, chat, and execution surfaces:

```sh
PROFILE=aient
aient --profile "$PROFILE" agent list --search QUERY --status active --limit 20
aient --profile "$PROFILE" agent list --include-archived --cursor CURSOR --json
```

An exact Thread ID or title query can be combined with status, archive, page
size, and cursor filters. Human pagination prints a continuation command that
preserves the active filters; `--json` emits one versioned page.

Retained workspace-operation storage under `~/.aient/workspace-operations` is
also bounded. Inspect it locally, without authentication or network access:

```sh
aient sandbox workspace list [--json]
aient sandbox workspace resume OPERATION
aient sandbox workspace discard OPERATION
```

New whole-bundle publication is limited to an aggregate 10 GiB and must leave
at least 1 GiB of filesystem space after copying. There is no count limit, no
configuration override, and the CLI does not evict retained operations
automatically. Existing receipts remain available for explicit recovery. This
does not change broad non-Git selection or temporary staging; keep using precise
`--include` and `--exclude` rules for selected local content.

Every customer-development create advertises
`developmentSizeBindingVersion: v2`. A compatible product response binds the
complete `aientSizeBinding=v2` class, source, and envelope before the
control-plane request. Partial, crossed, or mismatched bindings fail closed;
legacy explicit V1 and an old server's unbound environment default remain
rollout-only compatibility paths.

Use `aient sandbox run --timing-json` to emit one content-free JSON line on
stderr without changing command output or exit status. It records proven
workspace activation, first output, terminal status, downloads, and confirmed
cleanup or retention. A failed lease restore remains the primary error and
reports `retained=true` with `cleanupConfirmed=false`.

Disposable cleanup issues exactly one DELETE with the full configured
`--timeout`, followed by a fresh equally bounded GET-only absence check. The
CLI does not retry DELETE or infer success from ambiguous deletion. Stopping a
lease heartbeat is quiet; genuine lease-refresh failures continue to warn and
retry while the operation remains active.

Selected submodules can activate with declared parent-repository extras. The
server verifies every declared path and the atomic root exchange before publish.
Capability preflight also recognizes the exact old helper generation mounted by
an already-running sandbox, preserving retained-operation recovery during a
mixed-version rollout.

The standard `aient-agent` and retained `aient-pr-e2e` catalogs currently use
the V2 workspace writer. The server keeps the closed V2 and V3 readers and
negotiates capabilities per template; the CLI does not infer one global
workspace layout version.

Linux managed renewal remains preview pending a real systemd-user and D-Bus
Secret Service renewal, restart, and logout canary. Use
`--project-access=none` for host-only login when you do not explicitly want to
opt into that preview.

> GitHub automatically adds “Source code (zip)” and “Source code (tar.gz)” to
> every release. Those archives contain only this public documentation skeleton.
> They are not Aient CLI source, are not listed in `checksums.txt`, and are not
> verified release artifacts.

## Install on macOS or Linux

Set the release version and select the archive for your machine:

```sh
VERSION=0.11.5
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
would otherwise block this currently unnotarised binary. Verify the
signed checksum manifest before trusting the downloaded archive:

```sh
cd "${RELEASE_DIR}"
cosign verify-blob \
  --bundle checksums.txt.sigstore.json \
  --certificate-identity \
    "https://github.com/haf/glimt/.github/workflows/aient-cli-release.yml@refs/tags/${TAG}" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  checksums.txt

awk -v archive="${ARCHIVE}" \
  '$2 == archive && NF == 2 && length($1) == 64 && $1 !~ /[^0-9a-f]/ { print }' \
  checksums.txt > "${ARCHIVE}.sha256"
if [ "$(wc -l < "${ARCHIVE}.sha256" | tr -d ' ')" -ne 1 ]; then
  echo "Expected exactly one checksum for ${ARCHIVE}" >&2
  exit 1
fi

if command -v sha256sum >/dev/null 2>&1; then
  sha256sum -c "${ARCHIVE}.sha256"
else
  shasum -a 256 -c "${ARCHIVE}.sha256"
fi
```

Install without requiring Python, Node.js, `uv`, or a package manager:

```sh
if [ "$(tar -tzf "${ARCHIVE}")" != "aient" ]; then
  echo "Archive must contain exactly one path named aient" >&2
  exit 1
fi
if [ "$(tar -tvzf "${ARCHIVE}" | awk 'NR == 1 { print substr($1, 1, 1) }')" != "-" ]; then
  echo "Archive entry aient must be a regular file" >&2
  exit 1
fi
STAGE_DIR="$(mktemp -d)"
tar -xzf "${ARCHIVE}" -C "${STAGE_DIR}"
mkdir -p "${HOME}/bin"
install -m 0755 "${STAGE_DIR}/aient" "${HOME}/bin/.aient.new"
mv "${HOME}/bin/.aient.new" "${HOME}/bin/aient"
"${HOME}/bin/aient" version
```

The checksum selection must produce exactly one manifest entry, and the archive
must contain exactly one regular file named `aient`; do not weaken either
check. Ensure `${HOME}/bin` is on `PATH`. Upgrade by repeating the download,
verification, and atomic install with a newer published version. Published
release assets are immutable; Aient fixes a bad release with a new version.

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

The skill covers customer OAuth profiles, project selection, exact workspace
synchronization, retained execution recovery, and current public command
boundaries. Native Aient Harness agents already have sandbox tools and should
not create nested CLI sandboxes.

## Verify SLSA provenance

For the full build-provenance check, also download `multiple.intoto.jsonl` and
use a current [GitHub CLI](https://cli.github.com/) to verify the Sigstore
signature, Rekor entry, exact workflow certificate, artifact digest, and source
identity. The source digest below is the peeled private tag commit for 0.11.5:

```sh
curl -fsSL "${BASE}/multiple.intoto.jsonl" \
  -o "${RELEASE_DIR}/multiple.intoto.jsonl"
cd "${RELEASE_DIR}"
SOURCE_DIGEST="2eb7ae09be5adf0105ff2865f90c902cbce14dac"
SIGNER_DIGEST="cdab76e75fef610b59a7f528a6dba359e624d6af"
gh attestation verify "${ARCHIVE}" \
  --bundle multiple.intoto.jsonl \
  --repo haf/glimt \
  --predicate-type https://slsa.dev/provenance/v0.2 \
  --cert-identity \
    "https://github.com/haf/glimt/.github/workflows/aient-cli-provenance.yml@refs/tags/v1.0.1" \
  --cert-oidc-issuer https://token.actions.githubusercontent.com \
  --source-ref "refs/tags/${TAG}" \
  --source-digest "${SOURCE_DIGEST}" \
  --signer-digest "${SIGNER_DIGEST}" \
  --format json > provenance-verification.json

jq -e --arg source "git+https://github.com/haf/glimt@refs/tags/${TAG}" \
  --arg sha "${SOURCE_DIGEST}" '
    length == 1 and
    .[0].verificationResult.statement._type ==
      "https://in-toto.io/Statement/v0.1" and
    .[0].verificationResult.statement.predicateType ==
      "https://slsa.dev/provenance/v0.2" and
    (.[0].verificationResult.statement.subject | length) == 18 and
    .[0].verificationResult.statement.predicate.builder.id ==
      "https://github.com/haf/glimt/.github/workflows/aient-cli-provenance.yml@refs/tags/v1.0.1" and
    .[0].verificationResult.statement.predicate.buildType ==
      "https://github.com/slsa-framework/slsa-github-generator/generic@v1" and
    .[0].verificationResult.statement.predicate.invocation.configSource == {
      uri: $source,
      digest: {sha1: $sha},
      entryPoint: ".github/workflows/aient-cli-release.yml"
    }
  ' provenance-verification.json > /dev/null
```

The verified source is `haf/glimt` and the provenance builder is the immutable
repository-owned `aient-cli-provenance.yml@v1.0.1` workflow, while this public
repository is only the byte-for-byte release host. That distinction is
intentional. `gh attestation verify` performs the cryptographic and identity
verification; the final exact predicate check is not a replacement for it. See
[Security](SECURITY.md).

## Start a customer session

Use a named customer OAuth profile; this never asks for an operator API key.
From a configured project, select the refresh-backed customer OAuth profile
explicitly with `aient --profile NAME agent ...`. Ambient project access is
sandbox-scoped and intentionally does not grant Agent conversation authority.

```sh
PROFILE=aient
aient auth login --profile "$PROFILE"
aient --profile "$PROFILE" auth status
aient --profile "$PROFILE" agent list --search QUERY --status active --limit 20
aient --profile "$PROFILE" agent status THREAD [--json]
aient --profile "$PROFILE" agent events THREAD [--follow] [--json]
aient --profile "$PROFILE" agent message THREAD MESSAGE... [--operation-id UUID] [--immediate] [--json]
aient --profile "$PROFILE" agent respond THREAD INTERACTION ANSWER... [--operation-id UUID] [--json]
aient --profile "$PROFILE" agent cancel THREAD [--operation-id UUID] [--json]
aient --profile "$PROFILE" agent chat THREAD
aient --profile "$PROFILE" agent execution status EXECUTION [--json]
aient --profile "$PROFILE" agent execution attach EXECUTION
aient --profile "$PROFILE" agent execution wait EXECUTION [--json]
aient --profile "$PROFILE" agent execution cancel EXECUTION [--json]
aient sandbox run --environment development --size medium -- go test ./...

aient sandbox run --environment development --repository owner/repository \
  --size medium \
  --name retained-sandbox --keep -- true
aient sandbox status retained-sandbox \
  --environment development --repository owner/repository
aient --timeout 15m sandbox exec retained-sandbox \
  --environment development --repository owner/repository \
  --workdir /workspace/repo \
  -- go test ./...
aient sandbox logs retained-sandbox \
  --environment development --repository owner/repository
aient sandbox delete retained-sandbox \
  --environment development --repository owner/repository
aient --profile "$PROFILE" auth logout
```

`auth login` opens the Aient consent flow in your browser. An administrator
must first enable customer CLI development on the selected environment. The
Agent read surface requires `aient.agent.read`; messages, answers, explicit
cancellation, and interactive mutations require `aient.agent.write`.
`status --json` emits one deterministic document, while `events --follow
--json` emits JSONL that can resume from a durable sequence.

`agent chat` attaches to the same public API lifecycle. Enter `! COMMAND` to
run a supervised shell command under `/workspace/repo` in the environment and
sandbox frozen from the latest rendered Agent status. The CLI shows those IDs
and its preallocated execution UUID before output, sends no workstation process
environment or local repository inference, and sends no Agent message for the
command. Reconnect by passing the exact `EXECUTION` identifier to the execution
commands above. Ctrl-C in ordinary chat detaches without cancelling Agent work; Ctrl-C in an active
remote-command view explicitly requests cancellation of that exact execution.

This release does not start a new Agent Thread. Use `agent list` to find a
reconnectable Thread ID, or copy it from another authoritative Aient surface. The
`--immediate` requires one explicit stable operation UUID for one logical
message; an ordinary `agent message` may omit it and let the CLI generate one.
Reuse an explicit UUID only for an exact retry. `--immediate` asks the active task
mailbox to accept text at a stable boundary without cancellation or root
replacement. The result is `steered`, `started`, `queued`, or `duplicate` and
includes the applicable task and steering-message IDs. An incompatible active
task origin or attachment plan fails with
`cooperative_steering_scope_mismatch` instead of reporting acceptance.

The current release supports repository-independent and verified-repository
development, including brokered environment/GitHub capabilities for
`sandbox run` and retained `sandbox exec`. It may infer a repository selector
from local Git remotes, but the server verifies authority; a configured remote
never grants access. Retained
`sandbox exec` prints an execution UUID, streams live output, and can reconcile
transport loss through `sandbox execution attach|status|wait|cancel` without
replaying the command. Use one active terminal per execution: a second live
attachment supersedes the first, and output received by either attachment is
not replayed to the other. Use `sandbox exec --detach` for an existing retained
sandbox, or `sandbox run --detach --keep` for a newly composed run. The CLI
prints one stable execution ID after detached ownership is accepted; later
status, attach, wait, or cancel calls observe that same execution without
redispatch. Foreground `sandbox run` is also supervised and streams stdout and
stderr live. Foreground execution remains connection-bound: after transport
loss it must be reattached within the short server-owned reconnect grace or the
service cancels it. Use `--detach` when command survival must not depend on a
client attachment. Each `sandbox shell` still opens a fresh non-resumable PTY without
brokered environment/GitHub capabilities.

Inspect or update explicit runtime-service repository mappings with
`aient services list`, `services paths`, `services classify`, and
dry-run-by-default `services apply`. These commands require the
`services:read` and `services:manage` OAuth scopes. Existing sandbox access
remains valid; an older profile may need one interactive login before service
management.

To print the one-time OAuth URL instead of opening the configured browser, run
`aient auth login --print-url` and paste it into the intended browser while the
same command waits. From a configured project this also installs and verifies
managed project access. After it returns, ordinary repository commands use the
short-lived project envelope without `--profile`, token flags, copied refresh
credentials, a foreground exporter, or PID tracking.

Environment/repository-bound retained calls must repeat both selectors;
repository-independent calls repeat only `--environment`. Without the matching
selectors, the owner-only client deliberately cannot see the bound sandbox.
Customer compute uses the public `small`, `medium`,
and `large` size classes. Omit `--size` to use the environment default; the
server owns the underlying template and resource envelope.

To synchronize ignored and untracked content broadly, supply exclusions
without includes:

```sh
aient sandbox run --environment development \
  --exclude '**/node_modules/**' \
  --exclude '.local/**' \
  --exclude '**/.env*' \
  -- go test ./...
```

Exclude-only selection starts with every eligible non-Git path, omits regular
macOS AppleDouble `._*` files, and then subtracts the patterns. That documented
filter applies only to regular files selected recursively: Git-tracked paths
and explicitly named single-file uploads remain exact, and the local source is
unchanged. Selection never replaces Git-tracked content or includes `.git`
metadata. Beyond the AppleDouble filter, it has no implicit cache or secret
exclusions. Review the selected scope and explicitly exclude `.env` files,
credentials, tokens, private keys, and other secret-bearing non-Git paths
before uploading.

When the server advertises explicit repository path ownership, generated,
dependency, and cache directories that are not selected by Git or the explicit
file filters are absent from a new candidate. The CLI no longer infers those
directories as preserved workspace mount roots in that negotiated mode, so
package-owned cleanup such as `rm -rf packages/otel/dist` can remove and
recreate them normally. Older servers retain their legacy preservation
behavior.

To retain an exact ordinary remote directory across synchronization, select it
explicitly:

```sh
aient sandbox sync retained-sandbox . \
  --environment development --repository owner/repository \
  --preserve-remote-dir .cache/tool
```

`--preserve-remote-dir` is repeatable and accepts exact repository-relative
non-Git directories. The path must already be an ordinary remote directory;
it cannot overlap Git content, selected uploads, reset paths, symlinks, files,
or another preservation root. Use `--reset-remote-dir` when the intended result
is an ordinary empty directory instead:

```sh
aient sandbox sync retained-sandbox . \
  --environment development --repository owner/repository \
  --reset-remote-dir packages/otel/dist
```

Both options are locally bounded and fail before upload when their ownership
would be ambiguous. A server that does not advertise explicit path ownership
continues to support ordinary synchronization, but rejects explicit
preservation rather than silently changing its meaning.

For `/workspace/repo`, release `0.9.0` negotiates the typed V2 activation
protocol with an attested server. Git state and explicit files are uploaded as
immutable layers under one candidate revision, verified, assembled, and then
atomically activated. A failure before activation preserves the previous live
workspace. Older or unattested servers remain on the compatible legacy route.
Once a sandbox has migrated to V2, synchronization is fixed at
`/workspace/repo`; custom `--remote-dir` mutations are rejected rather than
downgraded.

Repositories with initialized submodules remain rejected by default. When a
build deliberately needs the parent repository without checked-out child
content, opt into the incomplete Gitlink-only representation explicitly:

```sh
aient sandbox run --environment development \
  --submodules=gitlinks \
  -- go test ./...
```

The CLI requires every initialized child and initialized descendant to be
clean and checked out at the exact object recorded by its parent index. It then
omits child bytes and binds the complete sorted parent-index Gitlink set into
the V2 or V3 activation manifest. Dirty, untracked, mismatched, malformed, or
ambiguous child state fails locally before sandbox creation or synchronization
requests. This mode is not recursive submodule transfer. When a command
consumes exact child repository content, select only that explicit dependency
closure:

```sh
aient sandbox run --environment development \
  --submodules=selected \
  --submodule apps/aient-docs \
  -- pnpm --filter aient-app build
```

Selected children must be initialized, clean, complete-history repositories at
their exact parent-index Gitlink OIDs. Repeat `--submodule` for every selected
ancestor; selection never expands recursively. Initialized but unselected
children contribute no bytes and remain empty Gitlink boundaries remotely.
Legacy or retained readers without exact selected-submodule support reject the
operation instead of silently broadening or weakening the transfer policy.

Default snapshots do not emit that explicit Gitlink manifest. They preserve
uninitialized Gitlinks as exact raw Git index entries, so their path bytes and
total count are not subject to the manifest-only UTF-8, literal-backslash, or
512-entry wire rules. The explicit `--submodules=gitlinks` mode continues to
apply all three checks locally and fail before any remote request.

Active operations are protected from a stale idle or timeout cleanup decision:
the cleanup applies only to the exact sandbox revision it observed. This guard
does not refresh or extend the active-operation lease or the server-owned hard
expiry; a later cleanup pass reevaluates the current revision.

Organisation owners and administrators can set or replace an encrypted
environment secret after consenting to the dedicated write capability:

```sh
aient --profile example auth login
printf '%s' "$STRIPE_SECRET_KEY" |
  aient --profile example environment secrets set \
    --environment development \
    STRIPE_SECRET_KEY \
    --stdin
```

The profile selects the organisation and `--environment` is required. Secret
values are never returned by the command. Setting a secret is separate from
enabling customer CLI workloads and from deciding which environment
capabilities are sandbox-exportable.

For help, contact [support@aient.ai](mailto:support@aient.ai).
