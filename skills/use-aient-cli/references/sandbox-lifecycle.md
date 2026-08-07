# Sandbox lifecycle and commands

The published stable release is `0.10.4`. The installed binary's help is the
authority for its exact command surface.

- [Disposable offload](#disposable-offload)
- [Retained customer workflow](#retained-customer-workflow)
- [Lifecycle reads and waits](#lifecycle-reads-and-waits)
- [Supervised execution recovery](#supervised-execution-recovery)
- [Compute sizes and lease behavior](#compute-sizes-and-lease-behavior)
- [Workspace and file selection](#workspace-and-file-selection)
- [Public boundary](#public-boundary)

## Disposable offload

From a Git checkout:

```sh
aient sandbox run \
  --environment development \
  --size medium \
  -- pnpm test
```

With an explicit repository selector when inference is ambiguous or the command
needs repository-scoped provider capability:

```sh
aient sandbox run \
  --environment development \
  --repository acme/widget \
  -- pnpm test
```

`--environment` synchronizes the current Git workspace when `--workspace` is
omitted. Exact Git HEAD, index, tracked changes, and selected non-Git files
activate at `/workspace/repo`. `run` deletes its sandbox after the command;
`--keep` retains it.

For a non-Git tree, use `--upload PATH`; it is a direct file upload rather than
repository synchronization. Use repeatable `--download
ABSOLUTE_REMOTE_PATH=LOCAL_PATH` to fetch artifacts before cleanup.

## Retained customer workflow

Use `run --keep` when a customer checkout must remain available for later
commands. It creates the environment-bound sandbox and performs the initial
workspace synchronization in one composition:

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
  --repository acme/widget \
  -- true || exit 1

aient sandbox status "${SANDBOX}" --environment development
aient sandbox wait "${SANDBOX}" --environment development

aient --timeout 15m sandbox exec "${SANDBOX}" \
  --environment development \
  --repository acme/widget \
  --workdir /workspace/repo \
  -- go test ./...

aient sandbox logs "${SANDBOX}" \
  --environment development \
  --tail-lines 200

aient sandbox delete "${SANDBOX}" || exit 1
trap - EXIT
```

Omit `--repository` from `run` and `exec` when the command does not need
repository-scoped provider capability and inference is not desired. If the
original supervised command did require the selector, repeat it for every
execution recovery operation.

Bare `sandbox create --environment ENV [--size CLASS]` is also a valid customer
entry point, but it creates empty compute; it does not synchronize a checkout.
Use it for non-workspace workloads, then delete it explicitly.

`sandbox sync`, `sandbox files`, and `sandbox shell` do not accept the
environment selector. They remain ordinary unbound/operator
surfaces, not a retained customer loop. Do not use an operator credential to
bypass that boundary. To transfer a newer customer workspace, create a fresh
environment-bound `sandbox run`.

## Lifecycle reads and waits

Environment-bound inventory and lifecycle reads use the environment context:

```sh
aient sandbox list --environment development
aient sandbox status laptop-offload --environment development
aient sandbox wait laptop-offload --environment development
aient sandbox logs laptop-offload \
  --environment development \
  --tail-lines 200
```

Do not add `--repository` to these commands; their help does not accept
it, and lifecycle observation does not release provider capability.

Choose the right surface:

| Command | Meaning |
|---|---|
| `sandbox status SANDBOX` | Return the current lifecycle snapshot without waiting. |
| `sandbox wait SANDBOX` | Poll until that sandbox becomes ready. |
| `sandbox logs SANDBOX` | Return a bounded recent sandbox log snapshot. |
| `sandbox execution status SANDBOX EXECUTION` | Return content-free state for one supervised execution. |
| `sandbox execution wait SANDBOX EXECUTION` | Wait for that execution's content-free terminal outcome. |
| `sandbox execution attach SANDBOX EXECUTION` | Attach to new live output; previously delivered bytes are not replayed. |

`sandbox logs` is not command stdout/stderr history. Output from supervised
`exec` is live-only.

Delete deliberately:

```sh
aient sandbox delete laptop-offload
```

Delete intentionally has no environment selector. Cleanup remains owner-scoped
so revoked or removed environment access cannot strand a retained sandbox.

## Supervised execution recovery

`sandbox exec` prints an execution UUID before dispatch and streams live
stdout/stderr. Preserve that UUID and the calling tool's local session handle
separately:

```sh
aient --timeout 15m sandbox exec laptop-offload \
  --environment development \
  --repository acme/widget \
  --workdir /workspace/repo \
  -- pnpm test
```

After transport loss, never submit the command again. Observe the same
execution:

```sh
aient sandbox execution status laptop-offload EXECUTION_UUID \
  --environment development \
  --repository acme/widget
aient sandbox execution attach laptop-offload EXECUTION_UUID \
  --environment development \
  --repository acme/widget
aient sandbox execution wait laptop-offload EXECUTION_UUID \
  --environment development \
  --repository acme/widget
aient sandbox execution cancel laptop-offload EXECUTION_UUID \
  --environment development \
  --repository acme/widget
```

Use only one attachment at a time. A second attachment supersedes the first,
and output already delivered to either attachment is not replayed. Status and
wait return state/outcome, not command bytes.

This is reconnection, not durable detach. An unobserved, connection-bound
execution is cancelled after its server-owned grace period.

## Compute sizes and lease behavior

Environment-bound customer `create` and `run` accept only:

- `--size small`
- `--size medium`
- `--size large`

Omit `--size` to use the environment's configured default. The server resolves
the selected class to product-owned resources and rejects a class the
environment does not allow.

Do not copy shared operator flags such as `--template`, `--cpu`, `--memory`,
`--cpu-limit`, `--memory-limit`, `--metadata`, or `--named-volume` into
customer instructions. Customer mode rejects raw infrastructure authority.

Active foreground CLI operations automatically receive a short rolling lease
while they are healthy. Upload, command, shell, download, and bootstrap work do
not need a longer default lease merely to stay alive. This guarantee:

- never shortens an existing longer lease;
- does not change the default lease;
- cannot cross the server-owned hard expiry;
- stops when the client is lost; and
- does not make a retained sandbox permanent.

A supervised execution has bounded command-budget and reconnect-grace lease
coverage. After that window, it does not own the sandbox. Explicitly delete
retained sandboxes even when they will eventually expire.

## Workspace and file selection

Git synchronization always includes exact tracked state. Non-Git selection is
separate:

| Flags | Explicit non-Git layer |
|---|---|
| no `--include` and no `--exclude` | none; Git-only synchronization |
| one or more `--include` | only matching non-Git paths |
| `--include` plus `--exclude` | included paths minus exclusions |
| `--exclude` without `--include` | **all non-Git paths** minus exclusions |

Exclude-only mode is broad, including ignored paths. It can upload `.env`,
`.npmrc`, `.aws/credentials`, SSH keys, cloud configuration, portable Aient
tokens, dependency caches, and build output unless every such path is excluded.
There is no implicit secret or cache denylist.

Recursive non-Git selection and recursive `sandbox files put` directory
uploads omit regular macOS AppleDouble sidecars whose base name starts with
`._`. The filter does not change Git-tracked paths, explicitly named
single-file uploads, ordinary dotfiles, directories named `._*`, or local
source bytes.

Prefer narrow, repeatable `--include` globs. If broad exclude-only selection is
unavoidable, inventory the entire non-Git tree first and place an explicit
secret-upload warning beside the command. Exclusions do not alter tracked Git
state, so remove a tracked secret from Git rather than relying on `--exclude`.

Keep access-token files outside the workspace. Never upload a token file, its
hard link, refresh/profile state, an OS credential store, or another credential
source.

`sandbox sync` supports `--remote-dir` for an alternate absolute destination.
Git synchronization atomically replaces that destination directory, so use a
replaceable child such as `/workspace/alternate-repo`, never the mounted
`/workspace` root:

```sh
aient sandbox sync unbound-sandbox . \
  --remote-dir /workspace/alternate-repo
```

Raw `sandbox run --upload` may target `/workspace` because it uses direct file
upload rather than Git-directory activation.

### Exact generated-directory reset

Use repeatable `--reset-remote-dir` on `sandbox run` or ordinary unbound
`sandbox sync` when an exact repository-relative generated directory must be
replaced by an empty ordinary directory during the same certified activation:

```sh
aient sandbox run \
  --environment development \
  --reset-remote-dir packages/example/dist \
  -- pnpm test
```

Reset paths are exact directory paths, not globs. The operation changes only
the remote candidate; local files are not removed or modified. Do not reset a
directory whose contents the remote command still needs. The repository root,
absolute paths, wildcard paths, any `.` or `..` component, `.git`, duplicates,
overlapping reset roots, Git-tracked content, and Gitlink boundaries fail
locally before network side effects. Reset cannot be combined with
`--exclude`.

### Initialized submodule policy

Initialized submodules are rejected by default. Use
`--submodules=gitlinks` only when the command needs the parent repository but
deliberately does not need checked-out child content:

```sh
aient sandbox run \
  --environment development \
  --submodules=gitlinks \
  -- go test ./...
```

Every initialized child and descendant must be clean and checked out at the
exact object recorded by its parent index. The CLI transfers the complete
sorted Gitlink boundary and omits child bytes. Dirty, untracked, mismatched,
malformed, or ambiguous child state fails locally. This policy is not recursive
submodule transfer.

## Public boundary

The command groups are `auth`, `environment`, `sandbox`, `version`,
`completion`, `help`, and reserved `agent`.

The following remain operator-only:

- `--profile operator`, `--api-key`, `--api-url`, and `--auth-env-file`
- raw template/resource/storage/metadata flags
- `sandbox suspend` and `sandbox resume`
- `sandbox docker bootstrap`

`sandbox sync`, `sandbox files`, and `sandbox shell` remain available only on
the unbound lifecycle path. They may be used with an ordinary unbound customer
or operator sandbox, but not to bypass an environment-bound customer sandbox's
authorization boundary.

The `agent` group is reserved for a later slice. Version `0.10.4` does not provide
durable detach, port publication/forwarding, shell reattachment, output
history, or replayed stdout/stderr.
