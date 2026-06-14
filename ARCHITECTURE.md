# Blincus — Architecture

Blincus is a local-first, provider-agnostic development environment platform
built on Incus (LXC successor). It extends the upstream `ublue-os/blincus` CLI
into a full stack: runtime, orchestration, devcontainer compatibility, and
AI-agent-ready scaffolding.

All components are maintained as forks under `Interested-Deving-1896` and
coordinated via [`fork-sync-all`](https://github.com/Interested-Deving-1896/fork-sync-all).

---

## Stack layers

```
┌─────────────────────────────────────────────────────────────────┐
│  Developer UX                                                    │
│  blincus CLI  ·  distrobox (host integration)                   │
│  devcontainers-cli (devcontainer.json compat)                   │
│  claude-devcontainer-bootstrap (AI-agent scaffolding)           │
├─────────────────────────────────────────────────────────────────┤
│  Provider abstraction                                            │
│  devpod (run devcontainer.json on any backend)                  │
│  Providers: local Incus · talos-incus (K8s) · cloud VMs         │
├─────────────────────────────────────────────────────────────────┤
│  Orchestration                                                   │
│  talos-incus  ·  omni-talos  ·  extensions-talos                │
│  (K8s clusters running inside Incus VMs)                        │
├─────────────────────────────────────────────────────────────────┤
│  Container / VM runtime                                          │
│  incus  ·  incus-os  ·  distrobuilder  ·  firecracker-containerd│
├─────────────────────────────────────────────────────────────────┤
│  Image / update layer                                            │
│  rugix (OTA updates)  ·  ashos (atomic snapshots)               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Components

### Developer UX layer

| Repo | Upstream | Role |
|---|---|---|
| `blincus` | `ublue-os/blincus` | CLI that launches full-OS Incus containers from devcontainer.json. The primary user-facing tool. |
| `distrobox` | `89luca89/distrobox` | Run any Linux distro inside a container with full host integration (GUI, home dir, systemd). Complements blincus for host-side tooling. |
| `devcontainers-cli` | `devcontainers/cli` | Reference implementation of the devcontainer spec. Used for feature installation and container lifecycle. |
| `claude-devcontainer-bootstrap` | `niabhail/claude-devcontainer-bootstrap` | Shell scaffolder that generates `.devcontainer/` with MCP server config, Claude Code setup, and corporate cert support. |

### Provider abstraction layer

| Repo | Upstream | Role |
|---|---|---|
| `devpod` | `loft-sh/devpod` | Client-only tool that runs devcontainer.json on any backend (local Docker, K8s, cloud VMs, Incus). Provides the provider plugin model. |

DevPod providers relevant to this stack:
- **Incus provider** — runs devcontainers directly in Incus instances (full OS, not single-process Docker)
- **talos-incus provider** — targets a Talos K8s cluster running inside Incus VMs

### Orchestration layer

| Repo | Upstream | Role |
|---|---|---|
| `talos-incus` | `windsorcli/talos-incus` | Runs Talos (immutable K8s OS) inside Incus VMs. Enables K8s-in-Incus without nested Docker. |
| `omni-talos` | `siderolabs/omni` | Cluster management plane for Talos. Handles provisioning, upgrades, and node lifecycle. |
| `extensions-talos` | `siderolabs/extensions` | System extensions for Talos (GPU drivers, extra kernel modules, etc.). |
| `talos` | `siderolabs/talos` | The Talos OS itself — immutable, API-driven, purpose-built for K8s. |

### Runtime layer

| Repo | Upstream | Role |
|---|---|---|
| `incus` | `lxc/incus` | The container and VM runtime. Successor to LXD. Supports system containers, application containers, and VMs. |
| `incus-os` | `lxc/incus-os` | Minimal immutable OS image purpose-built to run Incus. |
| `distrobuilder` | `lxc/distrobuilder` | Builds LXC/Incus images from YAML specs. Used to produce custom base images for blincus templates. |
| `firecracker-containerd` | `firecracker-microvm/firecracker-containerd` | Micro-VM container runtime. Alternative to full Incus VMs for lightweight isolation. |

### Image / update layer

| Repo | Upstream | Role |
|---|---|---|
| `rugix` | `rugix/rugix` | OTA update system for immutable/embedded Linux. Handles A/B partition updates for incus-os and custom images. |
| `ashos` | `ashos/ashos` | Atomic snapshot-based OS manager. Enables rollback-safe system updates via btrfs/zfs snapshots. |

---

## Devcontainer feature publishing

The `git-platform-clis` devcontainer feature (from `fork-sync-all`) is published
to GHCR and available to all blincus-based environments:

```
ghcr.io/Interested-Deving-1896/fork-sync-all/git-platform-clis:1
```

Installs: `gh`, `glab`, `tea` by default. Optional: `hub`, `bb`, `forgejo-cli`.

---

## OCI artifact flow

```
distrobuilder  →  builds base images
     ↓
incus-os       →  hosts the Incus runtime
     ↓
incus          →  runs system containers / VMs
     ↓
blincus        →  manages devcontainer lifecycle on top of Incus
     ↓
devpod         →  provider abstraction (same devcontainer.json, any backend)
     ↓
GHCR           →  publishes features + images as OCI artifacts
```

---

## Component wiring

Components are declared as git submodules in `config/subtree-manifest.yml` and
managed by `manage-subtrees.sh` (from `fork-sync-all`). Each component lives
under `components/<layer>/<name>/`.

```
components/
├── runtime/
│   ├── incus/
│   ├── incus-os/
│   ├── distrobuilder/
│   └── firecracker-containerd/
├── orchestration/
│   ├── talos/
│   ├── talos-incus/
│   ├── omni-talos/
│   └── extensions-talos/
├── ux/
│   ├── distrobox/
│   ├── devpod/
│   ├── devcontainers-cli/
│   └── claude-devcontainer-bootstrap/
└── images/
    ├── rugix/
    └── ashos/
```

---

## Upstream sync

All component forks are registered in
[`fork-sync-all/registered-imports.json`](https://github.com/Interested-Deving-1896/fork-sync-all/blob/main/registered-imports.json)
and kept current via the `Sync Registered Imports` workflow (daily).

## What is not in scope

- **opencontainers org** — standards body (`runc`, image-spec, runtime-spec).
  Consumed as a dependency, not forked or modified.
- **Gitpod Classic** — sunset. Ona (successor) is the hosted platform used for
  development environments. Not part of this stack.
- **K8s-in-incus (Esteban-Cruz)** — superseded by `talos-incus`.
