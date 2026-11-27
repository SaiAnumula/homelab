# Homelab

[![Validate Compose files](https://github.com/SaiAnumula/homelab/actions/workflows/validate.yml/badge.svg)](https://github.com/SaiAnumula/homelab/actions/workflows/validate.yml)

This repository documents the machines and services I run at home and why they are set up the way they are. It contains sanitized Docker Compose files, [physical and logical network topology diagrams](docs/network-topology.md), and notes on hardware, storage, monitoring, and remote access.

It does not contain credentials, private IP addresses, media, personal data, or application databases.

## Environment

| System | Hardware | Role |
|---|---|---|
| Ubuntu home server | NVIDIA GTX 1050 Ti, SSD plus a 2 TB HDD | Always-on Docker host for Plex, Jellyfin, Immich, Homepage, Wallos, and monitoring |
| Main desktop | Intel Core i5-13600KF, 24 GB DDR4, AMD Radeon RX 9070 (16 GB VRAM), CachyOS | Daily workstation, local AI inference, and larger game servers |
| Laptop node | Reused laptop running Ubuntu | Secondary Docker host (monitoring agents) and smaller game servers |
| Raspberry Pi 5 | 4 GB model | NAS proof of concept |
| Admin devices | Laptops and phones on the tailnet | Remote administration |

## Hardware status

As of November 2026, one of the desktop's memory modules has failed. The desktop dropped from 32 GB to 24 GB of DDR4 and runs on the remaining modules. The RX 9070 still has its full 16 GB of VRAM.

The always-on services are unaffected because Plex, Jellyfin, Immich, and the dashboard run on the home server. On the desktop, jobs that fit in 32 GB now push the system into swap. I size desktop workloads for 24 GB of RAM and 16 GB of VRAM until the module is replaced.

## Services

| Service | Purpose | Host | Availability |
|---|---|---|---|
| Plex | Media streaming for the household | Home server | Always on |
| Jellyfin | Open-source media streaming from the same library | Home server | Always on |
| Immich | Photo backup, search, and face recognition | Home server | Always on |
| Homepage | Tailscale-only landing dashboard for homelab services (not templated here) | Home server | Always on |
| Wallos | Self-hosted subscription tracker behind Tailscale Serve (not templated here) | Home server | Always on |
| [Mission Control](https://github.com/SaiAnumula/homelab-mission-control) | Monitoring dashboard for every node | Home server | Always on |
| Prometheus, Grafana, Loki, Alloy, and Alertmanager | Metrics, dashboards, log aggregation, and alerting for the server and its containers | Home server | Always on |
| Cockpit | Host administration web UI over Tailscale | All Linux hosts | Always on |
| Minecraft and Palworld | Multiplayer game servers, run as systemd services | Desktop or laptop node | Started when needed |

## Networking and access

The Compose files in this repository are sanitized templates of the live deployment, and the two do not match everywhere. In the templates, Immich and Jellyfin publish their web ports on the server's Tailscale address. On the live server, Immich publishes to loopback and Tailscale Serve forwards the tailnet to it, and Jellyfin runs with host networking, listening on every interface behind the host firewall. Plex uses host networking in both the template and the live deployment, for local discovery (GDM); the host firewall limits port 32400 to the `tailscale0` interface, and Plex Remote Access is off. The router forwards no ports for any of them.

I administer every machine over Tailscale too. Approved devices reach SSH and web interfaces through the tailnet.

Game servers use a separate path. When friends outside the tailnet need to join, a Playit tunnel forwards only the game port. The rest of the network and every admin interface stay private.

See [docs/network-topology.md](docs/network-topology.md) for the diagram and trust boundaries.

## Monitoring

[Mission Control](https://github.com/SaiAnumula/homelab-mission-control) is a dashboard I wrote to see every machine on one page. Each Linux host runs a small Python telemetry agent that uses only the standard library. The dashboard on the home server polls each agent and shows:

- CPU, GPU, VRAM, memory, disk, and temperature
- Docker container state and health
- The systemd units I list, including `minecraft.service` and `palworld.service`
- SMART health for each drive, when `smartctl` is available

Agents listen only on the LAN and the tailnet.

Alongside Mission Control, the home server runs a standard observability layer: Prometheus scrapes and stores metrics, Grafana serves the dashboards, Loki with the Alloy collector aggregates container and host logs, and Alertmanager routes alerts. Those containers belong to the live deployment but are not part of the three Compose projects in this repository, so the examples here cover only part of what runs on the server.

## Design decisions

### Containers for each service

Each service has its own Docker Compose file, so I can read, rebuild, or move it on its own. Application data and secrets stay on the server and out of Git.

### GPU transcoding on the home server

The GTX 1050 Ti handles Plex and Jellyfin transcoding through NVENC and NVDEC. Jellyfin mounts the Plex library read-only, so both servers share one copy of the media. Immich uses the same card through CUDA for face recognition and smart search. Both containers request the GPU through the NVIDIA container runtime.

### Storage split between SSD and HDD

Photo originals and the media library live on the 2 TB HDD. Databases, thumbnails, transcoded video, and Plex and Jellyfin metadata live on the SSD, because those are read most often. The paths in the Compose files match my server, so review them before reuse.

### Pinned image versions

Every image in the templates is pinned to a release, and CI validates the rendered Compose output, so a floating tag like `latest` is rejected even when it only appears after an env file is applied. The live server may run different tags than the templates record; the templates hold the intended pins, and upgrades happen when I change a version on purpose, not whenever a container restarts.

### Backups that get tested

[docs/backup-and-restore.md](docs/backup-and-restore.md) lists what gets backed up, the commands, and a restore test that runs on a second machine.

### Reused hardware

The always-on services run on hardware I already had. Bigger game servers run on the desktop only while people are playing.

## Repository layout

```text
homelab/
├── README.md
├── .gitignore
├── .github/workflows/
│   └── validate.yml
├── docs/
│   ├── backup-and-restore.md
│   ├── network-topology.md
│   └── diagrams/
│       ├── logical-topology.svg
│       └── physical-topology.svg
├── immich/
│   ├── docker-compose.yml
│   └── .env.example
├── jellyfin/
│   ├── docker-compose.yml
│   └── .env.example
└── plex/
    ├── docker-compose.yml
    └── .env.example
```

## Prerequisites

- Docker Engine with Docker Compose v2 (`docker compose`).
- NVIDIA Container Toolkit if you keep the GPU transcoding and Immich ML blocks. Remove `deploy.resources.reservations.devices` on CPU-only hosts.
- Tailscale is optional. The templates render with any IPv4 address in `TAILSCALE_IP`.

## Using the examples

1. Change the volume paths and GPU settings to match your system.
2. Copy the `.env.example` in the service folder to `.env`.
3. Replace every placeholder with a strong, unique value, and set `TAILSCALE_IP` to the host's address from `tailscale ip -4`.
4. Run `docker compose config` to check the file. Required variables, such as the Immich database settings, fail with an explicit message when they are unset or empty.
5. Confirm storage paths, permissions, and backups, then start the stack.

The Compose files document my setup. They are not production defaults for every environment.

## Roadmap

- [x] Add service health and resource monitoring ([Mission Control](https://github.com/SaiAnumula/homelab-mission-control))
- [ ] Replace the failed desktop memory module
- [x] Add Compose validation to GitHub Actions
- [x] Document the backup and restore process
- [ ] Run and log the first restore test
- [ ] Add an Ansible playbook for host setup

## Related project

[Raspberry Pi NAS build walkthrough](https://youtu.be/0SLmClxMjXE)
