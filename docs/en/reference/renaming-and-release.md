<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# Quarshell rename and release plan

Product and internal implementation names move to **Quarshell**. The compatibility
page may state that Quarshell supports OpenShell v0.1.4; that upstream name is
kept only where it describes the wire contract, dependency, attribution,
fixtures or compatibility scope.

This map is the migration source of truth. The GitHub owner remains `cautem`.
Product repositories have already been renamed in place to `quarshell-*`, and
the Go module path changes have merged to `main` for core, proto, providers,
runtime, driver, proxy, gateway, SDK, CLI, display and slogx. The old module
release tags remain unchanged; new-path tags and the consumer dependency
cascade have not yet been published. CLI names, package metadata, config paths,
environment variables and runtime resource identifiers still use Cautem names
until their migration and rollback behavior is implemented and tested.

The runtime compatibility gates are still open: the current gap check reports
12 of 18 tracked requirements as open or blocked. This is a migration plan and
status record, not a full compatibility or release claim. Product-owned code
should use Quarshell names; OpenShell remains only where it identifies the
wire contract, dependency, attribution, fixtures or compatibility scope.

## Migration status — 2026-10-16

- **GitHub repositories:** the product repositories now use `quarshell-*`
  names under owner `cautem`. The reserved `cautem/.github` repository remains
  unchanged.
- **Go module declarations:** new paths are on `main` for the eleven Go
  modules listed above. Published tags still point to commits with the old
  module declarations, so existing consumers continue to use the old paths.
- **New-path releases:** not published. Update dependency pins and imports in
  dependency order, then build external consumers with `GOWORK=off` before
  tagging each package wave.
- **Product cutover:** binary aliases, Python/npm package identity, config and
  data directories, environment variables, images, volumes, networks and
  labels have not completed migration. Preserve existing Cautem resources and
  data until an explicit, tested migration adopts them.
- **Runtime gates:** Docker, Podman and MicroVM evidence remains partial. The
  Gateway keeps MicroVM fail-closed, and no full OpenShell parity claim is
  supported by the current evidence.

## Name map and data migration

| Current | Target | Migration behavior |
|---|---|---|
| `cautem/cautem-core`, `cautem/cautem-proto`, `cautem/cautem-driver`, `cautem/cautem-runtime` | `cautem/quarshell-core`, `cautem/quarshell-proto`, `cautem/quarshell-driver`, `cautem/quarshell-runtime` | Repositories and Go module declarations are renamed. Preserve old tags; publish and verify new-path tags after the consumer cascade. |
| `cautem/cautem-proxy`, `cautem/cautem-providers`, `cautem/cautem-gateway` | `cautem/quarshell-proxy`, `cautem/quarshell-providers`, `cautem/quarshell-gateway` | Repositories and Go module declarations are renamed. Consumer imports, image references and gateway config still need dependency-ordered migration. |
| `cautem/cautem-sdk`, `cautem/cautem-python` | `cautem/quarshell-sdk`, `cautem/quarshell-python` | Repositories are renamed. Publish new package paths and verify external Go/Python consumers; PyPI publication remains separate. |
| `cautem-cli`, `cmd/cautem`, `cautem-console` | `quarshell-cli`, `cmd/quarshell`, browser app in `quarshell-ui` | Keep `cautem` executable as a warning alias for one release cycle. |
| `cautem/cautem-ui`, `cautem/cautem-display` | `cautem/quarshell-ui`, `cautem/quarshell-display` | Repositories are renamed. Republish package metadata and finish workflow/link migration. |
| `cautem/slogx` (local checkout `cautem-slogx`) | `cautem/quarshell-slogx` | Repository and Go module declaration are renamed; new-path tag and consumer migration remain. |
| `cautem/sandbox.dev` (local checkout `cautem-docs`) | `cautem/quarshell-docs` | Repository is renamed; update the site metadata, links and release documentation. |
| `cautem/.github` (local checkout `cautem-org-profile`) | `cautem/.github` | Keep GitHub's special organization-profile repo name; rebrand its content to Quarshell. |
| `cautem.control.v1`, `@cautem/control-client` | `quarshell.control.v1`, `@quarshell/control-client` | Version the new API package. The old endpoint remains an adapter through the sunset window. |
| `CAUTEM_*` environment variables | `QUARSHELL_*` | New names win when both are set. Read the old name only when the new name is absent; warn once and write migrated values under the new name. Reject conflicting values. |
| `~/.config/cautem`, `~/.local/share/cautem`, `/etc/cautem` | Quarshell equivalents | Copy before first write, preserve permissions, never delete the old directory automatically, and keep a rollback copy. |
| `cautem-*` images, volumes and networks | `quarshell-*` | Discover and adopt resources by recorded IDs/labels. Do not remove resources just because their old name matches. |
| `cautem.internal/*`, `host.cautem.internal` | `quarshell.internal/*`, `host.quarshell.internal` | Recognize both labels and aliases during migration. Create new resources with Quarshell labels; preserve established IDs and endpoints until deletion is explicitly requested. |
| `cautem` package symbols, help text and product-owned config keys | Quarshell names | Rename by ownership-aware inventory, not global replacement. Keep upstream-generated types and protocol strings exact. |
| OpenShell service/package names and `*.openshell.localhost` addresses | unchanged on the compatibility endpoint | These are wire and compatibility identifiers. Quarshell-owned APIs use Quarshell package names. |
| SPDX, copyright notices, third-party module names, historical changelogs | retain attribution/provenance | Update current product references only; never rewrite license ownership or upstream history. |

Configuration precedence is `QUARSHELL_*` → migrated Quarshell file → legacy
`CAUTEM_*` / legacy file → built-in default. If both namespaces define
different values for the same setting, startup fails with the setting name and
source locations; it must not choose silently. The planned migration tool must
create a backup and support dry run, report, apply and rollback. These behaviors
are release gates and are not available until implemented and tested.

## Release sequence

1. **Compatibility baseline and contract.** Keep OpenShell v0.1.4 pinned by
   tag and commit. Check proto drift, the full RPC inventory and every known
   behavior gap. Keep `GetSandboxProviderEnvironment` fail-closed until the
   endpoint-bound credential path is implemented.
2. **Backend SPI.** Keep the transition adapter inside the driver module.
   Migrate Docker and Podman independently, with capability declaration and
   validation before durable writes. Add backend contract tests before switching
   Gateway dispatch.
3. **Docker and Podman acceptance.** Close lifecycle, egress, ownership,
   transfer, TTY, daemon recovery and published-artifact gates on supported
   hosts. Keep a separate evidence record for Docker, Podman rootful and Podman
   rootless.
4. **MicroVM acceptance.** Qualify registry and tar rootfs imports, digest and
   platform binding, host isolation, guest protocol, policy broker, recovery
   and target-host tests. Keep Gateway fail-closed until every required gate
   passes.
   Remove Gateway fail-closed validation only after every mandatory gate passes.
5. **Publish compatible releases at the current paths.** Core/proto →
   runtime/proxy/providers → driver → gateway → SDKs → CLI/UI. Run the published
   consumer gate with `GOWORK=off` before changing any module path.
6. **Complete Quarshell module releases.** Repository names and the eleven Go
   module declarations have changed. Migrate inter-module imports, workspace
   pins, workflow checkouts, package metadata and artifacts in dependency order.
   Publish immutable `github.com/cautem/quarshell-*` tags with SBOMs and signed
   artifacts only after each module's tests and a consumer build without local
   replacements pass. Do not move or rewrite old tags.
7. **Migration candidate.** Publish the Quarshell binary/config/resource
   migration tool and a Cautem-to-Quarshell release candidate. Test fresh install,
   upgrade, old alias behavior, conflicting config, existing resource adoption
   and rollback from the last Cautem release.
8. **Documentation and final gates.** Update English and Russian install,
   quickstart, architecture, security, backend matrix, config reference, CLI
   help/completions, SDK examples and compatibility statement. Run link checks,
   strict docs build, full test matrix, vulnerability checks and published
   artifact smoke.
9. **Release.** Tag/publish each module only after its own gates and downstream
   consumer build pass. State exact OpenShell version and the remaining limits.
   Do not describe full parity while any public method, mode or security gate is
   open.

## Cutover and rollback checks

| Check | Pass condition |
|---|---|
| Config migration | New values take precedence, identical old values migrate, conflicts stop startup with a useful diagnostic. |
| CLI transition | `quarshell` works; `cautem` forwards arguments and environment, prints a migration warning, and preserves exit codes. |
| Existing resources | Old IDs, labels, volumes and networks remain discoverable. Adoption is explicit and cleanup uses recorded ownership. |
| New consumers | Go, Python, npm and binary consumers resolve published Quarshell artifacts without local workspace replacements. |
| Rollback | Restore old config from backup, use the old binary and dependencies, reconnect to existing resources, and preserve gateway state. |
| Compatibility claim | The statement lists the pinned version, tested surfaces, backends and open gaps. |

The repository rename and Go module declarations are already in place under
owner `cautem`; this was an in-place rename, not a transfer. The `.github`
profile repo keeps its reserved name. New module/package releases and data
migration still require the gates above.
