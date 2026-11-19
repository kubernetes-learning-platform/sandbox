# AGENTS.md: sandbox

Instructions for AI coding agents working in this repository. `CLAUDE.md` is a symlink to this file, so Claude Code reads the same instructions.

## What this repo is

Everything that runs a sandbox for kubernetes-learning-platform, a live Kubernetes lab with an optional AI helper. Each session gets its own disposable sandbox: three Sysbox node machines running k3s, plus a workstation for the learner's shell and the harness (ADR 0004).

| Project | What it is | Ships as |
|---|---|---|
| `KubernetesLearning.EnvManager` | `envmgr`: a daemon under systemd on the course VM. Sandbox lifecycle (warm pool, capacity cap, lifetime limit, re-adoption) in one reconcile loop, the RabbitMQ consumers, the harness relay, one gRPC service for `api` | Binary → GitHub Releases |
| `KubernetesLearning.Harness` | Native AOT program inside every sandbox: exec, PTY, files, faults, scenario checks, Kubernetes and Docker watchers, topology, traffic probe | Binary → GitHub Releases, baked into the sandbox template |
| `KubernetesLearning.SandboxClient` | The client `api` uses: `ISandboxSession`, plus `FakeSandboxSession`, a scripted cluster | NuGet → GitHub Packages |
| `KubernetesLearning.Sbx` | Dev CLI: `sbx create \| list \| exec \| watch \| destroy` | Not shipped |

Providers: `SysboxProvider` (per session a Docker network, three node containers and a workstation container, all under Sysbox; the Hetzner test VM now, and the course VM if it has no KVM), `QemuProvider` (a QEMU/libvirt VM per session with the nodes inside; the course VM if it has KVM, tested on a home Proxmox server) and `ProcessProvider` (the harness as a plain process, for tests and quick runs). Which one the course VM runs is OPEN until platform#6.

- **Owner:** Filip.
- **Status:** scaffolded (sandbox#1). The four projects are empty and build; there are no tests yet.
- **Design:** `README.md` here, ARCHITECTURE §4.2, §4.3 and §10, and ADR 0003 and ADR 0004 in the `docs` repo. Where they disagree, the `docs` repo wins.

## Layout

```
sandbox/
├── src/
│   ├── KubernetesLearning.EnvManager/     # envmgr: Grpc/, Relay/, Queue/, Providers/{Sysbox,Qemu,Process}/, Lifecycle/
│   ├── KubernetesLearning.Harness/        # harness, Native AOT: Transport/, Handlers/, Faults/, Watch/, Topology/, Traffic/, Native/
│   ├── KubernetesLearning.SandboxClient/  # ISandboxSession, SandboxSession, Fakes/
│   └── KubernetesLearning.Sbx/            # sbx, the dev CLI
├── tests/                                 # from Milestone 1: SandboxClient, EnvManager, Harness and Integration tests
├── images/
│   ├── node/                              # the node machine: systemd, sshd, k3s
│   ├── workstation/                       # the workstation container for the Sysbox provider
│   ├── base/                              # Packer (QEMU builder) → the qcow2 sandbox template
│   └── scenarios/                         # demo-web, traffic-probe
├── scripts/                               # image builds, test VM setup
├── KubernetesLearning.Sandbox.slnx
├── global.json                            # SDK 10.0
├── nuget.config                           # nuget.org, plus GitHub Packages for KubernetesLearning.*
├── Directory.Packages.props               # package versions, including KubernetesLearning.Contracts
├── Directory.Build.props                  # copy from platform/lint, like .editorconfig; .lint-sync records the commit
└── .github/workflows/                     # ci.yml: platform's dotnet-ci, plus the harness AOT publish;
                                           # contracts-bump.yml, lint-sync.yml: the update bot
```

## Build and check

The solution is `KubernetesLearning.Sandbox.slnx`, with SDK 10.0 pinned in `global.json`. There are no tests yet. Before saying a change is done, run what CI runs:

- `dotnet format --verify-no-changes`
- `dotnet build -c Release`
- `../platform/lint/sync.sh --check .`, with `platform` at the commit in `.lint-sync`
- the harness as Native AOT: `dotnet publish src/KubernetesLearning.Harness -c Release -r <rid>`. CI publishes linux-x64 and linux-arm64 on native runners; AOT can't cross-compile from macOS, so locally use `-r osx-arm64` to check the AOT build.

CI (`.github/workflows/ci.yml`) reports `ci / .NET checks`, `Native AOT (linux-x64)` and `Native AOT (linux-arm64)`, all required in the `main` ruleset. Local development runs no VMs: run `envmgr` with `Provider=Process` for quick runs. Real sandboxes run on the team's Hetzner test VM (sandbox#11).

**Lint configs.** `.editorconfig`, `Directory.Build.props` and `.lint-sync` are copies from `platform/lint/`: every warning is an error, `AnalysisLevel=latest-recommended`, and code style is enforced on build and by `dotnet format`. Don't edit them here; change `platform/lint/` instead, and CI fails a hand edit. Build settings for this repo go in the `.csproj` files or a nested `Directory.Build.props` that imports the root one. They were made with `lint/sync.sh --dotnet`. Projects fresh from `dotnet new` don't pass as generated: run `dotnet format` once, because the templates ship CRLF line endings and a byte order mark, then seal the template's types and add their access modifiers.

## Code rules

- **AOT safety in the harness.** No reflection-based serialization: `System.Text.Json` uses source-generated contexts. No `dynamic`, no `Activator.CreateInstance` on unknown types, no `Assembly.Load`. Trim and AOT warnings are errors.
- **Kubernetes watches.** `KubernetesClient.Aot` has no watch support and no kubeconfig loading. Watch through `kubectl get <kind> -A -w --output-watch-events -o json` subprocesses.
- **Processes.** Never call `fork()` from .NET. PTYs use `openpty` through `LibraryImport` P/Invoke.
- **Harness transport.** WebSocket only, for every provider. The first frame is `Hello{sandboxId, token, version}`, and `envmgr` rejects anything before a valid token. Keep the harness endpoint tiny: it runs on the VM host, next to the stack, and is the only address a sandbox can reach.
- **Isolation.** No secrets or API keys enter a sandbox; it gets only its own one-time token. Nothing is downloaded at runtime: every image is baked into the template. Nodes and workstations run under Sysbox, never `--privileged`. The learner has no sudo and no Docker access on the workstation, and only the `harness` user can read the token (ADR 0004 §3).
- **api link drops.** The `Connect` stream breaks whenever `envmgr` restarts. A dropped api stream must not close the harness connection, and a reattach triggers a `SnapshotRequest`.
- **Work queue.** Manual acks, redeliveries handled idempotently by `requestId`, the result published before the ack, never more than `Sandbox__MaxConcurrent`. Only start and teardown use the queue; the harness never touches it.
- **Lifecycle.** libvirt (or Docker) is the source of truth for which sandboxes exist; the state dir holds only tokens and IP leases. After a restart, re-adopt running sandboxes instead of recreating them. Failed creates don't leak sandboxes, qcow2 overlays, IP leases, tokens or cloud-init seeds.
- **Faults** go through the `FaultLedger` with their undo action. **Checks** stay cheap: they run in every sandbox every few seconds.
- **Both architectures.** Binaries ship for linux-x64 and linux-arm64. No x64-only assumptions.
- **Types and config.** Message types come from the `KubernetesLearning.Contracts` package, never hand-written. Config is env vars only; `README.md` lists them.
- **Scripts** go in `scripts/`.

## Releases and dependencies

- Every `feat` or `fix` merge to `main` publishes a release, versioned from the PR title like `contracts` (platform#20): the binaries, the `SandboxClient` package and the scenario images. The sandbox template is built from the private deploy repo, because public repos can't use the self-hosted runner.
- New `contracts` versions arrive as bot commits, `build(deps): bump contracts to X.Y.Z`, from `.github/workflows/contracts-bump.yml`, which calls `platform`'s update bot on `repository_dispatch`. The bot runs `platform`'s `dotnet-ci` on the commit before it pushes to `main`, not the Native AOT publish, which runs only on PRs. If that fails, it opens a PR here instead; fix the code in that PR.
- New lint configs arrive the same way, as `chore: sync lint configs` from `.github/workflows/lint-sync.yml` after a change to `platform/lint/`. If the new rules break the build, the bot opens a PR instead; fix the code in that PR.
- `KubernetesLearning.Contracts` is pinned once, on one line in `Directory.Packages.props` (Central Package Management), and every project references it without a version, so the bot can find and bump it. New packages get their version there too. See `platform`'s README, Update bot.
- **Restoring** needs `GITHUB_PACKAGES_TOKEN`, because GitHub Packages wants a token even for public packages: `gh auth refresh --hostname github.com --scopes read:packages` once, then `export GITHUB_PACKAGES_TOKEN="$(gh auth token)"`. `nuget.config` maps `KubernetesLearning.*` to GitHub Packages and everything else to nuget.org. CI passes its `GITHUB_TOKEN`, so every workflow job that restores grants `packages: read`.

## Commits, branches and PRs

- **Always ask the user before creating a branch, committing or pushing.** Show the commit message when you ask.
- **Commits:** Conventional Commits only, one line: `type: short description`, lowercase, no trailing period. No body. No `Co-Authored-By` or any other attribution.
- **Types:** `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`, `style`. Add `!` for a breaking change.
- **Branches:** use the name in the `Branch:` line of the issue being worked on. Don't make one up. `gh issue develop <number> --name <branch> --checkout` creates it linked to the issue.
- **Where changes land:** every change goes through a PR from the issue's branch. README, `AGENTS.md` and other doc updates go in the PR of the feature they describe, not in separate commits. When a change in another repo makes this repo's docs wrong, fix them in a small `docs:` PR here. Only the `docs` repo commits straight to `main`.
- **PRs:** the title follows the commit format, at most 72 characters; it becomes the squash commit on `main`. The description is one or two plain sentences, plus `Closes #<number>`. Merging needs one approval from someone other than the author, and the branch must be up to date with `main`. Filip is requested as reviewer on every PR (`.github/CODEOWNERS`), and the author is assigned automatically. Only Filip merges.
- **Public repo:** everything committed is public at once. Never commit secrets, real `.env` files or personal data; push protection only catches known token formats.

## Keep docs in sync

When a change touches this repo's design, layout, commands or conventions, update `AGENTS.md`, `README.md` and `.github/copilot-instructions.md` in the same change. If it changes a design fact, update ARCHITECTURE §4.2–4.3 in the `docs` repo too, and mention it to the user.

## Writing style

Plain, professional wording in docs and comments. No emoji, no marketing tone. Files are UTF-8 with LF line endings, following `.editorconfig`.
