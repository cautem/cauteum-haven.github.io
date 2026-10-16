<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# ADR 0001: Domain model and compute backend SPI

- **Status:** Accepted; implementation migration is in progress
- **Date:** 2026-10-16
- **Decision owners:** Quarshell maintainers
- **Scope:** compute backends, Gateway orchestration, API compatibility adapters

## Context

The current driver package has a flat legacy `Spec` and `Engine` contract. Those
types combine workload input, host paths, runtime assets, backend settings and
optional operations. They make a new backend inherit assumptions that belong to
an existing container implementation. The product also needs to keep a pinned
external control protocol compatible while allowing its own domain and future
API to evolve independently.

The migration has started in `cautem-driver/driver/backend.go`: it defines a
product-owned request and `Backend` contract, typed backend identity and
capability values, optional extension interfaces and one adapter for legacy
engines. Gateway dispatch resolves a native backend before the compatibility
adapter. Docker, Podman and MicroVM migration and production acceptance are
tracked separately; acceptance of this architecture does not claim those gates
have passed.

## Decision

1. **The product owns the domain contract.** Orchestration uses
   `BackendCreateRequest`, `Backend`, `BackendInfo`, and typed domain values.
   Transport messages, upstream protobuf types and container-engine request
   types stay outside this contract.
2. **Keep the required lifecycle small.** `Backend` owns identity, capability
   discovery, create validation, create, start, stop, delete, list and inspect.
   Optional behavior is expressed by narrow extension interfaces, such as
   `InteractiveExec`, `FileTransfer`, `ServiceForward`, `LogAccess` and runtime
   extensions. A backend may implement only the behavior it can support.
3. **Capabilities and implementations must agree.** Callers obtain an
   extension through `driver.RequireExtension[T]`. The helper checks both the
   advertised capability and the method implementation and returns a typed
   `UnsupportedError` when either is absent. A caller must not silently
   downgrade a requested security or lifecycle behavior.
4. **Validate before committing state.** Gateway resolves and validates a
   backend create request before writing durable lifecycle state or consuming a
   one-use upload token. Secrets are short-lived operation inputs and are not
   copied into durable create records.
5. **Keep one legacy migration seam.** `AdaptLegacyEngine` is the compatibility
   boundary for existing `Engine` implementations. Conversion between
   `BackendCreateRequest` and legacy `Spec` stays in the driver module. Gateway,
   SDK and API schemas do not expose the legacy `Spec`. Native backends are
   selected first; a registered native factory error is returned as-is. The
   legacy adapter is used only when no native factory is registered.
6. **Keep wire compatibility at the edge.** Protocol adapters translate
   external request/response values to and from product domain values. The
   compatibility adapter may retain exact external service, field and package
   names required by its wire contract. Product-owned API and domain names do
   not inherit those identifiers. Unsupported operations are reported as
   typed unsupported results at the protocol boundary.
7. **Keep clients independent.** CLI and browser UI are separate API clients.
   The CLI may contain terminal presentation code and call the SDK; browser UI
   uses a typed browser client and Gateway API. Gateway does not import CLI or
   UI packages, and browser UI does not launch the CLI.

## Contract example

This excerpt uses the implemented Go SPI. A backend-specific configuration type
belongs to that backend package; the shared request does not need a generic
serialized driver configuration field:

```go
type Backend interface {
    ID() BackendID
    Capabilities(context.Context) (BackendInfo, error)
    ValidateCreate(context.Context, BackendCreateRequest) error
    Create(context.Context, BackendCreateRequest) (Handle, error)
    Start(context.Context, core.ID) error
    Stop(context.Context, core.ID) error
    Delete(context.Context, core.ID) error
    List(context.Context) ([]Info, error)
    Inspect(context.Context, string) (Info, error)
}

type BackendCreateRequest struct {
    SandboxID core.ID
    Name      string
    Workspace WorkspaceTarget
    Workload  WorkloadSpec
    Resources ResourceLimits
    Mounts    []Mount
    Network   NetworkSpec
    Runtime   RuntimeAssets
    Config    any // typed by the selected backend, not a wire-format map
}

transfer, err := driver.RequireExtension[driver.FileTransfer](
    ctx, backend, driver.CapabilityFileTransfer, "copy into sandbox",
)
if err != nil {
    return err // preserve Unsupported; do not claim the operation succeeded
}
return transfer.CopyTo(ctx, sandboxID, source, destination)
```

The real `BackendCreateRequest` also includes the sandbox name and unsafe-mount
admission flag. Runtime and network values are ephemeral. Implementations call
`ValidateCreate` before allocating backend resources and reject inputs they
cannot safely represent.

## Module and adapter layout

The target ownership boundaries are:

```text
core/                 domain values, policy, validation and typed errors
proto/                product API schemas and generated product clients
driver/                Backend SPI and Docker, Podman and MicroVM adapters
runtime/               guest init, supervisor and guest protocol
proxy/                 egress enforcement and credential mediation
providers/             provider profiles and credential refresh validation
gateway/               auth, orchestration, durable state and API adapters
sdk/                   Go and Python API clients
cli/                   commands, SDK calls and terminal presentation
ui/                    browser application and typed browser API client
```

These are ownership boundaries, not a requirement that every repository be
split immediately. Current repository and Go module paths still use Cautem
names; their staged rename and release order is documented in the
[rename and release plan](../../reference/renaming-and-release.md).

The Gateway resolver in `cautem-gateway/internal/httpapi/compute.go` first
checks registered native backends, then a legacy engine and its adapter. The
external protocol handler remains a separate adapter in the Gateway. This
means a future backend can own a typed config and implement the domain SPI
without importing CLI packages, a transport schema or the old flat `Spec`.

## Consequences

- A backend can be added behind the SPI without adding backend conditionals to
  API clients or making its configuration part of a shared wire schema.
- Optional features have explicit capability negotiation and can fail closed.
- The compatibility adapter and flat `Spec` remain in the driver during
  migration. They are transitional seams, not the extension point for new
  backends.
- Docker, Podman and MicroVM must migrate independently and pass their runtime
  acceptance gates. The current code and local test results do not establish
  full feature parity or production readiness.
- At the gRPC compatibility boundary, unsupported behavior maps to the
  protocol's unsupported status. Existing wire identifiers remain stable while
  product-owned APIs can use their own namespace.
- Module, package and binary renames wait for configured target remotes,
  published replacement dependencies and a consumer build without local
  workspace overrides. See the release plan for the cutover and rollback order.

## Verification

The decision is reflected in `driver/backend.go`, the Gateway backend resolver,
the compatibility RPC adapter, the RPC inventory and backend contract tests.
Those tests verify the adapter and the explicit unsupported path. Backend
production acceptance remains governed by the feature matrix and open-gap
register; this ADR does not change their status.
