# Sandbox lifecycle and commands

- [One-shot offload](#one-shot-offload)
- [Retained customer execution](#retained-customer-execution)
- [Supervised execution recovery](#supervised-execution-recovery)
- [Explicit files on unbound/operator sandboxes](#explicit-files-on-unboundoperator-sandboxes)
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
non-Git layer, not to change tracked Git state. For Git workspace
synchronization, an `--exclude` is valid only when the same command has at
least one `--include`; exclusion narrows that explicit extra-file layer rather
than defining a standalone upload set. Raw `sandbox run --upload` and
`sandbox files put --exclude` use independent file-transfer filters and do not
require `--include`.
`sandbox sync` supports `--remote-dir` for an alternate absolute destination.
Git synchronization atomically replaces that destination directory, so it must
name a replaceable child such as `/workspace/alternate-repo`, never the mounted
`/workspace` root itself:

```sh
aient sandbox sync laptop-offload . \
  --remote-dir /workspace/alternate-repo
```

Raw `--upload` may target `/workspace` because it uses the file-upload path
rather than Git-directory activation.

For a non-Git tree, use `--upload PATH`; it is a direct file upload rather than
repository synchronization. Use repeatable `--download REMOTE=LOCAL` to fetch
artifacts before cleanup. For retained customer work, continue with the next
section.

## Retained customer execution

For a customer OAuth session, create and retain the sandbox through the
environment-aware `run --keep` path. The initial `run` transfers the selected
workspace; public `0.6.2` cannot re-sync that environment-bound retained
sandbox:

```sh
aient sandbox run \
  --name laptop-offload \
  --keep \
  --environment development \
  --repository acme/widget \
  -- true

aient --timeout 15m sandbox exec laptop-offload \
  --environment development \
  --repository acme/widget \
  --workdir /workspace/repo -- go test ./...
aient sandbox delete laptop-offload
```

Use the bound `execution attach|status|wait|cancel` commands with the same
environment and repository selectors to observe a supervised `exec`. Customer
environment inventories use:

```sh
aient sandbox list --environment development
```

The following endpoints do not accept the environment/repository selectors
needed to authorize a bound customer sandbox in `0.6.2`:

- `sandbox sync`
- `sandbox files put|get|ls|rm`
- `sandbox shell`
- `sandbox status`
- `sandbox wait`
- `sandbox logs`

They are ordinary unbound/operator lifecycle commands, not a customer retained
loop. Do not work around the authorization failure with an operator credential.
Create a new environment-bound `sandbox run` to transfer newer local state.
To inspect the retained workspace, run a specific command such as `ls` through
bound `sandbox exec`; use bound `sandbox execution status` for one execution
and `sandbox list --environment development` for customer inventory. Command
output remains live-only rather than becoming a `sandbox logs` history.
Although public help also lists bare `sandbox create`, it is not the
environment-bound customer entry point. Do not copy its template, CPU, memory,
named-volume, or metadata flags into customer instructions merely because they
appear in help; current environment policy owns customer resources.

## Supervised execution recovery

`sandbox exec` prints an execution UUID before dispatch and streams live
stdout/stderr. Capture the UUID and the terminal tool's session handle
separately:

```sh
aient --timeout 15m sandbox exec laptop-offload \
  --environment development \
  --repository acme/widget \
  --workdir /workspace/repo -- pnpm test
```

After transport loss, never POST the command again. Observe the same execution:

```sh
aient sandbox execution status laptop-offload EXECUTION_UUID \
  --environment development --repository acme/widget
aient sandbox execution attach laptop-offload EXECUTION_UUID \
  --environment development --repository acme/widget
aient sandbox execution wait laptop-offload EXECUTION_UUID \
  --environment development --repository acme/widget
aient sandbox execution cancel laptop-offload EXECUTION_UUID \
  --environment development --repository acme/widget
```

Always pass the same `--environment` and `--repository` selectors when the
original execution required them. Status and wait return content-free
state/outcome; output is live-only and not replayed. Only one attachment should
observe an execution at a time.

This is reconnection, not durable detach. An unobserved, connection-bound
execution is cancelled after its server-owned grace period.

## Explicit files on unbound/operator sandboxes

```sh
aient sandbox files put SANDBOX LOCAL_PATH /absolute/remote/directory
aient sandbox files ls SANDBOX /absolute/remote/directory
aient sandbox files get SANDBOX /absolute/remote/file LOCAL_PATH
aient sandbox files rm SANDBOX /absolute/remote/path
```

These commands are available only on the ordinary unbound/operator path in
`0.6.2`; they cannot target an environment-bound customer sandbox retained by
`run --keep`. Use them for deliberate artifacts, not as a fallback while
`sandbox sync` is still active. Never upload an access-token file, its hard
link, or another credential source.

## Public 0.6.2 boundary

Public commands include `auth`, `sandbox`, `version`, `completion`, and `help`.
The `agent` group is reserved but not functional. Sandbox operations include
`list`, `create`, `status`, `wait`, `sync`, `run`, `exec`, `shell`,
`execution attach|status|wait|cancel`, `logs`, `files put|get|ls|rm`, and
`delete`.

`suspend`, `resume`, and Docker bootstrap are operator-only. Bare `create` is
not the environment-bound customer entry point; use `run --keep` for retained
customer `exec` work. `sync`, `files`, `shell`, `status`, `wait`, and `logs`
remain unbound/operator operations and reject that bound customer sandbox.
Public `0.6.2` does not provide customer retained re-sync, durable detach, port
publication/forwarding, size presets, shell reattachment, output history, or
replayed command output.
