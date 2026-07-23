---
name: use-aient-cli
description: "Operate the public Aient CLI for remote development and test offload: customer OAuth login, named project profiles, portable access-only credentials, exact Git/worktree synchronization, one-shot sandbox runs, retained sandbox exec/shell/file workflows, and live execution recovery. Use when asked to run local work in an Aient sandbox, move a checkout between local and remote compute, reconnect to an execution, troubleshoot a CLI transfer or sandbox command, or isolate Aient credentials by organisation, repository, or folder."
---

# Use Aient CLI

Use the public customer CLI to run local work remotely without copying operator
credentials into a project. These instructions describe public release `0.6.2`.
Start with:

```sh
aient version
aient --help
```

Treat the installed binary's help as authoritative if the version differs.
When already inside an Aient sandbox or the native Aient Harness, run commands
there directly; do not create a nested CLI sandbox.

## Choose the workflow

- Run a disposable check: `aient sandbox run --environment development -- pnpm test`
- Iterate in retained compute: `sandbox create`, then `sync`, `exec`, and
  finally `delete`.
- Debug interactively: `aient sandbox shell SANDBOX`. Each invocation opens a
  new, non-resumable shell.
- Transfer one explicit artifact: `aient sandbox files put|get|ls|rm`.
- Recover a supervised command after transport loss: preserve the execution
  UUID and use `sandbox execution attach|status|wait|cancel`.

Read [sandbox-lifecycle.md](references/sandbox-lifecycle.md) before composing a
retained workflow or using execution recovery.

## Select customer identity

Prefer a named profile per customer or organisation and a nearest-project
`.aient/config.yaml` selector. Keep refresh credentials in the operating-system
credential store. For a different process or machine, export a short-lived,
access-only credential instead of copying `~/.aient`, profile files, Keychain
items, Secret Service entries, or refresh state.

Read [auth-and-project-context.md](references/auth-and-project-context.md)
before logging in, switching projects, exporting a credential, or adding
repository context.

## Preserve operation truth

For any command launched through an agent, task runner, or wrapper:

1. Preserve the complete child result, including a running session handle,
   terminal exit status, and output.
2. If the tool yields a session handle, wait or poll that exact handle until a
   terminal result arrives.
3. Consider `sandbox sync` complete only when the actual CLI process exits
   successfully **and** prints its final `Synchronized ...` line.
4. Do not start a second sync, upload files, extract archives, or otherwise
   mutate the workspace while a sync or another workspace mutation may still
   be running.
5. Use only one active output attachment per execution. A second attachment
   supersedes the first, and earlier bytes are not replayed.

A wrapper's own “completed” message, partial transfer progress, elapsed time,
HTTP acceptance, or the later existence of `/workspace/repo` does not prove
that the original CLI operation completed.

Read [troubleshooting.md](references/troubleshooting.md) whenever output is
partial, a connection ends, the workspace is missing/busy, or authentication
selects the wrong project.

## Respect the current boundary

- `operator` is an internal dogfood profile. Do not tell customers to use an
  operator API key, `--auth-env-file`, `sandbox suspend`, `sandbox resume`, or
  Docker bootstrap.
- Do not claim public `0.6.2` supports a functional `aient agent`, durable
  detach, port publication/forwarding, size presets, resumable shell history,
  or persisted/replayed stdout and stderr.
- Do not invent flags from Modal or another CLI. Check `aient COMMAND --help`
  before presenting a ready-to-run command.
