# Troubleshooting

## Partial sync or missing repository

Success requires both:

- the actual `aient sandbox sync` process returns a successful terminal exit;
- its output contains the final `Synchronized ...` completion line.

Progress such as `56.0 MiB / 448.4 MiB`, a wrapper-level “Script completed,” an
HTTP 200 observed elsewhere, or a later directory listing is insufficient on
its own. A generic wrapper may have returned after its tool yielded while the
child process remained active.

If an exec tool returns a running session ID, retain it and poll that exact
session. Do not discard everything except stdout. If the handle was discarded,
do not infer completion or start compensating mutations. First determine
whether the original operation is still active from an authoritative operation
record or process/session view.

Never overlap these workspace mutations:

- two syncs;
- sync plus `files put`;
- sync plus manual archive extraction;
- sync plus another operation that replaces or activates the workspace.

Wait for a terminal result before retrying. After confirmed failure, inspect
`aient sandbox status SANDBOX`, `aient sandbox files ls SANDBOX /workspace`,
and safe verbose metadata with `--verbose`.

## Workspace busy

A workspace-busy response normally means another mutation owns the exclusive
fence. Do not bypass it or assemble a second tree manually. Find and wait for
the owning CLI/session operation, then retry once the owner is terminal.

## Execution stream ended

For retained `sandbox exec`, use the printed execution UUID:

```sh
aient sandbox execution status SANDBOX EXECUTION_UUID
aient sandbox execution attach SANDBOX EXECUTION_UUID
```

Do not rerun the original command merely because stdout ended or the network
returned EOF. A second attachment supersedes the first; coordinate one active
observer. Output already delivered to an earlier attachment is not replayed.

If no execution UUID was preserved, status recovery cannot safely identify the
command. Treat that as a calling-tool lifecycle-handle defect, not evidence
that the CLI command failed or succeeded.

## Authentication or wrong customer project

```sh
aient auth status --all
aient auth profiles
aient --profile EXPECTED auth status
```

Inspect the nearest `.aient/config.yaml`, `AIENT_PROFILE`, and explicit global
flags. Do not solve customer OAuth errors with `--profile operator` or an
operator API key.

If a development environment is denied, an organisation administrator must
enable customer CLI workloads for that environment. Repository inference is
only a selector; use `--repository owner/name` to disambiguate, but expect the
server to reject any repository outside the authenticated organisation's
verified installation and policy.

Portable credentials cannot refresh. Export a new access-only token after
expiry or rejection; do not copy refresh/profile state.

## Check capability before promising it

Run the relevant help command:

```sh
aient sandbox run --help
aient sandbox exec --help
aient sandbox execution --help
aient sandbox files --help
```

Do not invent `--detach`, `--port`, `--size`, agent, or resumable-shell
workflows for public `0.6.2`.
