# DHCP Service

## Technology

OPNsense 26.x replaces the older DHCPv4 menu (configured separately per interface) with a unified service: **Dnsmasq DNS & DHCP** (`Services > Dnsmasq DNS & DHCP`), which handles DHCP and DNS together. This is the service used in this lab, rather than the alternative Kea DHCP also available.

## Configuration per network

| Network | DHCP enabled | Range | Rationale |
|---|---|---|---|
| LAN (Users) | ✅ Yes | 10.10.10.100 → 10.10.10.200 | User workstations, standard dynamic assignment |
| SERVERS | ❌ No | — | Static IPs assigned manually (see [addressing/ip-plan.md](../addressing/ip-plan.md)) — a server needs to keep a stable, predictable address |
| IT | ✅ Yes | 10.10.30.50 → 10.10.30.100 | Admin workstations, smaller range (few devices expected) |

## Expected behavior (DORA sequence)

```
DHCP Discover (client)
       ↓
DHCP Offer (Dnsmasq)
       ↓
DHCP Request (client)
       ↓
DHCP ACK (Dnsmasq)
```

The client obtains: IP address, subnet mask, gateway, DNS server(s).

## Two-level configuration — issue encountered

On OPNsense with Dnsmasq, there are **two distinct settings that are easy to confuse**:
1. `Services > Dnsmasq DNS & DHCP > General > Interface` — the global list of interfaces the service actually **listens** on
2. `Services > Dnsmasq DNS & DHCP > DHCP ranges` — the DHCP ranges configured per interface

A range can be perfectly configured in (2) and still be completely non-functional if the matching interface isn't checked in (1). See the full breakdown of this incident, diagnosed via tcpdump capture, in [troubleshooting/incident-02.md](../troubleshooting/incident-02.md).

## Validation tests

| Test | Result |
|---|---|
| `PC-user1` on LAN → `ipconfig` | IP `10.10.10.101/24`, gateway `10.10.10.1` correctly obtained |
| Secondary network adapter on IT → `ipconfig /renew` | IP `10.10.30.89/24`, gateway `10.10.30.1` correctly obtained (after fixing incident 02) |
| SERVERS (Ubuntu, Windows Server) | No DHCP — static IPs configured manually (Netplan on Ubuntu, Network Control Panel on Windows) |
