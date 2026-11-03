# Copilot instructions: sandbox

This repo runs the disposable per-session lab of kubernetes-learning-platform, an AI tutor with a live Kubernetes lab. The full design is in the `docs` repo (ARCHITECTURE.md §4.2–4.3, §10).

## Projects (.NET 10)
- `KubernetesLearning.EnvManager`: host daemon under systemd. VM lifecycle, warm pool, TTL reaper, re-adoption after a restart, harness relay, one gRPC service on a unix socket.
- `KubernetesLearning.Harness`: **Native AOT** program inside every sandbox VM. Exec, PTY, files, faults, Kubernetes and Docker watchers, topology, traffic probe.
- `KubernetesLearning.SandboxClient`: NuGet package `api` uses (`ISandboxSession`, plus `FakeSandboxSession` for tests).
- `KubernetesLearning.Sbx`: dev CLI.
- Message types come from the `KubernetesLearning.Contracts` package. Never hand-write duplicates.

## When reviewing, focus on
- **AOT safety in the harness.** No reflection-based serialization: `System.Text.Json` must use source-generated contexts. No `dynamic`, `Activator.CreateInstance` on unknown types, or `Assembly.Load`. Trim and AOT warnings are errors.
- **Kubernetes watches.** `KubernetesClient.Aot` has no watch support and no kubeconfig loading. Watches use `kubectl get … -w --output-watch-events -o json` subprocesses. Flag attempts to use the client's `Watch` APIs.
- **Process spawning.** Never call `fork()` from .NET. PTYs use `openpty` through `LibraryImport` P/Invoke.
- **vsock.** .NET has no `AddressFamily.Vsock`. Use `(AddressFamily)40` with a custom `EndPoint`. On the host, the socket is a unix socket (`v.sock_5000`, hybrid vsock).
- **Isolation.** Secrets and API keys never enter the VM. The VM has no network route to host services, only vsock to envmgr. Nothing is downloaded at runtime: every image is baked into the VM image.
- **Faults.** Every fault goes through the `FaultLedger` with its undo action, so it can be reverted and marked in the UI. Flag ad-hoc shell commands that break things outside the ledger.
- **Topology.** It's built only from watch events. A snapshot on connect, then diffs with increasing revisions.
- **envmgr restarts.** VMs run in transient systemd units `kl-vm-<id>`, and state lives in `/var/lib/kubernetes-learning`. Code must keep running VMs alive across an envmgr restart and re-adopt them.
- **Concurrency and cleanup.** Cancellation tokens get passed through, sockets and processes are disposed, and failed creates don't leak VMs, taps or units.
- **Both architectures.** Binaries ship for linux-x64 and linux-arm64. Flag x64-only assumptions.
- Config is env vars only (`Provider`, `WarmPool__Size`, `StateDir`, `BaseImage__Path`, `Sandbox__MaxLifetime`).

## Project-wide rules
- PR titles are Conventional Commits (`feat: short description`), one line, at most 72 characters, no trailing period.
- Files are UTF-8 with LF line endings.

## Don't
- Don't comment on formatting that `dotnet format` or `.editorconfig` handles.
- Keep reviews short. Skip praise and summaries of the diff.
