---
name: use-aient-cli
description: "Operate the public Aient CLI for remote development and test offload: customer OAuth and folder-scoped profiles, portable access-only credentials, exact Git/worktree synchronization, disposable and retained environment sandboxes, size presets, lifecycle reads, supervised execution recovery, environment-secret administration, and safe non-Git file selection. Use when asked to run local work in an Aient sandbox, move a checkout between local and remote compute, inspect or clean up a retained sandbox, reconnect to an execution, troubleshoot a CLI transfer, set an environment secret, or isolate Aient credentials by organisation, repository, or folder."
---

# Use Aient CLI

Use the public customer CLI to run local work remotely without copying operator
credentials into a project. The published stable release is `0.8.1`. Check:

```sh
aient version
aient --help
```

Treat the installed binary's help as authoritative. When already inside an
Aient sandbox or the native Aient Harness, run commands there directly; do not
create a nested CLI sandbox.

## Choose the workflow

- Run a disposable check:
  `aient sandbox run --environment development -- pnpm test`
- Retain a synchronized checkout: use environment-bound `sandbox run --keep
  --name NAME`, then environment-bound `exec`, lifecycle reads, and explicit
  `delete`.
- Create empty customer compute only when no checkout transfer is needed:
  `aient sandbox create --environment development [--size small|medium|large]`.
- Recover a supervised command after transport loss: preserve its execution
  UUID and use `sandbox execution attach|status|wait|cancel`; never replay the
  start request.

Read [sandbox-lifecycle.md](references/sandbox-lifecycle.md) before composing a
retained workflow, selecting non-Git files, or using execution recovery.

## Select customer identity

Prefer a named profile per organisation and a nearest-project
`.aient/config.yaml` selector per checkout or worktree. Keep refresh credentials
in the operating-system credential store. For a different process or machine,
export a short-lived, access-only credential instead of copying `~/.aient`,
profile files, Keychain items, Secret Service entries, or refresh state.

Read [auth-and-project-context.md](references/auth-and-project-context.md)
before logging in, switching folders/projects, exporting a credential, or
setting an environment secret.

## Keep environment context on bound operations

Pass `--environment` to `sandbox list`, `status`, `wait`, and
`logs` when targeting an environment-bound customer sandbox. Do not add
`--repository` to those lifecycle reads; it is unsupported and grants no useful
read authority.

Pass the original `--environment` and, when required, `--repository` to
`sandbox exec` and `sandbox execution attach|status|wait|cancel`.

Do not confuse the wait surfaces:

- `sandbox status` is a nonblocking lifecycle snapshot.
- `sandbox wait` waits for sandbox readiness.
- `sandbox execution wait` waits for one supervised execution's content-free
  terminal outcome.
- `sandbox logs` is a bounded sandbox log snapshot, not retained exec output.

## Use product-owned compute and secrets

Use `--size small|medium|large` on environment-bound `sandbox create` or
`sandbox run`, or omit it for the environment default. Customer mode rejects
raw template, CPU, memory, scheduler, storage, metadata, and named-volume
authority even though shared operator help exposes some of those flags.

Set an encrypted environment secret with `environment secrets set
--environment ENV NAME --stdin`. The actor must be an owner/admin and may need
browser re-consent for `aient.environment.secrets.write`. Setting a secret does
not enable workloads or make the value sandbox-exportable; environment
capability policy is a separate decision.

## Select uploads safely

With neither `--include` nor `--exclude`, Git synchronization transfers no
non-Git files. Exclude-only selection starts with **all**
non-Git paths and subtracts exclusions. It can therefore upload ignored `.env`,
`.npmrc`, cloud credentials, portable token files, and other secrets.

Prefer narrow `--include` globs. Use exclude-only mode only after auditing the
whole non-Git tree and explicitly excluding every credential source. Exclusions
do not remove tracked Git files. Never upload an access-token file, a hard link
to it, profile refresh state, or an operating-system credential store.

## Preserve operation truth

For any command launched through an agent, task runner, or wrapper:

1. Preserve the complete child result, including a running session handle,
   terminal exit status, execution UUID, and output.
2. If the tool yields a session handle, wait or poll that exact handle until a
   terminal result arrives.
3. Consider `sandbox sync` complete only when the actual CLI process exits
   successfully and prints its final `Synchronized ...` line.
4. Do not overlap sync, file upload, extraction, or another workspace mutation.
5. Use only one live attachment per execution. Earlier bytes are not replayed.

A wrapper's “completed” message, transfer progress, elapsed time, HTTP
acceptance, or later directory existence does not prove that the original CLI
operation completed.

Read [troubleshooting.md](references/troubleshooting.md) whenever output is
partial, a connection ends, the workspace is missing/busy, authentication
selects the wrong project, or a sandbox nears expiry.

## Protect secret-bearing output

An authorized child command can deliberately print a mounted secret. Aient does
not persist normal command output, but the connected caller, terminal, agent
transcript, or task runner may record returned bytes. Avoid `env`, `printenv`,
shell xtrace, and `echo` of provider tokens or mounted secrets unless the human
explicitly requests disclosure.

## Respect the public boundary

- Active foreground operations receive bounded lease protection automatically.
  This does not extend the default lease, cross hard expiry, or make `--keep`
  permanent. Delete retained sandboxes explicitly.
- `sync`, `files`, and `shell` remain unbound/operator surfaces; do not use
  operator credentials to bypass an environment-bound customer rejection.
- `operator`, raw infrastructure flags, `--auth-env-file`, `sandbox suspend`,
  `sandbox resume`, and Docker bootstrap are internal/operator paths.
- Do not claim `aient agent`, durable detach, port forwarding, resumable shell
  history, or persisted/replayed stdout and stderr are available.
- Check `aient COMMAND --help` before presenting a ready-to-run command. Do not
  invent flags from another sandbox CLI.
