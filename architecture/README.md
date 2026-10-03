# Network Architecture Diagram

This folder should contain:
- `network-diagram.png` — visual export of the network diagram (screenshot from your diagramming tool, or an export of the .drawio file below)
- `topology.drawio` — editable source file (draw.io / diagrams.net)

## Reference text diagram

```
                              INTERNET
                                  │
                           ┌──────┴──────┐
                           │  OPNsense   │
                           │ Firewall/RTR│
                           └──────┬──────┘
                                  │
                ┌─────────────────┼─────────────────┐
                │                 │                 │
           LAN (Users)      SERVERS            IT / Admin
          10.10.10.0/24    10.10.20.0/24      10.10.30.0/24
                │                 │                 │
           PC-user1         SRV-Ubuntu1         Admin workstation
           (Windows)        DC-Server1          (SSH/VPN testing)
                             (Nginx, SSH)
                             (AD, DNS)
                                                      │
                                              WireGuard VPN
                                             10.10.99.0/24
                                                      │
                                           PC-Admin-Remote
                                          ("home" workstation)
```

## How to generate the visual diagram

1. Go to [draw.io](https://app.diagrams.net/) (or the diagrams.net app)
2. Recreate the diagram above using standard network shapes (router, switch, servers, workstations)
3. Export as `.png` → save it here as `network-diagram.png`
4. Save the `.drawio` source file in this same folder

Quick alternative: an annotated screenshot of the OPNsense Dashboard (`Interfaces > Overview`) with a legend can also work as a starting point.
