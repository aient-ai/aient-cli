# Authentication and project context

## Named customer profiles

Log into each organisation with its own local name:

```sh
aient --profile acme auth login
aient --profile acme auth status
aient auth profiles
aient auth profiles default acme
```

`auth login` opens browser consent. The named profile's rotating refresh
credential stays in macOS Keychain or Linux Secret Service. Local profile files
contain non-secret lookup and selection metadata.

Select a profile in this order:

1. `--profile NAME`
2. `AIENT_PROFILE=NAME`
3. the nearest ancestor `.aient/config.yaml`
4. the user default set by `auth profiles default`
5. the compatibility profile on an otherwise unconfigured installation

Conflicting `--profile` and `AIENT_PROFILE` values fail closed.

## Bind a folder to a project

Create `.aient/config.yaml` in the checkout:

```yaml
profile: acme
organisation: ACME
repository: acme/widget
```

Only scalar `profile`, `organisation`, and `repository` keys are supported.
The CLI searches from the current directory upward, so one file can cover a
whole checkout or linked worktree. Do not put tokens, secrets, API keys, or
provider credentials in this file.

All local values are selectors, never authority. A Git `origin` or `upstream`
may help infer `owner/name`, but changing a remote does not grant access. The
server rechecks the signed actor, organisation membership, environment policy,
sandbox ownership, verified repository record, and provider installation.
Uploading and testing a local workspace does not require repository authority;
provider-backed Git push or pull-request operations do.

## Move access without moving refresh authority

Export a short-lived, access-only credential:

```sh
AIENT_ACCESS_DIR="$(mktemp -d /tmp/aient-access.XXXXXX)"
AIENT_ACCESS_FILE="${AIENT_ACCESS_DIR}/acme-access.json"
cleanup_aient_access() {
  rm -f -- "${AIENT_ACCESS_FILE}"
  rmdir -- "${AIENT_ACCESS_DIR}" 2>/dev/null || true
}
trap cleanup_aient_access EXIT

aient --profile acme auth export \
  --format file \
  --ttl 15m \
  --output "${AIENT_ACCESS_FILE}"

aient --access-token-file "${AIENT_ACCESS_FILE}" auth status
aient --access-token-file "${AIENT_ACCESS_FILE}" \
  sandbox list --environment development

cleanup_aient_access
trap - EXIT
```

`auth export --format file` creates the file atomically with mode `0600`. The
JSON contains a short-lived bearer access token: it is a secret even though it
has no refresh token and cannot be renewed. Keep the temporary directory
outside the uploaded workspace, leave the cleanup trap active, and export a
new file after expiry.

Portable sources are mutually exclusive:

- `--access-token-file PATH`
- `AIENT_ACCESS_TOKEN_FILE=PATH`
- `AIENT_ACCESS_TOKEN=TOKEN`

Do not combine a portable source with `--profile` or `AIENT_PROFILE`. Project
organisation/repository selectors still apply, but the portable token remains
the only customer credential. Prefer the file form over an environment token
because it avoids shell history and process-environment leakage.

Use `aient --profile NAME auth logout` to revoke and remove one named login.
Never copy the entire `~/.aient` directory or operating-system credential-store
state between projects or agents.
