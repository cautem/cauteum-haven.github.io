<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# Architecture and module boundaries

The accepted decision that governs these contracts is recorded in
[ADR 0001: Domain model and compute backend SPI](decisions/0001-domain-and-backend-spi.md).

Quarshell is the product name used by the target architecture. Existing
published module paths and command names are still in the Cautem namespace;
the staged change is described in [the rename and release plan](../reference/renaming-and-release.md).

```mermaid
flowchart LR
    cli["CLI and terminal TUI"] --> sdk["Go / Python SDK"]
    ui["Browser UI"] --> ts["Typed HTTP / gRPC client"]
    sdk --> api["Gateway API adapters"]
    ts --> api
    api --> domain["Domain and orchestration"]
    domain --> spi["Compute backend SPI"]
    spi --> docker["Docker adapter"]
    spi --> podman["Podman adapter"]
    spi --> vm["MicroVM adapter"]
    domain --> state["Durable state and secret references"]
    docker --> runtime["Sandbox runtime"]
    podman --> runtime
    vm --> runtime
    runtime --> proxy["Policy and credential broker"]
```

## Ownership and dependency direction

| Module | Owns | May depend on |
|---|---|---|
| `quarshell-core` | Domain values, policy, validation, typed errors, capability definitions | Standard library and small domain utilities |
| `quarshell-proto` | Quarshell control API schemas and generated client types | Schema tooling |
| `quarshell-driver` | Compute SPI and Docker, Podman, MicroVM implementations | Core, runtime asset contracts, engine clients |
| `quarshell-runtime` | Guest init, supervisor, SSH/PTY, guest protocol | Core and runtime assets |
| `quarshell-proxy` | Egress enforcement, credential mediation, audit | Core policy and runtime protocol |
| `quarshell-providers` | Provider profiles, credential declarations, refresh validation | Core policy |
| `quarshell-gateway` | API composition, auth, durable state, operation orchestration | Core, proto, driver, runtime, proxy, providers |
| `quarshell-sdk` and `quarshell-python` | Typed API clients and streaming helpers | Public API schemas |
| `quarshell-cli` | Commands, terminal presentation, shell completions | SDK and terminal libraries |
| `quarshell-ui` | Browser presentation and browser-safe client | Typed browser client and gateway API |
| `quarshell-display`, `quarshell-slogx`, `quarshell-docs` | Display, logging utility, documentation | Their narrow public inputs |

The dependency flow is one way: API clients call the gateway; the gateway
selects and validates a backend; backend implementations use the runtime and
host engine. Gateway code does not import CLI or UI packages. The CLI may keep
its terminal TUI as a presentation package and use the SDK for operations. The
browser UI uses a typed API client and never launches the CLI or imports its
internal packages.

The current Cautem release still bundles the older Go `cautem-console` server
from the CLI repository for compatibility. It is a separate SDK client and does
not import the CLI command packages. It is not the target browser app: the
TypeScript PWA lives in the separate `cautem-ui` module and is the only target
Quarshell console. Keep the Go binary through the Cautem transition, complete
the PWA acceptance tracked by plan 2, then remove the legacy binary from CLI
release packaging. The UI module stays TypeScript-only.

This boundary follows the public integration surfaces of current developer
tools. OpenShell v0.1.4 separates its CLI, SDK, server and compute driver
crates. Cursor documents its editor Agent and a CLI with interactive and
non-interactive modes; the CLI supports the editor's modes, while the docs do
not claim both clients share an internal API. OpenAI publishes Codex CLI, SDK
and app-server integrations: the SDK serves programmatic tasks, while
app-server exposes thread, turn, event and approval flows to an application
that owns its interface and workflow. This evidence supports keeping Quarshell
CLI and browser UI as separate clients of a shared public API. It does not
establish private implementation details for Cursor or Codex. See the
[Cursor Agent overview](https://cursor.com/docs/agent/overview),
[Cursor CLI overview](https://cursor.com/docs/cli/overview), and
[Codex app-server integration guidance](https://developers.openai.com/blog/codex-as-a-platform).

## Compute backend SPI

`cautem-driver/driver/backend.go` introduces the product-owned `Backend` SPI.
Its request separates workspace, workload, resource limits, mounts, network,
runtime assets and backend-owned typed configuration. The base interface owns
create validation and lifecycle operations. Optional interfaces include
`InteractiveExec`, `LogAccess`, `FileTransfer`, `ServiceForward`,
`RootfsImporter`, `CredentialRotator`, `RuntimeTokenStarter`,
`RuntimePolicyUpdater`, `RuntimeLaunchSpecRestorer`,
`RuntimeCredentialRefresher`, `RuntimeContractValidator`,
`RuntimeProxyReadiness`, `RuntimeProcessWaiter`, `RuntimeProcessLogs` and
`RuntimeProcessTerminal`, and `RuntimeRootfsTarStager`.
`RequireExtension[T]` checks both the advertised
capability and the implemented method, returning `UnsupportedError` on either
mismatch. Orchestration can validate a create request before it writes durable
state or consumes a one-use upload.

The adapter below is the migration seam for an existing backend. The encoder
is backend-specific, and the flat `Spec` remains inside the driver module while
the Docker and Podman implementations move over incrementally:

```go
backend, err := driver.AdaptComputeDriver(
    driver.BackendID("docker"),
    driver.BackendInfo{
        Capabilities: map[driver.Capability]struct{}{
            driver.CapabilityPTY: {},
            driver.CapabilityInteractiveExec: {},
            driver.CapabilityLogAccess: {},
            driver.CapabilityFileTransfer: {},
        },
    },
    engine,
    func(req driver.BackendCreateRequest) (driver.Spec, error) {
        return driver.Spec{
            Name: req.Name,
            Image: req.Workload.Image,
            Workspace: req.Workspace.HostPath,
            Command: append([]string(nil), req.Workload.Command...),
            TTY: req.Workload.TTY,
            Labels: req.Labels,
        }, nil
    },
)
if err != nil {
    return err
}
if err := registry.Register(backend); err != nil {
    return err
}

selected, err := registry.Get(driver.BackendID("docker"))
if err != nil {
    return err
}
if err := selected.ValidateCreate(ctx, request); err != nil {
    return err // do not persist lifecycle state or consume upload tokens
}
transfer, err := driver.RequireExtension[driver.FileTransfer](
    ctx, selected, driver.CapabilityFileTransfer, "copy",
)
if err != nil {
    return err
}
_ = transfer // call CopyTo or CopyFrom only after this check
handle, err := selected.Create(ctx, request)
```

New implementations can register the product contract directly. The Gateway
resolves a native `Backend` factory before consulting the legacy `Engine`
factory; a registered factory error is returned as-is and never hidden by a
fallback. This lets a backend use its own typed startup config and optional
extensions without implementing the compatibility API or flat `Spec`:

```go
func RegisterCustomVM() error {
    return driver.RegisterBackendWithConfig("custom-vm", func(raw map[string]any) (driver.Backend, error) {
        config, err := parseCustomVMConfig(raw)
        if err != nil {
            return nil, err
        }
        return customvm.NewBackend(config), nil
    })
}

type customVMBackend struct {
    config customvm.Config
}

func (b *customVMBackend) ID() driver.BackendID { return "custom-vm" }

func (b *customVMBackend) Capabilities(context.Context) (driver.BackendInfo, error) {
    return driver.BackendInfo{
        ID: b.ID(),
        Capabilities: map[driver.Capability]struct{}{
            driver.CapabilityInteractiveExec: {},
            driver.CapabilityFileTransfer: {},
        },
    }, nil
}

// Implement ValidateCreate, Create, Start, Stop, Delete, List and Inspect.
// Implement driver.InteractiveExec and driver.FileTransfer only when those
// capabilities have a real, tested implementation.
```

The factory key is the backend ID and its returned `Backend.ID()` must match
that key. Configuration is parsed once when the configured backend is first
selected. Keep upstream request translation in the compatibility adapter;
native backends receive domain requests and can serve Quarshell-owned APIs
without inheriting upstream names or wire types.

Older interfaces such as `SpecValidator`, `GatewayRuntimeValidator`,
`ProxyReadiness`, `RootfsTarStager`, `MainProcessTTY`,
`MainProcessWaiter`, `MainProcessLogAttacher`, `LogOptionsDriver`,
`SandboxPolicyUpdater`, `LaunchSpecRestorer` and
`SandboxRuntimeCredentialRefresher` remain migration seams in the driver
package. The adapter exposes token-aware start, policy delivery, launch-spec
restoration, credential refresh, runtime validation and proxy readiness via
product-owned extensions.
Secret values in runtime or proxy inputs are short-lived request data and must
never be copied into durable create records.

Gateway resolves a registered native SPI backend first, then adapts a legacy
`Engine` for backends that have not migrated yet. It selects the SPI backend
for create preflight, create, start, stop, delete, create rollback, runtime
reconciliation and sandbox recreation.
`RuntimeLaunchSpecRestorer` restores process-local workload settings after a
Gateway restart; `RuntimeCredentialRefresher` refreshes credentials through an
explicit Kubernetes capability; `RuntimeContractValidator` gates VM create on
its Gateway requirements. `RuntimeProxyReadiness` handles the VM and Kubernetes
proxy readiness check, while local containers use base `Backend.Inspect`.
Docker and Podman expose canonical process wait, logs and terminal attachment
through separate process extensions, so control API code does not depend on
the CLI-facing `Engine` interface. Rootfs tar staging is dispatched through
`RuntimeRootfsTarStager` and `CapabilityRootfsTarStaging`.
`BackendCreateRequestFromLegacySpec` and
`LegacySpecFromBackendRequest` are the temporary conversion boundary. The
adapter rejects mounts, distinct workspace IDs and typed config that the legacy
`Spec` cannot represent. VM non-TTY exec now uses the `InteractiveExec`
extension after checking the backend's declared capabilities. Restart uses the
optional `RuntimeTokenStarter` extension when the backend explicitly advertises
`CapabilityTokenAwareStart`; Kubernetes declares this capability, and the
adapter forwards only to its token-aware lifecycle method:

```go
starter, err := driver.RequireExtension[driver.RuntimeTokenStarter](
    ctx, backend, driver.CapabilityTokenAwareStart, "restart with a fresh token",
)
if err == nil {
    err = starter.StartWithToken(ctx, sandboxID, freshToken)
}

policyUpdater, err := driver.RequireExtension[driver.RuntimePolicyUpdater](
    ctx, backend, driver.CapabilityPolicyUpdate, "deliver a runtime policy update",
)
if err == nil {
    err = policyUpdater.UpdateSandboxPolicy(ctx, sandboxID, effectivePolicy)
}

credentialRefresher, err := driver.RequireExtension[driver.RuntimeCredentialRefresher](
    ctx, backend, driver.CapabilityCredentialRefresh, "refresh runtime credentials",
)
if err == nil {
    err = credentialRefresher.RefreshRuntimeCredentials(ctx, sandboxID)
}

platformWatch, err := driver.RequireExtension[driver.PlatformEventStream](
    ctx, backend, driver.CapabilityPlatformEvents, "watch platform events",
)
if err == nil {
    events, err := platformWatch.WatchPlatformEvents(ctx, driver.PlatformEventWatchRequest{
        SandboxID:   sandboxID,
        SandboxName: sandboxName,
    })
    if err == nil {
        for result := range events {
            if result.Err != nil {
                break
            }
            // Handle result.Event.
        }
    }
}
```

`PlatformEventStream` carries backend-native observations without claiming a
durable cursor. The pinned ComputeDriver watch RPC is translated inside the
Gateway compatibility adapter into product-owned event values; its protobuf
types do not enter the watch handler. Backends that can guarantee resumable
generation and sequence semantics should implement `EventStream` instead.
Docker and Podman do not advertise native platform events, so Gateway reports
their persisted lifecycle changes from the sandbox store.

Kubernetes declares `CapabilityTokenAwareStart`, `CapabilityPolicyUpdate`,
`CapabilityCredentialRefresh` and `CapabilityProxyReadiness`; VM declares
`CapabilityGatewayRuntimeValidation`, `CapabilityProxyReadiness` and
`CapabilityRootfsTarStaging`.
Docker and Podman declare process wait/log/terminal capabilities.
Gateway requires the policy capability before delivering an update to its
remote runtime; local container backends continue to use the shared policy
file. Runtime reconciliation, sandbox recreation and launch-spec restoration
use the base SPI and its declared extensions. This is an incremental dispatch
migration, not a completed backend refactor.

## Protocol compatibility boundary

The protocol adapter translates the pinned upstream protobuf contract to
Quarshell domain operations. Upstream package and service names remain intact
on that wire endpoint. Quarshell-owned domain types, configs, logs, CLI help and
new APIs use Quarshell names. Generated protocol packages, wire identifiers,
license attribution, pinned source and compatibility fixtures retain upstream
names where the contract requires them.

The comparison baseline is OpenShell v0.1.4 at commit
`5a7728c304b5528e954e9ae900cf21f84e1a12b1`. It contains 83 methods in the
public service and adds provider-label filtering, operator-only credential
export and stable-placeholder negotiation. A method being registered is not
proof of behavioral compatibility. See the
[compatibility register](../reference/openshell-compatibility.md) and
[runtime acceptance matrix](../reference/runtime-acceptance.md).

## Runtime path

1. The CLI or browser client sends a typed API request to the gateway.
2. The gateway authorizes the workspace, resolves policy and provider
   references, selects a backend, and validates supported fields.
3. The backend creates the sandbox and any required runtime resources.
4. The runtime reports readiness and policy state to the gateway.
5. Network requests pass through the policy enforcement path; credential
   material is released only when the runtime can enforce the matching
   endpoint binding.

Docker and Podman have broader lifecycle coverage than MicroVM. MicroVM
Gateway creation remains fail-closed until the host isolation, image import,
recovery and target-host acceptance gates pass. Kubernetes, browser UI feature
completion and other remaining features are tracked in the internal second
implementation plan.
