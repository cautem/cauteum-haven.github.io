---
status: soon
---

<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# MicroVM

<div class="ws-soon" markdown>

![Under construction](../../assets/images/under-construction.svg){ width="420" }

**Experimental.** The local driver works, but Gateway use is not enabled yet.

</div>

## Current implementation

The driver creates one libkrun VM per sandbox. It can import a Linux rootfs from
an OCI registry reference or a staged tar archive, build an immutable ext4 base
image, and give each sandbox a writable overlay. Registry references use the
host Docker/Podman credential keychain, including Docker credential helpers. The selected image platform is
`linux/<host-architecture>`; manifest and layer digests are checked before the
rootfs enters the cache.

The driver also has an authenticated guest control channel using versioned v1
JSON over length-prefixed frames; host and guest reject missing or unsupported
versions. `start-workload` returns only after the guest agent observes a private,
versioned readiness marker written by the supervisor after it starts the
workload. The marker path is removed from the child workload environment.
The host rotates the per-sandbox token on each successful Start or recovery.
Rotation state is kept in the private sandbox directory, and a retry after a
lost response reuses the same rotation ID. The channel also provides process
restart, exec, logs, file copy, policy proxy and SSH relay.
Exec supports bounded one-shot commands and a full-duplex PTY session with
stdin, window resize and terminal-generated signals. The Gateway compatibility
adapter has a separate VM interactive-exec path; Gateway sandbox creation still
fails closed until workspace and target-host gates pass.
The image rootfs is cached under the VM state directory. The default cache
quota is 64 GiB; unreferenced images are reclaimed when more space is needed.

## Production readiness

The Linux worker now starts in private user and mount namespaces and makes mount
propagation private before creating its broker or VM. Startup fails when those
namespaces cannot be established or when the Gateway itself runs as host root.
The worker's namespace root still maps to the Gateway account; this is not a
dedicated host UID. macOS currently launches a separate worker process and
rejects host mounts, but it has no equivalent host mount namespace in this
driver. No workspace mount is passed to either platform yet.

The Gateway rejects VM create requests until approved host mounts are opened
and passed by file descriptor under a dedicated host identity, UID/GID mapping
and persistence are accepted, and Linux KVM and macOS ARM64 Gateway lanes pass.

OCI image `Entrypoint`, `Cmd`, `Env`, `User` and `WorkingDir` are applied to the
in-memory VM workload spec. Gateway recovery reloads the image config by the
pinned platform manifest digest. The driver keeps image environment and user
values out of durable state, filters reserved identity variables, and rejects
root identities. Unit tests and macOS ARM64 live acceptance cover OCI config,
private-registry credential-helper auth, digest-pinned recovery, one-shot TTY exec and
image env/workdir inheritance. The same live run covers PTY resize, stdin and
Ctrl-C delivery. Gateway-to-VM and Linux KVM acceptance, dedicated host UID,
workspace mounts/ownership, GPU passthrough and PID limits remain open. These gaps are tracked in the
[runtime acceptance matrix](../reference/runtime-acceptance.md).

## Host setup

Build the worker, static Linux guest agent and runtime helpers with
`cautem-driver/scripts/build-vm-runtime.sh`. Set `CAUTEM_VM_WORKER`,
`CAUTEM_VM_GUEST_AGENT`, `CAUTEM_VM_RUNTIME_DIR` and `CAUTEM_VM_STATE_DIR` for
the target host. Linux requires readable and writable `/dev/kvm`; macOS support
targets Apple Silicon with Hypervisor.framework and a signed worker bundle.
Rootfs preparation also requires e2fsprogs.

Set `CAUTEM_VM_IMAGE_CACHE_MAX_BYTES` to change the default 64 GiB image cache
quota. Registry credentials come from the Docker or Podman credential files
available to the driver process. Uploaded rootfs tar files are limited to 10
GiB compressed and 32 GiB expanded.

For live driver acceptance, follow the VM section in the
[driver README](https://github.com/cautem/cautem-driver#microvm-status). Gateway
and Linux KVM acceptance remain release gates.
