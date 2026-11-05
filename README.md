# Homelab

This repository documents the machines and services I run at home and why they are set up the way they are. It contains sanitized Docker Compose files, a network diagram, and notes on hardware, storage, monitoring, and remote access.

It does not contain credentials, private IP addresses, media, personal data, or application databases.

## Environment

| System | Hardware | Role |
|---|---|---|
| Ubuntu home server | NVIDIA GTX 1050 Ti, SSD plus a 2 TB HDD | Always-on Docker host for Plex, Immich, and the monitoring dashboard |
| Main desktop | Intel Core i5-13600KF, 24 GB DDR4, AMD Radeon RX 9070 (16 GB VRAM), CachyOS | Daily workstation and larger game servers |
| Laptop game node | Reused laptop running Linux | Smaller game servers, started when needed |
| Raspberry Pi 5 | 4 GB model | NAS proof of concept |
| Admin devices | Laptops and phones on the tailnet | Remote administration |

## Hardware status

As of November 2026, one of the desktop's memory modules has failed. The desktop dropped from 32 GB to 24 GB of DDR4 and runs on the remaining modules. The RX 9070 still has its full 16 GB of VRAM.

The always-on services are unaffected because Plex, Immich, and the dashboard run on the home server. On the desktop, jobs that fit in 32 GB now push the system into swap. I size desktop workloads for 24 GB of RAM and 16 GB of VRAM until the module is replaced.

## Services

| Service | Purpose | Host | Availability |
|---|---|---|---|
| Plex | Media streaming for the household | Home server | Always on |
| Immich | Photo backup, search, and face recognition | Home server | Always on |
| [Mission Control](https://github.com/SaiAnumula/homelab-mission-control) | Monitoring dashboard for every node | Home server | Always on |
| Minecraft and Palworld | Multiplayer game servers, run as systemd services | Desktop or laptop game node | Started when needed |

## Networking and access

I administer every machine over Tailscale. Approved devices reach SSH and web interfaces through the tailnet, so the router has no port forwards for administration.

Game servers use a separate path. When friends outside the tailnet need to join, a Playit tunnel forwards only the game port. The rest of the network and every admin interface stay private.

See [docs/network-topology.md](docs/network-topology.md) for the diagram and trust boundaries.

## Monitoring

[Mission Control](https://github.com/SaiAnumula/homelab-mission-control) is a dashboard I wrote to see every machine on one page. Each Linux host runs a small Python telemetry agent that uses only the standard library. The dashboard on the home server polls each agent and shows:

- CPU, GPU, VRAM, memory, disk, and temperature
- Docker container state and health
- The systemd units I list, including `minecraft.service` and `palworld.service`
- SMART health for each drive, when `smartctl` is available

Agents listen only on the LAN and the tailnet.

## Design decisions

### Containers for each service

Each service has its own Docker Compose file, so I can read, rebuild, or move it on its own. Application data and secrets stay on the server and out of Git.

### GPU transcoding on the home server

The GTX 1050 Ti handles Plex transcoding through NVENC and NVDEC. Immich uses the same card through CUDA for face recognition and smart search. Both containers request the GPU through the NVIDIA container runtime.

### Storage split between SSD and HDD

Photo originals and the media library live on the 2 TB HDD. Databases, thumbnails, transcoded video, and Plex metadata live on the SSD, because those are read most often. The paths in the Compose files match my server, so review them before reuse.

### Reused hardware

The always-on services run on hardware I already had. Bigger game servers run on the desktop only while people are playing.

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

1. Change the volume paths and GPU settings to match your system.
2. Copy the `.env.example` in the service folder to `.env`.
3. Replace every placeholder with a strong, unique value.
4. Run `docker compose config` to check the file.
5. Confirm storage paths, permissions, and backups, then start the stack.

The Compose files document my setup. They are not production defaults for every environment.

## Roadmap

- [x] Add service health and resource monitoring ([Mission Control](https://github.com/SaiAnumula/homelab-mission-control))
- [ ] Replace the failed desktop memory module
- [ ] Add an Ansible playbook for host setup
- [ ] Document the backup and restore test process
- [ ] Add Compose validation to GitHub Actions

## Related project

[Raspberry Pi NAS build walkthrough](https://youtu.be/0SLmClxMjXE)
