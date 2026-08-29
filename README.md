# Homelab

This repository documents the infrastructure I run and the decisions behind it. It includes sanitized Docker Compose configurations, a high-level network diagram, and notes about hardware reuse, storage, and remote administration.

No credentials, private addresses, media, personal data, or live application databases are stored in this repository.

## Environment overview

| System | Primary role |
|---|---|
| Ubuntu server | Always-on Docker host for Plex and Immich |
| Main desktop | Local AI inference and larger on-demand game workloads |
| Laptop game node | Smaller on-demand game-server workloads |
| Raspberry Pi 5 | NAS proof of concept |
| Linux workstations | Administration and daily use |

Hardware details change as the lab evolves, so this repository focuses on architecture and reproducible configuration rather than acting as a fixed inventory.

## Services

| Service | Purpose | Availability |
|---|---|---|
| Plex | Household media streaming | Always on |
| Immich | Self-hosted photo management | Always on |
| Minecraft and Palworld servers | Multiplayer game hosting | Started when needed |
| Hermes-based assistant | Local model and agent experimentation | Run on the main desktop |

## Networking and access

Administrative access uses **Tailscale**, allowing approved devices to reach systems without exposing SSH or management interfaces through router port forwarding.

Public game access is a separate path. When a game server needs to accept players who are not members of the Tailscale network, a managed tunnel such as Playit can expose only the required game service. This does not make the rest of the homelab or its administration interfaces public.

See [docs/network-topology.md](docs/network-topology.md) for the diagram and trust boundaries.

## Design decisions

### Containers for service isolation

Docker Compose keeps service configuration readable and repeatable. Application data and secrets stay outside version control.

### Private administration

Tailscale provides identity-based access between approved devices. Management services are not intentionally exposed to the public internet.

### Reusing hardware

Always-on services run on hardware that is already available and sufficient for the workload. More demanding local-AI and game workloads run on the main desktop only when needed.

### Separating hot data from bulk storage

Large media and photo libraries use higher-capacity storage, while frequently accessed application data and caches can use faster storage. The exact paths in the Compose files reflect one environment and should be reviewed before reuse.

## Repository layout

```text
homelab/
├── README.md
├── .gitignore
├── docs/
│   └── network-topology.md
├── immich/
│   ├── docker-compose.yml
│   └── .env.example
└── plex/
    ├── docker-compose.yml
    └── .env.example
```

## Using the examples

1. Review the volume paths and hardware settings for your own system.
2. Copy the relevant `.env.example` to `.env`.
3. Replace every placeholder with a strong, unique value.
4. Validate the configuration with `docker compose config`.
5. Start the stack only after confirming storage paths, permissions, and backups.

The Compose files are sanitized documentation of my setup, not universal production defaults.

## Roadmap

- [ ] Add an Ansible playbook for repeatable host setup
- [ ] Add service health and resource monitoring
- [ ] Document the backup and restore test process
- [ ] Add configuration validation to GitHub Actions

## Related project

[Raspberry Pi NAS build walkthrough](https://youtu.be/0SLmClxMjXE)

