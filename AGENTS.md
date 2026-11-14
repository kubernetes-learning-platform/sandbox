# AGENTS.md: sandbox

Instructions for AI coding agents working in this repository. `CLAUDE.md` is a symlink to this file, so Claude Code reads the same instructions.

## What this repo is

Everything that runs a sandbox for kubernetes-learning-platform, a live Kubernetes lab with an optional AI helper. Each session gets its own disposable sandbox with a 3-node k3d cluster inside.

| Project | What it is | Ships as |
|---|---|---|
| `KubernetesLearning.EnvManager` | `envmgr`: a daemon under systemd on the course VM. Sandbox lifecycle (warm pool, capacity cap, lifetime limit, re-adoption) in one reconcile loop, the RabbitMQ consumers, the harness relay, one gRPC service for `api` | Binary → GitHub Releases |
| `KubernetesLearning.Harness` | Native AOT program inside every sandbox: exec, PTY, files, faults, scenario checks, Kubernetes and Docker watchers, topology, traffic probe | Binary → GitHub Releases, baked into the sandbox template |
| `KubernetesLearning.SandboxClient` | The client `api` uses: `ISandboxSession`, plus `FakeSandboxSession`, a scripted cluster | NuGet → GitHub Packages |
| `KubernetesLearning.Sbx` | Dev CLI: `sbx create \| exec \| watch \| destroy` | Not shipped |

Providers: `DockerProvider` (Docker-in-Docker + k3d; privileged in dev, under Sysbox on the course VM) and `QemuProvider` (a QEMU/libvirt VM per session, used if the course VM supports nested virtualization). Which one runs on the VM is OPEN until platform#6.

- **Owner:** Filip.
- **Status:** planned layout. sandbox#1 scaffolds the solution.
- **Design:** `README.md` here, ARCHITECTURE §4.2, §4.3 and §10, and ADR 0003 in the `docs` repo. Where they disagree, the `docs` repo wins.

## Layout

```
sandbox/
├── src/
│   ├── KubernetesLearning.EnvManager/     # Grpc/, Relay/, Queue/, Providers/{Qemu,Docker}/, Lifecycle/
│   ├── KubernetesLearning.Harness/        # Transport/, Handlers/, Faults/, Watch/, Topology/, Traffic/, Native/
│   ├── KubernetesLearning.SandboxClient/  # ISandboxSession, SandboxSession, Fakes/
│   └── KubernetesLearning.Sbx/            # dev CLI
├── tests/                                 # EnvManager.Tests, Harness.Tests, Integration.Tests
├── images/
│   ├── base/                              # Packer (QEMU builder) → the qcow2 sandbox template
│   └── scenarios/                         # demo-web, traffic-probe
├── Directory.Build.props
└── .github/workflows/                     # call platform's reusable workflows
```

## Build and check

Not set up yet; sandbox#1 adds the exact commands here. The checks CI will run:

- `dotnet format --verify-no-changes`
- `dotnet build` and `dotnet test`
- the harness published as Native AOT for linux-x64 and linux-arm64. AOT can't cross-compile from macOS, so this runs on Linux.

Run them before saying a change is done. Local development needs no KVM: run `envmgr` with `Provider=Docker` and RabbitMQ from `platform`'s dev Compose. Give Docker Desktop at least 6 GB of memory.

## Code rules

- **AOT safety in the harness.** No reflection-based serialization: `System.Text.Json` uses source-generated contexts. No `dynamic`, no `Activator.CreateInstance` on unknown types, no `Assembly.Load`. Trim and AOT warnings are errors.
- **Kubernetes watches.** `KubernetesClient.Aot` has no watch support and no kubeconfig loading. Watch through `kubectl get <kind> -A -w --output-watch-events -o json` subprocesses.
- **Processes.** Never call `fork()` from .NET. PTYs use `openpty` through `LibraryImport` P/Invoke.
- **Harness transport.** WebSocket only, for every provider. The first frame is `Hello{sandboxId, token, version}`, and `envmgr` rejects anything before a valid token. Keep the harness endpoint tiny: it runs on the VM host, next to the stack, and is the only address a sandbox can reach.
- **Isolation.** No secrets or API keys enter a sandbox; it gets only its own one-time token. Nothing is downloaded at runtime: every image is baked into the template. On the container path, sandboxes run under Sysbox, never `--privileged`.
- **api link drops.** The `Connect` stream breaks whenever `envmgr` restarts. A dropped api stream must not close the harness connection, and a reattach triggers a `SnapshotRequest`.
- **Work queue.** Manual acks, redeliveries handled idempotently by `requestId`, the result published before the ack, never more than `Sandbox__MaxConcurrent`. Only start and teardown use the queue; the harness never touches it.
- **Lifecycle.** libvirt (or Docker) is the source of truth for which sandboxes exist; the state dir holds only tokens and IP leases. After a restart, re-adopt running sandboxes instead of recreating them. Failed creates don't leak sandboxes, qcow2 overlays, IP leases, tokens or cloud-init seeds.
- **Faults** go through the `FaultLedger` with their undo action. **Checks** stay cheap: they run in every sandbox every few seconds.
- **Both architectures.** Binaries ship for linux-x64 and linux-arm64. No x64-only assumptions.
- **Types and config.** Message types come from the `KubernetesLearning.Contracts` package, never hand-written. Config is env vars only; `README.md` lists them.

## Releases and dependencies

- A tag in this repo publishes the binaries, the `SandboxClient` package, the sandbox template and the scenario images.
- New `contracts` versions arrive as bot commits, `build(deps): bump contracts to X.Y.Z`. If the build fails after a bump, the bot opens a PR here instead; fix the code in that PR.

## Commits, branches and PRs

- **Always ask the user before creating a branch, committing or pushing.** Show the commit message when you ask.
- **Commits:** Conventional Commits only, one line: `type: short description`, lowercase, no trailing period. No body. No `Co-Authored-By` or any other attribution.
- **Types:** `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`, `style`. Add `!` for a breaking change.
- **Branches:** use the name in the `Branch:` line of the issue being worked on. Don't make one up. `gh issue develop <number> --name <branch> --checkout` creates it linked to the issue.
- **Where changes land:** while the repo is still empty, documentation goes straight to `main`. From the setup issues on (sandbox#1), every change goes through a PR from the issue's branch.
- **PRs:** the title follows the commit format, at most 72 characters; it becomes the squash commit on `main`. The description is one or two plain sentences, plus `Closes #<number>`. Only Filip merges.

## Keep docs in sync

When a change touches this repo's design, layout, commands or conventions, update `AGENTS.md`, `README.md` and `.github/copilot-instructions.md` in the same change. If it changes a design fact, update ARCHITECTURE §4.2–4.3 in the `docs` repo too, and mention it to the user.

## Writing style

Plain, professional wording in docs and comments. No emoji, no marketing tone. Files are UTF-8 with LF line endings, following `.editorconfig`.
