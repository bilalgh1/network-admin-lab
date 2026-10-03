# Firewall Rules — Inter-Network Security Matrix

## General principle: deny by default

OPNsense blocks all traffic on an interface by default until an explicit rule allows it. **LAN is the exception**: two "Default allow" rules (IPv4 and IPv6) are created automatically at install time. Manually added interfaces (SERVERS, IT, WIREGUARD) have **no default rules at all** — confirmed repeatedly by testing a simple ping to the gateway before any rule existed (consistent failure until a Pass rule was created).

## Rule evaluation order

OPNsense evaluates an interface's rules **top to bottom**; the first matching rule applies, later ones are ignored for that packet. A block rule placed *after* a broad "allow all" rule therefore has no effect. This was discovered firsthand while creating the `Block Users → IT` rule (see below) — the rule had to be moved above the LAN "Default allow" rules via drag-and-drop to actually take effect.

## Implemented rule matrix

| # | Interface | Action | Source | Destination | Port/Protocol | Description / Rationale |
|---|---|---|---|---|---|---|
| 1 | LAN | **Block** | LAN network | IT network | any | `Users → IT DENY` — a compromised user workstation should not be able to reach the admin network. Placed first (above the "allow all" rules). |
| 2 | LAN | **Pass** | LAN network | 10.10.20.10 (host) | TCP/80 | `Users → Servers ALLOW`, restricted to the strict minimum: access to the web server on HTTP only, not full access to the server (no SSH, no other port). Principle of least privilege. |
| 3 | LAN | **Pass** (install default) | LAN network | any | any | "Default allow LAN to any" rule — notably enables Internet egress (NAT). Kept in place, with the more specific rules (1, 2) sitting above it to take precedence for the relevant cases. |
| 4 | IT | **Pass** | IT network | any | any | `IT → infrastructure ALLOW` — deliberately broad access for the admin network, consistent with the need to manage/troubleshoot the whole infrastructure. |
| 5 | SERVERS | **Pass** | SERVERS network | any | any | Allows the server to reach out (system updates, external DNS resolution, etc.). Without this rule, no outbound traffic is possible (deny by default). |
| 6 | WIREGUARD | **Pass** | WireGuard net (10.10.99.0/24) | IT network | any | Remote admin VPN restricted to IT only — reinforces the restriction already enforced at the tunnel level (`AllowedIPs`), as defense in depth. |
| 7 | WAN | **Pass** | any | WAN address | UDP/51820 | Allows the WireGuard handshake to come in from outside — necessary for the VPN to work at all (deny by default also applies on WAN). |

*Guests → Internet ALLOW / Guests → Servers DENY / Guests → IT DENY*: not implemented in this iteration, as the Guest VLAN was not created (see [switching/vlan-config.md](../switching/vlan-config.md) for the rationale).

## Validation tests performed

| Test | Expected result | Actual result |
|---|---|---|
| `ping 10.10.30.1` from PC-user1 (Users → IT) | Fail (Block rule) | ✅ 100% loss |
| `ping 8.8.8.8` from PC-user1 (Users → Internet) | Success | ✅ 0% loss |
| `http://10.10.20.10` from PC-user1 (Users → Servers, port 80) | Success | ✅ Nginx page displayed |
| `ssh user@10.10.20.10` from a host on IT | Success (IT → any rule) | ✅ Connection established |
| `ping 10.10.30.1` from PC-Admin-Remote (VPN → IT) | Success | ✅ 0% loss |
| `ping 10.10.20.10` / `ping 10.10.10.1` from PC-Admin-Remote (VPN → Servers/LAN) | Fail | ✅ 100% loss each |

## Issue encountered — rule not saving

See [troubleshooting/incident-05.md](../troubleshooting/incident-05.md): while creating the `Block Users → IT` rule, clicking "Save" appeared to do nothing. Actual cause: the VM had been powered off and back on in between, silently invalidating the web/API session (no explicit error message was shown). A simple page reload fixed it.

## Documented but unused alternative — Floating Rules

If drag-and-drop reordering fails in the UI, a **Floating Rule** with the "Quick" option checked is evaluated before regular interface rules, regardless of their position in the table — a reliable alternative to guarantee a block rule takes precedence over a broad rule, without depending on manual reordering.
