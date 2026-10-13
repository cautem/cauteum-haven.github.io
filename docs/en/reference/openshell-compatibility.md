# OpenShell compatibility

cautem implements parts of the OpenShell contract on Docker and Podman. The comparison target for this work is OpenShell commit `a0814443f19c07102b19ff09d6ead3d3ba59f9c5`. Compatibility is currently partial. Kubernetes and microVM are outside this comparison.

The gateway has explicit handlers for all 74 public methods in that pinned gRPC service. Tests check method registration, descriptor shape, authentication routing, and bounded dispatch. This establishes that the methods can be called; it does not establish matching results for every input or backend.

The CLI has dispatch entries for the 16 top-level commands in the pinned OpenShell source. This is only a command-name check: it says nothing yet about leaf commands, flags, output, errors, or SDK behavior.

The current work is concentrated in five areas:

| Area | What still needs verification |
| --- | --- |
| Configuration | Defaults, precedence, empty values, validation, and observable effects of each gateway, policy, profile, and template field. |
| CLI and SDK | Commands, flags, environment variables, output, errors, and streaming behavior against pinned OpenShell fixtures. |
| Supervisor | Full event, signal, child-process, exit, and finalization parity across restart and reconnect. With the OpenShell gRPC relay, Linux reports the main workload and `/proc`-observed descendants; `SIGCHLD`/`wait4` promptly detect exits of the main process and supervisor-adopted children, while the 250 ms scan remains a fallback. Relay batching waits up to 25 ms; network time and retries prevent a guaranteed 250 ms delivery bound. Children reaped by their own parent and `exec` replacements can still be missed, and full restart/reconnect parity remains open. |
| gRPC extensions | Per-method behavior, external compute-driver data paths, and middleware/interceptor error, deadline, and cancellation behavior. |
| Published SDK modules | Standalone SDK consumer builds against released module versions, without local workspace replacements. |

The CLI `v0.1.6` release builds against published dependencies with `GOWORK=off`; its Linux, macOS, and Windows release archives and installer checks passed. This resolves the earlier isolated CLI build failure, but does not by itself prove standalone SDK behavior or OpenShell command semantics.

The [gateway compatibility register](https://github.com/cautem/cautem-gateway/blob/main/internal/httpapi/testdata/openshell_compatibility_gaps.json) records covered slices and open work against the pinned commit. An entry marked `verified` applies only to the named slice. The register must be rechecked against the current source and runtime before a broader compatibility claim.

For a local run, check the pinned proto and register with `task compat:pin-check`, `task compat:proto`, and `task compat:gaps`. The gap check rejects deletion of any of the 14 currently tracked requirements, but cannot discover requirements that were never entered. Backend behavior also requires the Docker and Podman test lanes; a green register check does not verify runtime compatibility.
