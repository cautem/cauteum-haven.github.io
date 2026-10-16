<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# OpenShell compatibility

Quarshell's compatibility target is the latest stable OpenShell release pinned
for this work: [v0.1.4](https://github.com/NVIDIA/OpenShell/releases/tag/v0.1.4),
commit `5a7728c304b5528e954e9ae900cf21f84e1a12b1`. The source tag is checked out
under `tools/upstream/openshell-v0.1.4`; `UPSTREAM_VERSION` and `UPSTREAM_SHA`
are the baseline for generated schemas, the RPC inventory and the gap register.
This is a partial compatibility target, not a full parity claim.

The inventory contains 83 public `openshell.v1.OpenShell` RPCs, 12 compute
driver RPCs, five credential driver RPCs and related middleware/interceptor
services. Handler registration and proto descriptor checks establish API shape;
they do not establish matching behavior for every method or backend. The
release register records evidence and gaps by scenario.

The versioned feature matrix links each tracked feature to its backend, public
API or configuration surface, test references, evidence level and gate result.
`implemented`, `local_test`, `target_host_test` and `released` describe the
strongest evidence level; `gate_result` records pass, partial, fail or open.
Test-file references and the upstream pin are validated by `task compat:gaps`.
No row claims `released` while the published-consumer gate is failing.

The Docker/Podman config inventory is generated from the pinned public driver
structs and checked against each local decoder. Its `value_behavior` records
omission, explicit empty/zero input and effective runtime behavior for all 54
fields: five rows are `verified` and 49 are `partial`, each with source/test
evidence. Partial rows name observed differences and remaining host/runtime
gates; they do not claim parity. Examples include Cautem image/socket defaults,
Podman health-check lifecycle, proxy boolean presence and rootless ID mapping.
The config corpus remains a release blocker. The inventory also records
unsupported runtime-image settings and the intentional rejection of gateway
client certificates.

## Compatibility status

| Surface | Current status |
|---|---|
| Wire schema | Generated from the pinned v0.1.4 source; drift and method counts are checked. Required upstream service and field names remain unchanged on the compatibility endpoint. |
| Public RPC handlers | Registered. A set of gaps still returns explicit unsupported/precondition results; per-method behavior is incomplete. |
| Docker and Podman | Broad lifecycle, policy, exec, transfer and credential coverage exists. Live target-host acceptance, published binaries, host networking, ownership and outage recovery remain release gates. |
| Logs and platform watch | Remote ComputeDriver watch events are translated inside the compatibility adapter and dispatched through the product `PlatformEventStream` capability, with a sandbox-ID filter. This stream has no durable cursor; resume/reconnect parity remains open. |
| Provider labels | Stored and returned; `ListProviders.label_selector` filters before pagination and rejects invalid selectors. |
| Operator credential export | `GetProviderCredentials` requires a verified mTLS peer certificate with the `operator` OU; bearer credentials are rejected for this method. Deployment and lifetime/refresh parity still need acceptance. |
| Stable placeholders and credential endpoint bindings | Not supported by the current runtime. Lint advertises support as false, and `GetSandboxProviderEnvironment` returns `CREDENTIALS_WITHHELD` instead of plaintext until the runtime can return policy-bound metadata. This is a release blocker. |
| MicroVM | macOS ARM64/HVF live acceptance verifies authenticated OCI import, digest-pinned recovery, image config, PTY/stdin/resize/Ctrl-C and driver recovery. Gateway runtime validation remains fail-closed; Linux KVM and Gateway-to-VM acceptance, workspace mounts/ownership, dedicated host UID, policy status reporting and registry auth variants remain open. |
| CLI and SDK | Top-level dispatch and command-specific help for every sandbox leaf are tested; help fixtures enumerate the accepted flags. The `sandbox create` parser also has acceptance, value-preservation, canonical-argv and malformed-input tests for every documented flag. Other nested parser behavior, output/timeout behavior, Python API and standalone/published consumers remain open. |

Known API gaps include readiness/configuration reports and peer relay paths.
The RPC inventory lists exact methods, stream direction, auth class, handler,
fixtures, tests and evidence. The gap register records remaining work:

- [RPC inventory](https://github.com/cautem/quarshell-gateway/blob/main/internal/httpapi/testdata/openshell_rpc_inventory.json)
- [Feature/API/test/evidence matrix](https://github.com/cautem/quarshell-gateway/blob/main/internal/httpapi/testdata/openshell_compatibility_matrix.json)
- [Compatibility gap register](https://github.com/cautem/quarshell-gateway/blob/main/internal/httpapi/testdata/openshell_compatibility_gaps.json)
- [Pinned Docker/Podman config field inventory](https://github.com/cautem/quarshell-driver/blob/main/internal/configcompat/testdata/openshell-v0.1.4-driver-config.json)
- [Runtime acceptance and test map](runtime-acceptance.md)
- [Architecture and backend SPI](../concepts/architecture.md)

## Recheck the pin

From the workspace root, run:

```sh
task compat:pin-check
task compat:proto
task compat:gaps
```

These checks confirm the checked-out source, generated proto drift, the RPC
inventory counts and the tracked gap baseline. They cannot discover a behavior
that has not yet been added to the inventory. Docker, Podman and MicroVM require
the separate host lanes documented in the [acceptance matrix](runtime-acceptance.md).

At the latest local compatibility check, 12 of 18 tracked requirements were
still open or blocked. Repository names and Go module declarations have moved
to Quarshell on `main`, but new module-path tags and dependent consumer builds
are still pending. CLI names, configuration, environment variables and runtime
resource labels remain in the Cautem namespace during this transition. See the
[rename and release plan](renaming-and-release.md) for the current map and gates.
