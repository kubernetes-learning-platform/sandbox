# sandbox

> **Everything that runs a sandbox:** the `envmgr` daemon on the Proxmox box, the in-VM `harness`, the `SandboxClient` library that `api` uses, the Proxmox VM template and the lesson images.

Part of [**kubernetes-learning-platform**](https://github.com/kubernetes-learning-platform/docs), an AI tutor with a live Kubernetes lab. Start at the [docs hub](https://github.com/kubernetes-learning-platform/docs) for the big picture.

**Owner:** Filip · **Status:** planned layout, scaffolding lands in M1.

---

## 🧭 How it fits

```mermaid
flowchart LR
    api["api<br/>Hetzner box<br/>uses SandboxClient"]
    envmgr["envmgr<br/>Proxmox host"]
    harness["harness<br/>inside the sandbox VM"]
    k3d["k3d cluster<br/>cp, w1, w2"]

    api -->|"gRPC over the WireGuard mesh"| envmgr
    harness -->|"WebSocket, isolated bridge"| envmgr
    harness -->|"kubectl, docker, watches"| k3d
```

Every session gets its own disposable sandbox: a VM on the Proxmox box (a linked clone of one VM template) running a 3-node k3d cluster. `api` runs on the Hetzner box and never talks to the VM directly. It asks `envmgr` for a sandbox across the WireGuard mesh, and `envmgr` relays frames to and from the `harness` inside it. Secrets and the LLM key never enter the VM. See [ADR 0001](https://github.com/kubernetes-learning-platform/docs/blob/main/adr/0001-hosting-and-harness-transport.md) for why it's split this way.

## 🧩 Projects

| Project | What it is | Ships as |
|---|---|---|
| `KubernetesLearning.EnvManager` | Daemon under systemd on the Proxmox host. Sandbox lifecycle through the Proxmox API, with the warm pool, the lifetime limit and re-adoption after a restart in one reconcile loop. Harness relay. | Binary → GitHub Releases |
| `KubernetesLearning.Harness` | Native AOT program inside every sandbox. Exec, PTY, files, faults, Kubernetes and Docker watchers, topology, traffic probe | Binary → GitHub Releases, baked into the VM template |
| `KubernetesLearning.SandboxClient` | The .NET client `api` uses: `ISandboxSession`, plus `FakeSandboxSession`, a scripted cluster for `api` tests and `web` development | NuGet → GitHub Packages |
| `KubernetesLearning.Sbx` | Dev CLI: `sbx create \| exec \| watch \| destroy` | Not shipped |

## 🔌 envmgr's gRPC service

One service, `EnvironmentManager`, on the Proxmox box's mesh address `10.66.0.2:7070`. A firewall rule accepts it only from the Hetzner box (`10.66.0.1`).

| Method | Purpose |
|---|---|
| `CreateSandbox` | Takes a VM from the warm pool (or clones one from the template). Returns the sandbox id and status. |
| `DestroySandbox` | Stops and deletes the VM, and frees its IP lease and token. |
| `GetSandbox` / `ListSandboxes` | Status. `ListSandboxes` is for debugging and the demo. |
| `WatchSandbox` | Status stream: `booting → cluster_ready → failed / destroyed`. |
| `Connect` | Bidirectional `HarnessFrame` stream that `envmgr` pipes to the harness. |

## 🛡️ Harness

One connection, one frame router:

- **Connection:** reads `Harness__Url`, `Harness__SandboxId` and `Harness__Token` (from `/etc/kubernetes-learning/harness.env`, written by cloud-init, or from env vars in the Docker provider), dials `envmgr`'s WebSocket endpoint and sends `Hello{sandboxId, token, version}`. One `HarnessFrame` per binary message.
- **Handlers:** `Exec`, `Pty`, `File`, `Fault`.
- **Fault ledger:** records every fault (`kill_node`, `delete_pod`, `partition`, `bad_image`) with its undo action, so it can be reverted and the UI can mark it.
- **Watchers:** Kubernetes (nodes, pods, deployments, replicasets, services, endpoints, events) and Docker events for the node containers.
- **Topology:** a normalized graph. A snapshot on connect and on every `SnapshotRequest`, then revisioned diffs.
- **Traffic probe:** a probe pod curls a lesson service, and the harness turns its logs into `TrafficSample` messages about once a second.

## 🖥️ Providers

Both providers use the same harness WebSocket endpoint, `Hello` handshake and one-time token. There is no vsock.

| Provider | Where | How |
|---|---|---|
| `ProxmoxProvider` | The Proxmox box (demo) | A linked clone of the VM template per session, in the `kl-sandboxes` pool, named `kl-sbx-<id>`. cloud-init sets its IP and writes `harness.env`. Proxmox runs the VMs, so they survive `envmgr` restarts. |
| `DockerProvider` | macOS / Windows dev machines | A privileged Docker-in-Docker container running k3d. The harness settings are passed as env vars. |

### Sandbox network (Proxmox)

- Sandbox VMs sit on their own bridge, `vmbr1` (`10.67.0.0/24`), with no physical port.
- `envmgr`'s harness endpoint listens on `10.67.0.1:7071`. It is the **only** address a sandbox can reach. Static nftables rules (in `platform/deploy/proxmox/`) drop traffic to the internet, the home LAN, the mesh and other sandboxes.
- `envmgr` calls the Proxmox API on `localhost` with a token whose role covers only the `kl-sandboxes` pool, the template, its storage and the bridge. The token never leaves the box.

### Reconcile loop

`envmgr` doesn't use Terraform or similar for sandboxes. One loop compares the desired state (`WarmPool__Size` idle sandboxes plus the ones in use) with the VMs in the pool, then clones or deletes until they match. It also deletes VMs past `Sandbox__MaxLifetime` and VMs it has no record of. Proxmox is the source of truth; `envmgr` only stores tokens and IP leases.

### When the mesh link drops

The `Connect` stream to `api` crosses the internet. If it breaks, `envmgr` keeps the harness connection open and the sandbox keeps running. When `api` reattaches, `envmgr` sends the harness a `SnapshotRequest`, so `api` starts again from a full topology instead of stitching diffs across the gap. `SandboxClient` redials with backoff and reports the drop to `api`.

## 📁 Layout

```
sandbox/
├── src/
│   ├── KubernetesLearning.EnvManager/     # Grpc/, Relay/, Providers/{Proxmox,Docker}/, Lifecycle/
│   ├── KubernetesLearning.Harness/        # Transport/, Handlers/, Faults/, Watch/, Topology/, Traffic/, Native/
│   ├── KubernetesLearning.SandboxClient/  # ISandboxSession, SandboxSession, Fakes/
│   └── KubernetesLearning.Sbx/            # dev CLI
├── tests/
│   ├── EnvManager.Tests/
│   ├── Harness.Tests/
│   └── Integration.Tests/                 # Docker provider + real harness + SandboxClient
├── images/
│   ├── base/                              # Packer (Proxmox builder) → the Proxmox VM template
│   └── lessons/
│       ├── demo-web/                      # v1, v2; replies with its pod name
│       └── traffic-probe/                 # tiny curl loop used by the traffic probe
├── Directory.Build.props
└── .github/workflows/                     # ci.yml, release.yml → platform workflows
```

## 🛠️ Local development

No KVM or Proxmox needed. Run `envmgr` with `dotnet run` and `Provider=Docker`. The harness (a linux-arm64 build on Apple Silicon) runs inside a privileged DinD container and dials `envmgr` over WebSocket, exactly as it does on Proxmox. The Proxmox provider is tested on the Proxmox box itself.

- Give Docker Desktop **at least 6 GB** of memory. One sandbox takes 3–4 GB, so a 16 GB Mac fits one at a time.
- envmgr settings are env vars only: `Provider=Proxmox|Docker`, `Grpc__Listen`, `Harness__Listen`, `Proxmox__Url`, `Proxmox__TokenId`, `Proxmox__TokenSecret`, `Proxmox__Node`, `Proxmox__TemplateId`, `Proxmox__Pool`, `Sandbox__Subnet`, `WarmPool__Size`, `StateDir`, `Sandbox__MaxLifetime`.
- Harness settings: `Harness__Url`, `Harness__SandboxId`, `Harness__Token`.

## 🚀 Releases

A tag publishes:
- `envmgr` and `harness` binaries for **linux-x64 and linux-arm64** → GitHub Releases
- `KubernetesLearning.SandboxClient` → GitHub Packages
- a new Proxmox VM template (x64) with the matching harness baked in, built by Packer through `platform`'s `vm-image.yml`. It runs on a self-hosted runner in an LXC on the Proxmox box, because GitHub-hosted runners can't reach Proxmox.
- the lesson images (`demo-web`, `traffic-probe`), multi-arch → GHCR

Nothing is downloaded at runtime: every image the cluster needs is baked into the VM template.

## ⚠️ Read before M1

Most of the [technical risks](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#10-technical-risks-read-before-m1) land in this repo:
- `KubernetesClient.Aot` has **no watch support**, so the harness watches through `kubectl get -w` subprocesses.
- .NET has no PTY API. Use `openpty` through P/Invoke, and **never call `fork()`** from .NET.
- The `api` link crosses the internet. The relay must keep the harness connection open through a drop and resend a full snapshot on reattach. Build this in M1, not as polish.
- `envmgr` runs on the Proxmox host and is the only thing a sandbox can reach. Keep the harness endpoint tiny: token check, then frames.
- Proxmox: `cicustom` snippets need the `snippets` content type on the storage `envmgr` writes to, and linked clones need the template on storage that supports them (LVM-thin, ZFS, or qcow2 on a directory).

## 📚 More

- [ARCHITECTURE §4 Services](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#4-services-and-their-entry-points) (envmgr and harness internals)
- [ARCHITECTURE §6 Key flows](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#6-key-flows)
- [PROJECT-BRIEF](https://github.com/kubernetes-learning-platform/docs/blob/main/PROJECT-BRIEF.md): the original sandbox design

## 📄 License

[MIT](LICENSE)
