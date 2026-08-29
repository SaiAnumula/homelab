# Network topology

The homelab uses two distinct remote-access paths:

- **Tailscale mesh:** private administration and access between approved devices.
- **Game tunnel:** optional public access to a specific game-server port without exposing management services.

```mermaid
flowchart TD
    Internet((Internet))
    Router["Router and firewall"]
    Tunnel["Managed game tunnel"]
    Game["On-demand game server"]

    subgraph LAN["Home LAN"]
        Server["Ubuntu Docker server"]
        Desktop["Main desktop"]
        Pi["Raspberry Pi NAS PoC"]
    end

    subgraph Mesh["Private Tailscale mesh"]
        Admin["Approved admin devices"]
    end

    Internet --> Router
    Router --> Server
    Router --> Desktop
    Router --> Pi
    Internet --> Tunnel
    Tunnel --> Game
    Admin -. "encrypted administration" .-> Server
    Admin -. "encrypted administration" .-> Desktop
    Admin -. "encrypted administration" .-> Pi
```

## Trust boundaries

- SSH and service administration are intended to remain reachable only from the LAN or Tailscale mesh.
- The public tunnel is limited to the game service it is configured to forward.
- Secrets, private addresses, device identifiers, and Tailscale configuration are intentionally omitted from this public documentation.

