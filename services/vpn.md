# Remote Access VPN (WireGuard)

## Scenario

An administrator works from home and needs to access the IT network for remote administration. A standard VPN user, on the other hand, should **not** have access to the entire company network (SERVERS and LAN must stay unreachable).

## Technology choice

**WireGuard** was chosen over OpenVPN or IPsec: a modern protocol, native to OPNsense, simpler to configure (a key pair plus a few fields), and it comes with a built-in tool ("Peer generator") that automatically generates the full client configuration.

## Architecture

| Element | Value |
|---|---|
| VPN network | 10.10.99.0/24 |
| Server instance (OPNsense) | `wg0`, Listen port UDP/51820, Tunnel Address 10.10.99.1/24 |
| Assigned logical interface | OPT3 → renamed `WIREGUARD` |
| Tested client | `PC-Admin-Remote`, an isolated VM simulating a "home" workstation (outside all internal lab networks) |

## Security restriction — defense in depth

The "a standard VPN user doesn't get access to the whole network" requirement is enforced at **two independent levels**:

1. **Tunnel level**: the `AllowedIPs = 10.10.30.0/24` field on the peer restricts, at the WireGuard protocol level itself, which destinations the client can route through the tunnel — technically, the client can't even attempt to send traffic to SERVERS or LAN through this tunnel.
2. **Firewall level**: a rule on the `WIREGUARD` interface only allows `WireGuard net → IT net`, consistent with the deny-by-default principle already applied to the other interfaces in this lab.

## Validation tests

| Test (from PC-Admin-Remote, through the tunnel) | Expected result | Actual result |
|---|---|---|
| `ping 10.10.30.1` (IT) | Success | ✅ 0% loss |
| `ping 10.10.20.10` (SERVERS) | Fail | ✅ 100% loss |
| `ping 10.10.10.1` (LAN) | Fail | ✅ 100% loss |

## Incident — handshake failure, multiple intertwined causes

Setting up the VPN ran into a particularly rich incident, with several independent causes that had to be isolated one by one: WAN network topology (NAT, then Bridged, then a simulated internal network), "Block private networks"/"Block bogon networks" options silently blocking inbound traffic on WAN before custom rules were even evaluated, and a stale client configuration left over from several rounds of testing.

Full diagnostic walkthrough (tcpdump on the WAN interface, peer status checks, isolating each variable) and final resolution detailed in [troubleshooting/incident-04.md](../troubleshooting/incident-04.md).

## Security note for a real-world deployment

The "Block private networks" and "Block bogon networks" options were disabled on the WAN interface to make the VPN work in this lab (where traffic travels over private/reserved addresses internally). **On a firewall exposed to a real public Internet IP, these options must stay enabled** — disabling them is only acceptable in this isolated lab context.
