# Network topology

The homelab has two ways in from outside the house:

- The Tailscale mesh carries administration traffic between approved devices.
- A Playit tunnel forwards one game-server port to players who are not on the tailnet. It reaches nothing else.

```mermaid
flowchart TD
    Internet((Internet))
    Router["Router and firewall"]
    Tunnel["Playit game tunnel"]

    subgraph LAN["Home LAN"]
        Server["Ubuntu Docker server<br/>Plex, Jellyfin, Immich,<br/>Mission Control"]
        Desktop["Main desktop"]
        Laptop["Laptop game node"]
        Pi["Raspberry Pi NAS PoC"]
        Game["On-demand game server"]
    end

    subgraph Mesh["Private Tailscale mesh"]
        Admin["Approved admin devices"]
    end

    Internet --> Router
    Router --> Server
    Router --> Desktop
    Router --> Laptop
    Router --> Pi
    Internet --> Tunnel
    Tunnel --> Game
    Desktop -.- Game
    Laptop -.- Game
    Server -- "telemetry polling" --> Desktop
    Server -- "telemetry polling" --> Laptop
    Admin -. "encrypted administration" .-> Server
    Admin -. "encrypted administration" .-> Desktop
    Admin -. "encrypted administration" .-> Laptop
    Admin -. "encrypted administration" .-> Pi
```

## Trust boundaries

- Immich and Jellyfin bind only to the server's Tailscale address. Plex uses host networking, so the host firewall limits port 32400 to the `tailscale0` interface.
- SSH and admin interfaces accept connections only from the LAN or the tailnet.
- The Playit tunnel forwards only the port of the game server it is configured for.
- Mission Control telemetry agents listen only on the LAN and the tailnet.
- This public copy leaves out secrets, private addresses, device identifiers, and Tailscale policy.
