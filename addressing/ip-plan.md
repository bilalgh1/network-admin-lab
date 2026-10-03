# IP Addressing Plan

## Context and design constraint

The original brief called for 5 networks (Users, Servers, IT, Guests, Management). The chosen virtualization environment (VirtualBox) limits the GUI to **4 network adapters per VM**. Rather than working around this limit from the start (via `VBoxManage` CLI, or a different hypervisor), the core infrastructure was built on **4 networks** (WAN + 3 internal networks), with the Guest VLAN extension documented as a future improvement instead of a hard blocker from day one.

A 5th network (WireGuard VPN, `10.10.99.0/24`) was added in Phase 8 by assigning the WireGuard interface as an additional logical interface on OPNsense — this approach doesn't consume a physical VirtualBox network adapter, so no new limit was hit at that point.

## Addressing table

| Network | Function | Range | Mask | Gateway | DHCP range |
|---|---|---|---|---|---|
| LAN | Users (employee workstations) | 10.10.10.0/24 | /24 (255.255.255.0) | 10.10.10.1 | 10.10.10.100 → 10.10.10.200 |
| SERVERS | Application servers | 10.10.20.0/24 | /24 | 10.10.20.1 | *(none — static IPs)* |
| IT | Admin workstations and access | 10.10.30.0/24 | /24 | 10.10.30.1 | 10.10.30.50 → 10.10.30.100 |
| WireGuard (VPN) | Remote admin access | 10.10.99.0/24 | /24 | 10.10.99.1 | *(peers configured individually)* |

## Rationale behind the design

- **A `/24` per network** (254 usable addresses) is more than enough for a ~50-employee SMB spread across several segments, while keeping subnet math simple — no over-engineering for this context.
- **SERVERS deliberately has no DHCP range**: servers need a stable, predictable address since several services depend directly on it (DNS resolution, firewall rules pointing at a specific IP, potential GPOs). An address that changed on every reboot would silently break these dependencies.
- **IT has a smaller DHCP range** (50 addresses, 50-100): this network is meant for a limited number of admin workstations/devices, no need for a range as large as Users.
- **The WireGuard network (10.10.99.0/24)** is kept isolated from the other internal ranges so the VPN access restriction (traffic scoped to the IT network only via the `AllowedIPs` field) is unambiguous — the `.99` prefix is a common convention for a management/VPN network, consistent with the example given in the original brief.

## Static addresses assigned

| Machine | Role | Network | Static IP |
|---|---|---|---|
| OPNsense-FW | Firewall / router | LAN / SERVERS / IT (gateway for each) | .1 on each network |
| SRV-Ubuntu1 | Web server (Nginx) + SSH | SERVERS | 10.10.20.10 |
| DC-Server1 | Domain controller (AD + DNS) | SERVERS | 10.10.20.20 |

Client workstations (`PC-user1`) and VPN peers get their address dynamically (DHCP for LAN/IT, tunnel configuration for WireGuard).
