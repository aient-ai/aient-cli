# Troubleshooting

## Partial sync or missing repository

Success requires both:

- the actual `aient sandbox sync` process returns a successful terminal exit;
- its output contains the final `Synchronized ...` completion line.

Progress such as `56.0 MiB / 448.4 MiB`, a wrapper-level “Script completed,” an
HTTP 200 observed elsewhere, or a later directory listing is insufficient. A
wrapper may have returned after its tool yielded while the child process
remained active.

If an exec tool returns a running session ID, retain it and poll that exact
session. Do not discard everything except stdout. If the handle was discarded,
do not infer completion or start compensating mutations. First determine
whether the original operation is still active from an authoritative operation
record or process/session view.

Never overlap:

- two syncs;
- sync plus `files put`;
- sync plus manual archive extraction;
- sync plus another workspace replacement or activation.

Wait for a terminal result before retrying. After confirmed failure, inspect
`sandbox status`, `files ls`, and safe verbose metadata only on an ordinary
unbound/operator sandbox. For an environment-bound retained customer sandbox,
use `sandbox status --environment ENV` for lifecycle state and a
specific bound `sandbox exec` command to inspect its workspace.

## Workspace busy

A workspace-busy response normally means another mutation owns the exclusive
fence. The exact HTTP `503` machine code
`workspace_layout_certification_busy` proves the command was rejected before
dispatch. Do not attach to or poll that rejected execution, and do not expect
the CLI to replay it. Find and wait for the owning CLI/session operation, then
start the command once after that owner is terminal.

Do not generalize this rule to every `503`. A generic or proxy-authored `503`, a
truncated response, or a transport failure is outcome-ambiguous. Preserve the
execution UUID and reconcile that same execution rather than submitting a
second start request. Never bypass the fence or assemble a second tree
manually.

## Execution stream ended

For retained `sandbox exec`, use the printed execution UUID:

```sh
aient sandbox execution status SANDBOX EXECUTION_UUID \
  --environment development \
  --repository acme/widget
aient sandbox execution attach SANDBOX EXECUTION_UUID \
  --environment development \
  --repository acme/widget
```

Do not rerun the original command merely because stdout ended or the network
returned EOF. A second attachment supersedes the first; coordinate one active
observer. Output already delivered is not replayed. Repeat the original
`--environment` and any required `--repository` selector.

If no execution UUID was preserved, status recovery cannot safely identify the
command. Treat that as a calling-tool lifecycle-handle defect, not evidence
that the CLI command failed or succeeded.

## Sandbox state, readiness, logs, or expiry

For an environment-bound retained sandbox:

```sh
aient sandbox status SANDBOX --environment development
aient sandbox wait SANDBOX --environment development
aient sandbox logs SANDBOX \
  --environment development \
  --tail-lines 200
```

Use `status` for a current nonblocking snapshot, `wait` only for readiness, and
`logs` for a bounded sandbox log snapshot. To wait for command completion, use
`sandbox execution wait SANDBOX EXECUTION_UUID` with the original execution
selectors. Sandbox logs are not retained command output.

Do not increase the default lease merely because foreground work is long.
Healthy active operations receive bounded rolling protection automatically.
That protection stops after client loss and cannot cross hard expiry. `--keep`
is not permanent ownership; explicitly delete a retained sandbox:

```sh
aient sandbox delete SANDBOX
```

If a sandbox expires despite an active client, preserve the exact command,
timestamps, sandbox name, client version, and safe diagnostic output for
support. Do not recreate first if doing so would destroy evidence.

## Authentication or wrong customer project

```sh
aient auth status --all
aient auth profiles
aient --profile EXPECTED auth status
```

Inspect the nearest `.aient/config.yaml`, `AIENT_PROFILE`, current directory,
and explicit global flags. A recovery command started outside the checkout may
select a different profile. Do not solve customer OAuth errors with
`--profile operator` or an operator API key.

If a development environment is denied, an organisation administrator must
enable customer CLI workloads for that environment. Repository inference is
only a selector; use `--repository owner/name` to disambiguate, but expect the
server to reject authority outside the authenticated organisation's verified
installation and policy.

Portable credentials cannot refresh. Export a new access-only token after
expiry or rejection; do not copy refresh/profile state.

If `status`, `wait`, or `logs` cannot see an environment-bound sandbox, confirm
the installed CLI reports `0.10.4` and repeat `--environment`. Do not add
`--repository` to lifecycle reads. `sync`, `files`, and `shell` remain
unbound/operator surfaces; do not bypass their rejection with operator
credentials.

## Broad non-Git upload

An `--exclude` without any `--include` selects every non-Git
path first. If a proposed command excludes only caches or build output, stop:
ignored `.env`, `.npmrc`, cloud credentials, SSH keys, and portable Aient
tokens may still upload.

Prefer narrow `--include` globs. If broad selection is truly required, audit
the complete non-Git tree and explicitly exclude every credential source.
Remember that `--exclude` never removes tracked Git files and that there is no
implicit secret denylist.

Release `0.10.4` automatically omits regular macOS AppleDouble sidecars whose
base name starts with `._` from recursive non-Git selection and recursive
directory uploads through `sandbox run --upload DIRECTORY` and
`sandbox files put SANDBOX DIRECTORY ABSOLUTE_REMOTE_DIRECTORY`. Do not infer a
broader cache or secret filter. Git-tracked paths and explicitly named
single-file uploads remain exact, and local files are never removed.

## Environment secret write fails

Use an owner/admin profile and stdin:

```sh
printf '%s' "${SECRET_VALUE}" |
  aient --profile acme environment secrets set \
    --environment development \
    SECRET_NAME \
    --stdin
```

An older OAuth session may lack `aient.environment.secrets.write`; log in again
to complete browser consent. A successful set still does not enable customer
workloads or mark the value sandbox-exportable. Do not print the secret inside a
sandbox to verify it.

## Check capability before promising it

Run the relevant help command:

```sh
aient sandbox run --help
aient sandbox status --help
aient sandbox wait --help
aient sandbox logs --help
aient sandbox exec --help
aient sandbox execution --help
```

Do not invent detach, port forwarding, agent, resumable-shell, raw customer
infrastructure, or persisted-output workflows.
