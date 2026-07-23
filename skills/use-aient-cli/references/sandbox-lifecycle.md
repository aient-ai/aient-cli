# Sandbox lifecycle and commands

- [One-shot offload](#one-shot-offload)
- [Retained iteration](#retained-iteration)
- [Supervised execution recovery](#supervised-execution-recovery)
- [Explicit files](#explicit-files)
- [Public 0.6.2 boundary](#public-062-boundary)

## One-shot offload

From a Git checkout:

```sh
aient sandbox run --environment development -- pnpm test
```

With an explicit repository selector when inference is ambiguous:

```sh
aient sandbox run \
  --environment development \
  --repository acme/widget \
  -- pnpm test
```

`--environment` implies synchronization of the current workspace when
`--workspace` is omitted. Exact Git HEAD, index, tracked changes, and selected
extra files activate at `/workspace/repo`. `run` deletes its sandbox after the
command; `--keep` retains it. Use `--include` and `--exclude` for the explicit
non-Git layer, not to change tracked Git state.

For a non-Git tree, use `--upload PATH`; it is a direct file upload rather than
repository synchronization. Use repeatable `--download REMOTE=LOCAL` to fetch
artifacts before cleanup.

To retain an environment/repository-bound sandbox for later brokered exec,
create it through `run --keep` and keep using the same selectors:

```sh
aient sandbox run \
  --name laptop-offload \
  --keep \
  --environment development \
  --repository acme/widget \
  -- true

aient sandbox exec laptop-offload \
  --environment development \
  --repository acme/widget \
  --workdir /workspace/repo \
  -- gh pr status
```

## Retained iteration

```sh
aient sandbox create --name laptop-offload --lease-seconds 21600
aient sandbox wait laptop-offload
aient sandbox sync laptop-offload . \
  --include 'fixtures/local-only/**' \
  --exclude '**/*.generated'
aient --timeout 15m sandbox exec laptop-offload \
  --workdir /workspace/repo -- go test ./...
aient sandbox files ls laptop-offload /workspace/repo
aient sandbox shell laptop-offload
aient sandbox delete laptop-offload
```

Use `sandbox list`, `status`, and `logs` to inspect named sandboxes. Customer
environment inventories use:

```sh
aient sandbox list --environment development
```

Each `sandbox shell` opens a fresh, non-resumable PTY. Public `0.6.2` does not
project brokered environment or GitHub capabilities into that shell; use a
repository/environment-bound `sandbox exec` when those capabilities are
required.

Do not use the customer-visible CPU or memory flags as portable sizing
controls; current environment policy owns customer resources.

## Supervised execution recovery

`sandbox exec` prints an execution UUID before dispatch and streams live
stdout/stderr. Capture the UUID and the terminal tool's session handle
separately:

```sh
aient --timeout 15m sandbox exec laptop-offload \
  --workdir /workspace/repo -- pnpm test
```

After transport loss, never POST the command again. Observe the same execution:

```sh
aient sandbox execution status laptop-offload EXECUTION_UUID
aient sandbox execution attach laptop-offload EXECUTION_UUID
aient sandbox execution wait laptop-offload EXECUTION_UUID
aient sandbox execution cancel laptop-offload EXECUTION_UUID
```

Pass the same `--environment` and `--repository` selectors when the original
execution required them. Status and wait return content-free state/outcome;
output is live-only and not replayed. Only one attachment should observe an
execution at a time.

This is reconnection, not durable detach. An unobserved, connection-bound
execution is cancelled after its server-owned grace period.

## Explicit files

```sh
aient sandbox files put SANDBOX LOCAL_PATH /absolute/remote/directory
aient sandbox files ls SANDBOX /absolute/remote/directory
aient sandbox files get SANDBOX /absolute/remote/file LOCAL_PATH
aient sandbox files rm SANDBOX /absolute/remote/path
```

Use these for deliberate artifacts, not as a fallback while `sandbox sync` is
still active. Never upload an access-token file, its hard link, or another
credential source.

## Public 0.6.2 boundary

Public commands include `auth`, `sandbox`, `version`, `completion`, and `help`.
The `agent` group is reserved but not functional. Sandbox operations include
`list`, `create`, `status`, `wait`, `sync`, `run`, `exec`, `shell`,
`execution attach|status|wait|cancel`, `logs`, `files put|get|ls|rm`, and
`delete`.

`suspend`, `resume`, and Docker bootstrap are operator-only. Public `0.6.2`
does not provide durable detach, port publication/forwarding, size presets,
shell reattachment, output history, or replayed command output.
