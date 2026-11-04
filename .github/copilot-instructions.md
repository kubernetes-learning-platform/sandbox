# Copilot instructions: sandbox

This repo runs the disposable per-session lab of kubernetes-learning-platform, an AI tutor with a live Kubernetes lab. The full design is in the `docs` repo (ARCHITECTURE.md §4.2–4.3, §10).

## Projects (.NET 10)
- `KubernetesLearning.EnvManager`: daemon under systemd on the Proxmox host. Proxmox and Docker (dev) providers, one reconcile loop (warm pool, capacity cap, lifetime limit, re-adoption), a RabbitMQ consumer, the harness relay, and one gRPC service on the mesh.
- `KubernetesLearning.Harness`: **Native AOT** program inside every sandbox VM. Exec, PTY, files, faults, scenario checks, Kubernetes and Docker watchers, topology, traffic probe.
- `KubernetesLearning.SandboxClient`: NuGet package `api` uses (`ISandboxSession`, plus `FakeSandboxSession` for tests).
- `KubernetesLearning.Sbx`: dev CLI.
- Message types come from the `KubernetesLearning.Contracts` package. Never hand-write duplicates.

## When reviewing, focus on
- **AOT safety in the harness.** No reflection-based serialization: `System.Text.Json` must use source-generated contexts. No `dynamic`, `Activator.CreateInstance` on unknown types, or `Assembly.Load`. Trim and AOT warnings are errors.
- **Kubernetes watches.** `KubernetesClient.Aot` has no watch support and no kubeconfig loading. Watches use `kubectl get … -w --output-watch-events -o json` subprocesses. Flag attempts to use the client's `Watch` APIs.
- **Process spawning.** Never call `fork()` from .NET. PTYs use `openpty` through `LibraryImport` P/Invoke.
- **Harness transport.** WebSocket only, for every provider (no vsock). The first frame is `Hello{sandboxId, token, version}`, and envmgr rejects anything before a valid token. Keep the harness endpoint small: it runs on the Proxmox host and is the only address a sandbox can reach.
- **Isolation.** Secrets and API keys never enter the VM; it gets only its own one-time token. The sandbox bridge reaches nothing but envmgr's harness endpoint. Nothing is downloaded at runtime: every image is baked into the VM template.
- **Mesh link drops.** The api `Connect` stream crosses the internet. A dropped api stream must not close the harness connection, and a reattach must trigger a `SnapshotRequest`. Flag code that ties the harness connection's lifetime to the api stream.
- **Work queue.** Manual acks, and redeliveries handled idempotently by `requestId`. Publish the result before acking. Never exceed `Sandbox__MaxConcurrent`. Only start and teardown use the queue, and the harness never touches it.
- **Checks stay cheap.** They run in every sandbox every few seconds. Flag slow or heavy checks.
- **Proxmox API.** Only through `ProxmoxApiClient`, with the pool-scoped token. Wait for Proxmox task completion (UPID) before the next step. The token is never logged.
- **Faults.** Every fault goes through the `FaultLedger` with its undo action, so it can be reverted and marked in the UI. Flag ad-hoc shell commands that break things outside the ledger.
- **Topology.** It's built only from watch events. A snapshot on connect, then diffs with increasing revisions.
- **envmgr restarts.** Proxmox is the source of truth for which VMs exist (pool `kl-sandboxes`); the state dir holds only tokens and IP leases. The reconcile loop must re-adopt running VMs, not recreate them.
- **Concurrency and cleanup.** Cancellation tokens get passed through, sockets and processes are disposed, and failed creates don't leak VMs, IP leases, tokens or cloud-init snippets.
- **Both architectures.** Binaries ship for linux-x64 and linux-arm64. Flag x64-only assumptions.
- Config is env vars only; the README lists them.

## Project-wide rules
- PR titles are Conventional Commits (`feat: short description`), one line, at most 72 characters, no trailing period.
- Files are UTF-8 with LF line endings.

## Don't
- Don't comment on formatting that `dotnet format` or `.editorconfig` handles.
- Keep reviews short. Skip praise and summaries of the diff.
