# sandbox

> **Everything that runs a sandbox:** the `envmgr` daemon on the Proxmox box, the in-VM `harness`, the `SandboxClient` library that `api` uses, the Proxmox VM template and the scenario images.

Part of [**kubernetes-learning-platform**](https://github.com/kubernetes-learning-platform/docs), a live Kubernetes lab with an optional AI helper. Start at the [docs hub](https://github.com/kubernetes-learning-platform/docs) for the big picture.

**Owner:** Filip · **Status:** planned layout, scaffolding lands in M1.

---

## 🧭 What this repo does

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/sandbox-dark.svg">
  <img src="assets/sandbox-light.svg" width="100%" alt="On the left, envmgr on the Proxmox host: a gRPC service that api calls, a relay, a reconciler, queue consumers fed by RabbitMQ, and providers that clone VMs. On the right, a sandbox VM: the harness connects to the relay over WebSocket, runs kubectl against the three k3d nodes cp, w1 and w2, and its watchers turn cluster events into topology diffs.">
</picture>

Every learner gets their own disposable computer: a virtual machine with a real 3-node Kubernetes cluster inside. This repo is everything that creates that VM, talks to it, and throws it away again.

It builds two programs and one library:

- **`envmgr`** runs on the machine that hosts the VMs, outside of them. It owns their lifecycle: it keeps a few booted and ready, hands one out when a session starts, deletes it when the session ends, and never runs more than the machine's memory allows. It also passes every message between `api` and the VM.
- **`harness`** runs inside every VM. It is `api`'s hands and eyes in there: it runs commands, provides the terminal, reads and writes files, breaks things on purpose when a scenario calls for it, checks whether a scenario step is solved, and watches the cluster so it can report every change.
- **`SandboxClient`** is the library `api` uses to talk to a sandbox. It hides the connection behind one interface, and includes a fake sandbox that plays a scripted cluster, so `api` and `web` can be developed without any VM.

The split has a reason. The learner gets a real shell in the VM, so the VM is treated as untrusted. It holds no secrets, it can reach exactly one address (envmgr's harness endpoint), and it can be destroyed at any time without losing the session, because the session lives in `api`.

### Follow one sandbox

1. A start request arrives from the work queue. The **consumers** take it only when there is room.
2. The **reconciler** and the **provider** supply a VM: one from the warm pool, or a fresh linked clone of the template. cloud-init gives a clone its address and a one-time token.
3. The VM boots and its cluster comes up. The **harness** dials envmgr's WebSocket endpoint and proves who it is with the token.
4. `api` opens a `Connect` stream on envmgr's **gRPC service**. The **relay** pairs that stream with the harness connection.
5. Frames now flow both ways: commands and keystrokes in, output and cluster changes out. The **watchers** turn Kubernetes and Docker events into one topology snapshot, then diffs.
6. When the session ends, a teardown request deletes the VM, and the reconciler boots a replacement for the pool.

`api` runs on the Hetzner box and never talks to the VM directly. See [ADR 0001](https://github.com/kubernetes-learning-platform/docs/blob/main/adr/0001-hosting-and-harness-transport.md) for why it's split this way, and [ADR 0002](https://github.com/kubernetes-learning-platform/docs/blob/main/adr/0002-learner-driven-scenarios-and-scale-target.md) for the queue. The whole system at a glance is in the [docs hub](https://github.com/kubernetes-learning-platform/docs#readme).

## 🧩 Projects

| Project | What it is | Ships as |
|---|---|---|
| `KubernetesLearning.EnvManager` | Daemon under systemd on the Proxmox host. Sandbox lifecycle through the Proxmox API, with the warm pool, the capacity cap, the lifetime limit and re-adoption after a restart in one reconcile loop. Consumes start and teardown requests. Harness relay. | Binary → GitHub Releases |
| `KubernetesLearning.Harness` | Native AOT program inside every sandbox. Exec, PTY, files, faults, scenario checks, Kubernetes and Docker watchers, topology, traffic probe | Binary → GitHub Releases, baked into the VM template |
| `KubernetesLearning.SandboxClient` | The .NET client `api` uses: `ISandboxSession`, plus `FakeSandboxSession`, a scripted cluster for `api` tests and `web` development | NuGet → GitHub Packages |
| `KubernetesLearning.Sbx` | Dev CLI: `sbx create \| exec \| watch \| destroy` | Not shipped |

## 🔌 envmgr's gRPC service

One service, `EnvironmentManager`, on the Proxmox box's mesh address `10.66.0.2:7070`. A firewall rule accepts it only from the Hetzner box (`10.66.0.1`).

| Method | Purpose |
|---|---|
| `CreateSandbox` | Takes a VM from the warm pool (or clones one from the template). Returns the sandbox id and status. Used by the `sbx` dev CLI; `api` goes through the work queue. |
| `DestroySandbox` | Stops and deletes the VM, and frees its IP lease and token. Used by the `sbx` dev CLI; `api` goes through the work queue. |
| `GetSandbox` / `ListSandboxes` | Status. `ListSandboxes` is for debugging and the demo. |
| `WatchSandbox` | Status stream: `booting → cluster_ready → failed / destroyed`. |
| `Connect` | Bidirectional `HarnessFrame` stream that `envmgr` pipes to the harness. |

## 📬 Work queue

`api` publishes start and teardown requests to RabbitMQ on the Hetzner box, and `envmgr` dials out to it across the mesh. Only these requests use the queue; everything live streams directly ([ADR 0002](https://github.com/kubernetes-learning-platform/docs/blob/main/adr/0002-learner-driven-scenarios-and-scale-target.md)).

- `envmgr` consumes with manual acks. It takes a warm sandbox or clones one, and pauses its consumer when no warm sandbox is free and it's at `Sandbox__MaxConcurrent`, which counts every running VM.
- Redeliveries happen, so it records `requestId` with the sandbox and handles repeats idempotently.
- It publishes `ProvisionResult{requestId, sandboxId, host}`, then acks. `host` is its own gRPC address.
- After repeated failures, a request dead-letters and `api` marks the session failed.
- A second sandbox host later is just a second consumer, with no scheduler to write.
- The harness never touches the broker, and sandboxes can't reach it.

## 🛡️ Harness

One connection, one frame router:

- **Connection:** reads `Harness__Url`, `Harness__SandboxId` and `Harness__Token` (from `/etc/kubernetes-learning/harness.env`, written by cloud-init, or from env vars in the Docker provider), dials `envmgr`'s WebSocket endpoint and sends `Hello{sandboxId, token, version}`. One `HarnessFrame` per binary message.
- **Handlers:** `Exec`, `Pty`, `File`, `Fault`, `Check`. `Check` runs a scenario's `resource`, `http` and `command` checks inside the VM.
- **Fault ledger:** records every fault (`kill_node`, `delete_pod`, `partition`, `bad_image`) with its undo action, so it can be reverted and the UI can mark it.
- **Watchers:** Kubernetes (nodes, pods, deployments, replicasets, services, endpoints, ingresses, events, with more kinds added as scenarios need them) and Docker events for the node containers.
- **Topology:** a normalized graph. A snapshot on connect and on every `SnapshotRequest`, then revisioned diffs.
- **Traffic probe:** a probe pod curls a scenario's service, and the harness turns its logs into `TrafficSample` messages about once a second.

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

`envmgr` doesn't use Terraform or similar for sandboxes. One loop compares the desired state (`WarmPool__Size` idle sandboxes plus the ones in use, never more than `Sandbox__MaxConcurrent` in total) with the VMs in the pool, then clones or deletes until they match. It also deletes VMs past `Sandbox__MaxLifetime` and VMs it has no record of. Proxmox is the source of truth; `envmgr` only stores tokens and IP leases.

### When the mesh link drops

The `Connect` stream to `api` crosses the internet. If it breaks, `envmgr` keeps the harness connection open and the sandbox keeps running. When `api` reattaches, `envmgr` sends the harness a `SnapshotRequest`, so `api` starts again from a full topology instead of stitching diffs across the gap. `SandboxClient` redials with backoff and reports the drop to `api`.

## 📁 Layout

```
sandbox/
├── src/
│   ├── KubernetesLearning.EnvManager/     # Grpc/, Relay/, Queue/, Providers/{Proxmox,Docker}/, Lifecycle/
│   ├── KubernetesLearning.Harness/        # Transport/, Handlers/, Faults/, Watch/, Topology/, Traffic/, Native/
│   ├── KubernetesLearning.SandboxClient/  # ISandboxSession, SandboxSession, Fakes/
│   └── KubernetesLearning.Sbx/            # dev CLI
├── tests/
│   ├── EnvManager.Tests/
│   ├── Harness.Tests/
│   └── Integration.Tests/                 # Docker provider + real harness + SandboxClient
├── images/
│   ├── base/                              # Packer (Proxmox builder) → the Proxmox VM template
│   └── scenarios/                         # custom scenario images, until the `scenarios` repo exists
│       ├── demo-web/                      # v1, v2; replies with its pod name
│       └── traffic-probe/                 # tiny curl loop used by the traffic probe
├── Directory.Build.props
└── .github/workflows/                     # ci.yml, release.yml → platform workflows
```

## 🛠️ Local development

No KVM or Proxmox needed. Run `envmgr` with `dotnet run` and `Provider=Docker`, plus RabbitMQ from `platform`'s dev Compose. The harness (a linux-arm64 build on Apple Silicon) runs inside a privileged DinD container and dials `envmgr` over WebSocket, exactly as it does on Proxmox. The Proxmox provider is tested on the Proxmox box itself.

- Give Docker Desktop **at least 6 GB** of memory. One sandbox takes 3–4 GB, so a 16 GB Mac fits one at a time.
- envmgr settings are env vars only: `Provider=Proxmox|Docker`, `Grpc__Listen`, `Harness__Listen`, `RabbitMq__Url`, `Proxmox__Url`, `Proxmox__TokenId`, `Proxmox__TokenSecret`, `Proxmox__Node`, `Proxmox__TemplateId`, `Proxmox__Pool`, `Sandbox__Subnet`, `Sandbox__MaxConcurrent`, `WarmPool__Size`, `StateDir`, `Sandbox__MaxLifetime`.
- Harness settings: `Harness__Url`, `Harness__SandboxId`, `Harness__Token`.

## 🚀 Releases

A tag publishes:
- `envmgr` and `harness` binaries for **linux-x64 and linux-arm64** → GitHub Releases
- `KubernetesLearning.SandboxClient` → GitHub Packages
- a new Proxmox VM template (x64) with the matching harness baked in, built by Packer through `platform`'s `vm-image.yml`. It runs on a self-hosted runner in an LXC on the Proxmox box, because GitHub-hosted runners can't reach Proxmox.
- the scenario images (`demo-web`, `traffic-probe`), multi-arch → GHCR

Nothing is downloaded at runtime: every image the cluster needs is baked into the VM template.

## ⚠️ Read before M1

Most of the [technical risks](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#10-technical-risks-read-before-m1) land in this repo:
- `KubernetesClient.Aot` has **no watch support**, so the harness watches through `kubectl get -w` subprocesses.
- .NET has no PTY API. Use `openpty` through P/Invoke, and **never call `fork()`** from .NET.
- The `api` link crosses the internet. The relay must keep the harness connection open through a drop and resend a full snapshot on reattach. Build this in M1, not as polish.
- `envmgr` runs on the Proxmox host and is the only thing a sandbox can reach. Keep the harness endpoint tiny: token check, then frames.
- Scenario checks run real commands in every sandbox every few seconds, so each one must stay cheap.
- Proxmox: `cicustom` snippets need the `snippets` content type on the storage `envmgr` writes to, and linked clones need the template on storage that supports them (LVM-thin, ZFS, or qcow2 on a directory).

## 📚 More

- [ARCHITECTURE §4 Services](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#4-services-and-their-entry-points) (envmgr and harness internals)
- [ARCHITECTURE §6 Key flows](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#6-key-flows)
- [PROJECT-BRIEF](https://github.com/kubernetes-learning-platform/docs/blob/main/PROJECT-BRIEF.md): the original sandbox design

## 📄 License

[MIT](LICENSE)
