# sandbox

> **Everything that runs a sandbox:** the `envmgr` daemon on the course VM, the `harness` inside each sandbox, the `SandboxClient` library that `api` uses, the sandbox template and the scenario images.

Part of [kubernetes-learning-platform](https://github.com/kubernetes-learning-platform/docs), a live Kubernetes lab with an optional AI helper. The [docs repository](https://github.com/kubernetes-learning-platform/docs) describes the overall system.

**Owner:** Filip · **Status:** scaffolded. The four projects are empty and build in CI; there are no tests yet.

---

## Overview

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/sandbox-dark.svg">
  <img src="assets/sandbox-light.svg" width="100%" alt="On the left, envmgr on the course VM: a gRPC service that api calls, a relay, a reconciler, queue consumers fed by RabbitMQ, and providers that clone sandboxes. On the right, a sandbox: the harness connects to the relay over WebSocket, runs kubectl and ssh against the three Sysbox node machines cp, w1 and w2, and its watchers turn cluster events into topology diffs.">
</picture>

Each session runs in its own disposable sandbox containing a 3-node Kubernetes cluster. The nodes are small machines: Sysbox system containers running systemd, sshd and k3s, so `ssh w2` and `systemctl` work. Next to them, the workstation holds the learner's shell and the harness: the sandbox VM itself when the host has KVM, or a Sysbox container when it doesn't ([ADR 0004](https://github.com/kubernetes-learning-platform/docs/blob/main/adr/0004-node-machines-in-the-sandbox.md)). This repository contains the software that creates the sandbox, communicates with it and deletes it.

- **`envmgr`** runs as a systemd service on the course VM. It manages the sandbox lifecycle: it maintains a pool of ready sandboxes, assigns one when a session starts, deletes it when the session ends, and enforces the host's capacity limit. It also relays all messages between `api` and the sandbox.
- **`harness`** runs on every sandbox's workstation, as its own user. It executes commands, provides the terminal, reads and writes files, injects faults for scenarios, runs scenario checks, and watches the cluster to report changes.
- **`SandboxClient`** is the library through which `api` uses a sandbox. It exposes one interface and includes a fake sandbox for development without a real one.

A sandbox is treated as untrusted, because the learner has shell access to it. It holds no secrets, it can reach only one address (the harness endpoint of `envmgr`), and it can be destroyed without losing the session, because session state is held by `api`.

### Sandbox lifecycle

1. A start request arrives from the work queue. The consumers accept it only when capacity is available.
2. The reconciler and the provider supply a sandbox, either from the warm pool or newly created: a set of Sysbox containers on their own Docker network, or a linked clone of the template VM. Each gets a one-time token.
3. The sandbox boots and the cluster starts. The harness connects to the WebSocket endpoint of `envmgr` and authenticates with the token.
4. `api` opens a `Connect` stream on the gRPC service. The relay pairs this stream with the harness connection.
5. Frames flow in both directions: commands and terminal input to the sandbox, output and cluster changes back. The watchers convert Kubernetes and Docker events into a topology snapshot followed by diffs.
6. When the session ends, a teardown request deletes the sandbox, and the reconciler replenishes the pool.

`api` runs in the stack on the same VM and does not communicate with a sandbox directly. [ADR 0003](https://github.com/kubernetes-learning-platform/docs/blob/main/adr/0003-course-vm-hosting.md) describes the hosting, and [ADR 0002](https://github.com/kubernetes-learning-platform/docs/blob/main/adr/0002-learner-driven-scenarios-and-scale-target.md) explains the queue.

## Projects

| Project | What it is | Ships as |
|---|---|---|
| `KubernetesLearning.EnvManager` | Daemon under systemd on the course VM. Sandbox lifecycle through Docker with Sysbox, or libvirt, with the warm pool, the capacity cap, the lifetime limit and re-adoption after a restart in one reconcile loop. Consumes start and teardown requests. Harness relay. | Binary → GitHub Releases |
| `KubernetesLearning.Harness` | Native AOT program inside every sandbox. Exec, PTY, files, faults, scenario checks, Kubernetes and Docker watchers, topology, traffic probe | Binary → GitHub Releases, baked into the sandbox template |
| `KubernetesLearning.SandboxClient` | The .NET client `api` uses: `ISandboxSession`, plus `FakeSandboxSession`, a scripted cluster for `api` tests and `web` development | NuGet → GitHub Packages |
| `KubernetesLearning.Sbx` | Dev CLI: `sbx create \| list \| exec \| watch \| destroy` | Not shipped |

## envmgr's gRPC service

One service, `EnvironmentManagerService`, on a host address that only the stack can reach (CALL). Plain HTTP/2, since `api` is on the same machine.

| Method | Purpose |
|---|---|
| `CreateSandbox` | Takes a sandbox from the warm pool (or clones one from the template). Returns the sandbox id and status. Used by the `sbx` dev CLI; `api` goes through the work queue. |
| `DestroySandbox` | Stops and deletes the sandbox, and frees its IP lease and token. Used by the `sbx` dev CLI; `api` goes through the work queue. |
| `GetSandbox` / `ListSandboxes` | Status. `ListSandboxes` is for debugging and the demo. |
| `WatchSandbox` | Status stream: `booting → cluster_ready → failed / destroyed`. |
| `Connect` | Bidirectional `HarnessFrame` stream that `envmgr` pipes to the harness. |

## Work queue

`api` publishes start and teardown requests to RabbitMQ in the stack, and `envmgr` connects to it on the same VM with its own credentials. Only these requests use the queue; everything live streams directly ([ADR 0002](https://github.com/kubernetes-learning-platform/docs/blob/main/adr/0002-learner-driven-scenarios-and-scale-target.md)).

- `envmgr` consumes with manual acks. It takes a warm sandbox or clones one, and pauses its consumer when no warm sandbox is free and it's at `Sandbox__MaxConcurrent`, which counts every running sandbox.
- Redeliveries happen, so it records `requestId` with the sandbox and handles repeats idempotently.
- It publishes `ProvisionResult{requestId, sandboxId, host}`, then acks. `host` is its own gRPC address.
- After repeated failures, a request dead-letters and `api` marks the session failed.
- A second sandbox host later is just a second consumer, with no scheduler to write.
- The harness never touches the broker, and sandboxes can't reach it.

## Harness

One connection, one frame router:

- **Connection:** reads `Harness__Url`, `Harness__SandboxId` and `Harness__Token` (from `/etc/kubernetes-learning/harness.env`, readable only by the `harness` user, or from env vars in the process provider), dials `envmgr`'s WebSocket endpoint and sends `Hello{sandboxId, token, version}`. One `HarnessFrame` per binary message.
- **Handlers:** `Exec`, `Pty`, `File`, `Fault`, `Check`. `Exec` runs a command on the workstation, or on a node through `ssh <node>`. `Check` runs a scenario's `resource`, `http` and `command` checks inside the sandbox.
- **Fault ledger:** records every fault (`StopNode`, `StopService`, `DeletePod`, `BadImage`, and later `Partition`) with its undo action, so it can be reverted and the UI can mark it.
- **Watchers:** Kubernetes (nodes, pods, deployments, replicasets, services, endpoints, ingresses, events, with more kinds added as scenarios need them) and Docker events for the node containers.
- **Topology:** a normalized graph. A snapshot on connect and on every `SnapshotRequest`, then revisioned diffs.
- **Traffic probe:** a probe pod curls a scenario's service, and the harness turns its logs into `TrafficSample` messages about once a second.

## Providers

Every provider uses the same harness WebSocket endpoint, `Hello` handshake and one-time token. There is no vsock. The nodes are the same on every path: Sysbox system containers from `images/node/` ([ADR 0004](https://github.com/kubernetes-learning-platform/docs/blob/main/adr/0004-node-machines-in-the-sandbox.md)).

| Provider | Where | How |
|---|---|---|
| `SysboxProvider` | The team's Hetzner test VM now, and the course VM if it has no KVM | Per session: its own Docker network, `cp`, `w1` and `w2` from the node image and a workstation container from `images/workstation/`, all under Sysbox and never `--privileged`. Built first. |
| `QemuProvider` | The course VM if it has KVM. Tested in a nested Ubuntu VM on a small home Proxmox server. | A linked clone per session: a qcow2 overlay on the template image, run by libvirt as `kl-sbx-<id>` and driven through `virsh`. A cloud-init NoCloud seed sets its IP and writes `harness.env`. The nodes run inside it, and libvirt keeps it running through `envmgr` restarts. |
| `ProcessProvider` | Tests and quick local runs | The harness as a plain process, with no nodes |

Which one the course VM runs depends on nested virtualization (OPEN until platform#6).

### Sandbox network

- With the QEMU provider, sandboxes sit on their own isolated libvirt network (`10.67.0.0/24`), with no route to the internet or the host network. With the Sysbox provider, each session gets its own Docker network under the same rule.
- `envmgr`'s harness endpoint listens on `10.67.0.1:7071`. It is the **only** address a sandbox can reach. nftables rules (in `platform/deploy/vm/`) drop traffic to the internet, the Tailscale address, the stack and other sandboxes.

### Reconcile loop

`envmgr` doesn't use Terraform or similar for sandboxes. One loop compares the desired state (`WarmPool__Size` idle sandboxes plus the ones in use, never more than `Sandbox__MaxConcurrent` in total) with the sandboxes the provider lists, then clones or deletes until they match. It also deletes sandboxes past `Sandbox__MaxLifetime` and sandboxes it has no record of. libvirt (or Docker) is the source of truth; `envmgr` only stores tokens and IP leases.

### When the api link drops

The `Connect` stream to `api` breaks when `envmgr` restarts or the stream fails. If it breaks, `envmgr` keeps the harness connection open and the sandbox keeps running. When `api` reattaches, `envmgr` sends the harness a `SnapshotRequest`, so `api` starts again from a full topology instead of stitching diffs across the gap. `SandboxClient` redials with backoff and reports the drop to `api`.

## Layout

```
sandbox/
├── src/
│   ├── KubernetesLearning.EnvManager/     # envmgr: Grpc/, Relay/, Queue/, Providers/{Sysbox,Qemu,Process}/, Lifecycle/
│   ├── KubernetesLearning.Harness/        # harness, Native AOT: Transport/, Handlers/, Faults/, Watch/, Topology/, Traffic/, Native/
│   ├── KubernetesLearning.SandboxClient/  # ISandboxSession, SandboxSession, Fakes/
│   └── KubernetesLearning.Sbx/            # sbx, the dev CLI
├── tests/                                 # from Milestone 1
│   ├── SandboxClient.Tests/
│   ├── EnvManager.Tests/
│   ├── Harness.Tests/
│   └── Integration.Tests/                 # envmgr + real harness + SandboxClient on the process provider
├── images/
│   ├── node/                              # the node machine: systemd, sshd, k3s
│   ├── workstation/                       # the workstation container for the Sysbox provider
│   ├── base/                              # Packer (QEMU builder) → the qcow2 sandbox template
│   └── scenarios/                         # custom scenario images, until the `scenarios` repo exists
│       ├── demo-web/                      # v1, v2; replies with its pod name
│       └── traffic-probe/                 # tiny curl loop used by the traffic probe
├── scripts/                               # image builds, test VM setup
├── KubernetesLearning.Sandbox.slnx
├── global.json                            # SDK 10.0
├── nuget.config                           # nuget.org, plus GitHub Packages for KubernetesLearning.*
├── Directory.Packages.props               # package versions, including KubernetesLearning.Contracts
├── Directory.Build.props                  # copy from platform/lint, like .editorconfig; .lint-sync records the commit
└── .github/workflows/                     # ci.yml (platform's dotnet-ci and the harness AOT publish), contracts-bump.yml and
                                           # lint-sync.yml (the update bot); release.yml with platform#20
```

## Packages

`KubernetesLearning.Contracts` comes from GitHub Packages, which needs a token even to read a public package. `nuget.config` maps `KubernetesLearning.*` to the organization's feed and reads the token from `GITHUB_PACKAGES_TOKEN`; everything else comes from nuget.org. Once, give the GitHub CLI the `read:packages` scope, then export the token before restoring:

```bash
gh auth refresh --hostname github.com --scopes read:packages
```

```bash
export GITHUB_PACKAGES_TOKEN="$(gh auth token)"
```

A classic personal access token with `read:packages` works too. CI passes its own `GITHUB_TOKEN`. The version is pinned once, in `Directory.Packages.props` (Central Package Management), and the projects reference the package without a version, so the update bot can bump it.

## Local development

Set `GITHUB_PACKAGES_TOKEN` first (see [Packages](#packages)). Build with `dotnet build -c Release` and check formatting with `dotnet format --verify-no-changes`. CI also publishes the harness as Native AOT for linux-x64 and linux-arm64; on a Mac, `dotnet publish src/KubernetesLearning.Harness -c Release -r osx-arm64` checks the AOT build. There are no tests yet.

No VMs run on your machine. For quick runs, start `envmgr` with `dotnet run` and `Provider=Process`: the harness runs as a plain process and dials `envmgr` over WebSocket, as it does in a real sandbox, with no nodes. Real sandboxes run on the team's Hetzner test VM through the Sysbox provider (sandbox#11). The QEMU path is tested in a nested Ubuntu VM on a small home Proxmox server with KVM.

- envmgr settings are env vars only: `Provider=Sysbox|Qemu|Process`, `Grpc__Listen`, `Harness__Listen`, `RabbitMq__Url`, `Libvirt__Uri`, `Qemu__TemplateImage`, `Qemu__StoragePool`, `Sandbox__Subnet`, `Sandbox__MaxConcurrent`, `WarmPool__Size`, `StateDir`, `Sandbox__MaxLifetime`.
- Harness settings: `Harness__Url`, `Harness__SandboxId`, `Harness__Token`.

## Releases

Every `feat` or `fix` merge to `main` publishes a release, versioned from the PR title like `contracts` (platform#20):
- `envmgr` and `harness` binaries for **linux-x64 and linux-arm64** → GitHub Releases
- `KubernetesLearning.SandboxClient` → GitHub Packages
- the scenario images (`demo-web`, `traffic-probe`), multi-arch → GHCR

The qcow2 sandbox template (x64), with the matching harness baked in, is built by Packer through `platform`'s `sandbox-template.yml`. It needs KVM, so it runs on the self-hosted runner on the course VM. This repo is public and can't use that runner, so the private deploy repo starts the build. The template stays on the VM, where the provider picks it up.

Nothing is downloaded at runtime: every image the cluster needs is baked into the sandbox template.

## Known technical risks

Most of the [technical risks](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#10-technical-risks-read-before-milestone-1) land in this repo:
- `KubernetesClient.Aot` has **no watch support**, so the harness watches through `kubectl get -w` subprocesses.
- .NET has no PTY API. Use `openpty` through P/Invoke, and **never call `fork()`** from .NET.
- The `api` link breaks whenever `envmgr` restarts. The relay must keep the harness connection open through a drop and resend a full snapshot on reattach. Build this in Milestone 1, not as polish.
- `envmgr` runs on the VM host, next to the stack, and is the only thing a sandbox can reach. Keep the harness endpoint tiny: token check, then frames.
- Scenario checks run real commands in every sandbox every few seconds, so each one must stay cheap.
- Nested virtualization on the course VM is unknown. Without it, the Sysbox provider runs there, and sandboxes share the kernel with the stack. Sysbox running nodes with systemd, sshd and k3s gets checked first (sandbox#12).

## Further reading

- [ARCHITECTURE §4 Services](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#4-services-and-their-entry-points) (envmgr and harness internals)
- [ARCHITECTURE §6 Key flows](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#6-key-flows)
- [PROJECT-BRIEF](https://github.com/kubernetes-learning-platform/docs/blob/main/PROJECT-BRIEF.md): the original sandbox design

## License

[MIT](LICENSE)
