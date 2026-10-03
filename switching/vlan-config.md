# Network Segmentation (VLAN equivalents)

## Approach

Since this lab is built in pure virtualization (VirtualBox), segmentation isn't implemented through a physical/virtual switch with 802.1Q trunking, but through **isolated VirtualBox internal networks** ("Internal Network"), each wired to its own OPNsense interface. Each internal network acts as a VLAN: machines connected to it can only talk to each other and to OPNsense — any inter-segment traffic must go through the firewall.

| VirtualBox internal network | Role (VLAN equivalent) | OPNsense interface | Device |
|---|---|---|---|
| `intnet-users` | Users (VLAN 10 equivalent) | LAN | em1 |
| `intnet-servers` | Servers (VLAN 20 equivalent) | SERVERS | em2 |
| `intnet-it` | IT / Admin (VLAN 30 equivalent) | IT | em3 |
| *(Bridged/NAT adapter)* | WAN — Internet uplink | WAN | em0 |

*Assumed and documented limitation in [addressing/ip-plan.md](../addressing/ip-plan.md): 4 networks instead of the originally planned 5, due to the 4-network-adapter-per-VM limit in the VirtualBox GUI.*

## Interface assignment on OPNsense

Interface assignment (`em0`-`em3` mapped to WAN/LAN/SERVERS/IT) is done through the OPNsense console menu (option 1, "Assign interfaces"), then IP address assignment (option 2, "Set interface IP address").

Each interface was then enabled and labeled through the web interface (`Interfaces > [name]`):
- **Enable Interface** checked
- **Description** renamed (LAN, SERVERS, IT) for readability throughout the rest of the OPNsense UI (Firewall menus, Services, etc.)
- **Static IPv4**, address matching the addressing plan

## Point of attention — WAN/LAN swapped on first boot

See [troubleshooting/incident-01.md](../troubleshooting/incident-01.md): on the very first "assign interfaces" step, WAN and LAN were swapped by mistake (the card physically wired to `intnet-users` was declared WAN, and vice versa). OPNsense shows `LAN (em0)` by default on first boot before any configuration, which can be misleading about the actual card order — always verify with `ifconfig` in shell (Diagnostics > Shell) if in doubt, rather than trusting the initial display order.

## VPN interface (WireGuard) as an additional logical segment

In Phase 8, the WireGuard instance (`wg0`) was assigned as a full logical interface (`Interfaces > Assignments`), appearing as `OPT3` and renamed `WIREGUARD`. This interface behaves exactly like the physical interfaces (SERVERS, IT) as far as the firewall is concerned: deny by default until a rule is created. See [services/vpn.md](../services/vpn.md).

## Future improvements

To practice real 802.1Q trunking, VLAN tagging, Spanning Tree Protocol (STP), and EtherChannel/LACP as originally planned in the brief, the logical next step would be to introduce a virtualized Cisco IOS switch (GNS3 or EVE-NG) between OPNsense and the VMs, with a trunk port toward the firewall and access ports per VLAN toward each machine — not implemented in this iteration due to time constraints.
