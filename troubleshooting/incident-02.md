# Incident 02 — DHCP not listening on the IT interface

## Category
Service / two-level configuration

## Context

Added a 2nd network adapter on `PC-user1`, connected to `intnet-it`, to simulate an admin workstation and test SSH access from the IT network.

## Symptom

The new adapter ("Ethernet 2") stays on an APIPA address (`169.254.x.x`). `ipconfig /renew "Ethernet 2"` fails:
```
unable to contact your DHCP server. Request has timed out.
```

## Hypotheses tested (in order)

1. **VirtualBox network adapter disabled** on the OPNsense side → checked `Settings > Network > Adapter 4` on `OPNsense-FW`: "Enable Network Adapter" was indeed unchecked → fixed, but the problem persisted after the fix and a reboot. Hypothesis partially confirmed but not sufficient.
2. **Dnsmasq service stopped** → checked via `service dnsmasq status` in the OPNsense shell (Diagnostics > Shell): service active (`running as pid ...`). Hypothesis dismissed.
3. **Traffic blocked somewhere in the path** (firewall, routing) → tested via packet capture (see below).

## Commands / Wireshark-tcpdump

Captured directly on OPNsense's IT interface:
```
tcpdump -i em3 port 67 or port 68 -n
```
Result: the client's DHCP Discover/Request packets **do arrive** at the interface (`BOOTP/DHCP, Request from ...` packets visible), but **no reply (Offer/ACK) is ever sent** by OPNsense.

## Analysis

The packet reaches its destination but isn't processed by the DHCP service — a sign of a configuration issue with the service itself, not a network/wiring problem.

Checked `Services > Dnsmasq DNS & DHCP > General > Interface`: this field (a multi-select list determining which interfaces the service actually **listens** on) only contained **LAN**. The IT interface had never been added to it — even though the DHCP range for IT (`Services > Dnsmasq DNS & DHCP > DHCP ranges`) was perfectly configured and visible in the ranges list.

## Root cause

Two distinct, easily confused configuration levels on Dnsmasq:
- (a) the global list of listened-on interfaces (`General > Interface`)
- (b) the DHCP ranges per interface (`DHCP ranges`)

A range can be correctly configured in (b) and still be completely non-functional if the matching interface isn't checked in (a). The service was running and receiving the packets, but silently ignoring them because IT wasn't in its listen list.

## Fix

Added `IT` to the `Interface` field (General), keeping `LAN` already present. Saved, then forced a restart of the Dnsmasq service.

## Verification

```
ipconfig /renew "Ethernet 2"
```
→ IP `10.10.30.89/24`, gateway `10.10.30.1` correctly obtained.

## Lesson learned

On OPNsense with the unified Dnsmasq DNS & DHCP service, an apparently correct configuration (a well-defined DHCP range) can have zero effect if the interface isn't explicitly added to the service's global listen list. This is a silent trap with no error message — only a packet capture (tcpdump) can reliably distinguish "the packet never arrives" from "the packet arrives but isn't processed," and therefore point the diagnosis in the right direction.
