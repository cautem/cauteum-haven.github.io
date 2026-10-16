<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# CLI surface

```text
cautem sandbox create|attach|list|get|stop|start|delete|exec|connect|upload|download|…
cautem cp <local-path> <sandbox>:/path | <sandbox>:/path <local-path>
cautem sandbox template create|list|get|delete
cautem sandbox provider …
cautem provider create|list|get|refresh|update|delete|profile …
cautem profile …
cautem gateway add|select|ensure|info|login|…
cautem policy get|set|…
cautem rule get|approve|reject|…
cautem workspace|forward|service|settings|inference|whoami|logs|term
cautem health|init|doctor|status|version|install|completions|ssh-proxy
```

## Create (compute)

| Flag | Meaning |
|------|---------|
| `--name` | Sandbox name |
| `--from` / `--image` | Image alias or OCI reference |
| `--build DIR` | Build a local image from a Docker-compatible build context |
| `--dockerfile PATH` | Dockerfile relative to `--build` (defaults to `Dockerfile`) |
| `--workspace` | Host path → `/workspace` |
| `--policy` | Base policy YAML |
| `--cpu` / `--memory` / `--pids-limit` | Runtime limits |
| `--template` | Named workload template |
| `--provider` | Credential provider (repeatable) |
| `--no-proxy` | Skip egress sidecar (dev) |
| `--display novnc` | GUI image path |
| `--gpu` | NVIDIA CDI devices |

`cautem sandbox create --help` lists the full flag set, including `--remote`,
`--gateway`, `--gateway-workspace`, `--env`, `--label`, `--forward`, `--upload`,
`--restart-policy`, `--approval-mode`, `--editor`, display controls and the
explicit hardening/volume/provider switches. Each `sandbox` leaf has its own
help route; run `cautem sandbox <leaf> --help` for its accepted arguments.

`--remote` creates on the selected Gateway compute backend. Use
`--gateway-workspace NAME` to select the target workspace. Image based creates
accept CPU, memory, GPU count, CDI device IDs via `--cdi`, PID limits via
`--pids-limit`, environment, labels and driver config JSON. A
shared template owns its workload settings and rejects inline overrides. A
remote main command attaches to the canonical process unless `--detach` is
set. Reconnect with `cautem sandbox attach NAME`; use `--tty` for a process
created with a PTY.

The SSH helper invoked by generated `sandbox ssh-config` entries supports the
current gateway shorthand, a named gateway, and an explicit gateway URL with a
sandbox token. Run `cautem ssh-proxy --help` for the exact forms.

Local `--build DIR` builds through Docker Engine or Podman before creating the
sandbox. Build context files follow `.dockerignore`, and the Dockerfile must
stay inside the context. Local builds are unavailable with `--remote`.

Add `--gateway` to a `sandbox template` command to manage the template in the
selected Gateway workspace. Without it, the command uses local templates.

## Sandbox subcommands

| Command | Shape |
|---------|-------|
| `sandbox list` | `--limit N`, `--offset N`, `--selector EXPR`, `-o table\|json`, `--ids`, `--names`, `--all-workspaces` |
| `sandbox get` | `NAME` |
| `sandbox start` / `sandbox stop` | Optional `NAME` (defaults to the last-used sandbox) |
| `sandbox delete` | `NAME...` or `--all` (exclusive) |
| `sandbox upload` | `NAME LOCAL_PATH [SANDBOX_DEST] [--no-git-ignore]` |
| `sandbox download` | `NAME SANDBOX_PATH [LOCAL_DEST]` |
| `sandbox ssh-config` | `NAME [--install]` |
| `sandbox connect` | `[NAME] [--editor vscode\|cursor]` or `[NAME] -- COMMAND...` |
| `sandbox provider list` | `SANDBOX` |
| `sandbox provider attach` / `detach` | `SANDBOX PROVIDER` |
| `sandbox template create` | `NAME` with image/from/policy, CPU/memory/PIDs/GPU, env, labels, driver config and provider flags |
| `sandbox template list` | Optional `--gateway` |
| `sandbox template get` / `delete` | `NAME [--gateway]` |

`--editor` and a command cannot be combined. `--no-git-ignore` is upload-only;
the default upload destination is `/workspace` and the default download
destination is the current directory.

## Workspace commands

| Command | Shape |
|---------|-------|
| `workspace create` | `--name NAME`, repeatable `--label KEY=VALUE` |
| `workspace list` | `--page-size N`, `--page-token TOKEN`, `--label-selector KEY=VALUE[,KEY=VALUE...]`, `-o table\|yaml\|json` |
| `workspace get` | `NAME` |
| `workspace delete` | `NAME...` |

The list command returns one page. Pass its `next_page_token` as `--page-token`
to fetch the next page. Label selector pairs are combined with AND.

## Soft defaults

When flags (and template) omit sizing, config/env may fill gaps — never a
hard-coded memory. See [Docker resources](../providers/docker/resources.md).
