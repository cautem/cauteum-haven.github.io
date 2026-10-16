<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# Gateway

The gateway is the optional control plane: encrypted credential store, sandbox
registry, observation log rings, and policy proposal workflow.

## Run

```bash
cautem gateway ensure
cautem gateway add http://127.0.0.1:7443 --local --name local
cautem gateway select local
cautem gateway info
```

Compose: `cautem-gateway/compose/docker-compose.yml`.

Listen address defaults to `127.0.0.1:7443`.
If you enable `CAUTEM_GATEWAY_ALLOW_UNAUTHENTICATED=1` or pass
`--allow-unauthenticated-users`, the gateway accepts only a loopback listen
address. It refuses to start on a public or LAN address in this mode.
`cautem gateway info` reports `allow_unauthenticated`, and
`cautem status` displays a warning. Keep reverse proxies and port
forwarding in mind: they can expose a loopback listener to other machines.

## Responsibilities

| Surface | Role |
|---------|------|
| Secrets | AES-GCM store; KEK via `CAUTEM_SECRETS_KEK` or `secrets.kek` |
| Providers | Named instances + profile composition metadata |
| Sandboxes | Registry and lifecycle management through RPC; bounded log ring |
| Sidecar | Resolves secrets over `host.cautem.internal` |

## HTTP API reference

The [OpenAPI document](https://github.com/cautem/cautem-gateway/blob/main/api/openapi.yaml)
covers health and authentication bootstrap only. Management operations use
`cautem.control.v1` or the pinned OpenShell RPC contract. SSH byte streams and
the supervisor relay use HTTP upgrade transport and are not REST resource APIs.

## Control client API

The gateway also serves the versioned `cautem.control.v1` Connect API for
the future management UI and generated clients. Its current read surface covers
viewer/capabilities, workspace and sandbox summaries, service/template
summaries, sandbox logs and streams. Runtime create/start/stop/delete actions
use idempotent request IDs and can be reconciled with `GetOperation`; operation
and audit history is available to authorized admins. The private generated
TypeScript client and schema are maintained in the gateway repository under
`api/`. The first read-only browser console is in `ui/`: it uses OIDC
authorization code with PKCE, keeps the access token in memory, and shows the
authorized sandbox inventory and bounded log tail. Configure an OIDC public
client, callback URL, and gateway-matching token audience. Production static
hosting and same-origin proxy wiring are still deployment work. The UI labels
registry state as registry data; it does not claim runtime health. Browser
clients must never receive gateway owner or sandbox supervisor credentials.

The CLI's `sandbox list/get`, sandbox inventory/detail SDK methods, filtered
log snapshots, and live log streams now use this API over native gRPC through
the Go SDK. `logs --all` enumerates caller-visible workspaces and opens at most 24
streams at once. The gateway beta containing the generated Go contract package
must be published before the matching SDK beta.

Provider profile list/show/import/update/delete, partial credential updates,
global and sandbox policy reads/writes, and sandbox policy history use
`cautem.control.v1` over native gRPC. Profile responses preserve cautem's
full YAML schema. These REST routes and their OpenAPI entries have been removed.
Settings, services, workspaces, inference, identity, SSH session management,
and command execution have also moved to RPC; only bootstrap, health, and relay
transport remain on HTTP.

## When required

| Operation | Gateway |
|-----------|---------|
| `--provider` attach | Required |
| Egress proxy + rewrite | Required |
| Plain create `--no-proxy` without providers | Optional (dev) |

## Config

Gateways are recorded in `~/.config/cautem/config.yaml`:

```yaml
current: local
gateways:
  local:
    url: http://127.0.0.1:7443
```

OIDC fields and tokens may appear for authenticated gateways after
`cautem gateway login`.

## Related

- [Credentials](./credentials.md)
- [Policy](./policy.md)
- Compose files: `cautem-gateway/compose/`

## OpenShell gateway TOML

The daemon reads `--config gateway.toml`, then `OPENSHELL_GATEWAY_CONFIG`,
then an optional `$XDG_CONFIG_HOME/openshell/gateway.toml`
(`~/.config/openshell/gateway.toml` fallback). Explicit missing files fail.
Supported values follow flag > environment > file precedence.

Current startup applies the main bind address, installation name in logs,
simple log levels, SSH session TTL, local auth, OIDC/JWKS settings, TLS server
and client certificates, external SNI certificates, mTLS identity, and builtin
or user `provider_profile_sources`. Referenced paths remain literal and
relative to the process working directory. `disable_tls=true` ignores server
certificate material; configuring a client CA at the same time is rejected.

This support is partial. Full OpenShell deployment files still require missing
driver, storage, gateway JWT, OTLP, rate-limit, middleware, interceptor, and
complete readiness/metrics consumers. Supplied unsupported settings fail before the daemon
creates state or opens listeners. The loader also rejects unknown/duplicate
keys, invalid required fields/enums, and a database URL embedded in TOML.
The parser accepts the pinned v0.1.4 schema version 2 and its singular
`compute_driver` selector; the existing `compute_drivers` list remains a local
multi-backend compatibility form. Parsing an upstream fixture does not mean all
of its driver settings are runnable: unsupported backend settings still fail
closed during startup.
Without an OpenShell file, the existing cautem defaults still apply.
The optional `health_bind_address` and `metrics_bind_address` tables, plus
`OPENSHELL_HEALTH_PORT` / `OPENSHELL_METRICS_PORT` and matching port flags,
start separate listeners. Health routes are available at `/healthz`, `/readyz`,
and `/health`; metrics currently exports only a gateway-up gauge. Database
readiness monitoring and full Prometheus instrumentation remain incomplete.

OpenShell OIDC issuer, audience, JWKS cache TTL, and role claim settings feed
the JWT validator. Admin roles may write gateway resources; user roles are
read-only; admins also satisfy the user role. When `scopes_claim` is set,
Control RPC methods that use scope checks map to the existing
`sandbox:read/write`, `provider:read/write`, and `config:read/write` groups;
individual handlers also enforce workspace and admin permissions. This is not
yet a complete per-method descriptor matrix. OIDC signature validation
supports the OpenShell RSA, P-256/P-384 ECDSA, and Ed25519 key families.

TLS can load a client CA and an external certificate pair from the original
gateway TOML. The client CA is optional verification when OIDC is enabled;
mTLS-only authentication requires a verified client certificate. External
certificates are selected by configured exact or wildcard SNI names. Automatic
local PKI generation and watched certificate reload are still pending.

By default, profiles come from both the built-in and stored user catalogs. The
source list can restrict that selection, for example:

```toml
[openshell.gateway]
provider_profile_sources = [{ type = "builtin" }]
```

An interceptor profile source is rejected until the gateway interceptor
runtime and its catalog protocol are implemented.
