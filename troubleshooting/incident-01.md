# Incident 01 — WAN/LAN interfaces swapped

## Category
Network configuration / interface assignment

## Symptom

`PC-user1` (connected to the `intnet-users` network, intended to become LAN) receives an APIPA address (`169.254.x.x`) instead of a valid DHCP lease. `ipconfig /release` then `/renew` fails with:
```
An error occurred while renewing interface Ethernet : unable to contact your DHCP server.
```

## Hypotheses tested

1. **Mismatched internal network name** between the two VMs (OPNsense and PC-user1) — compared the "Name" field character by character in VirtualBox on both sides → identical names (`intnet-users`) confirmed on both sides. Hypothesis dismissed.

## Commands / checks

- `ipconfig /release` / `ipconfig /renew` on the client (Windows)
- Visual comparison of VirtualBox network settings (`Settings > Network`) on both VMs

## Analysis

On OPNsense's very first boot, before any manual configuration, the console displays `LAN (em0)` and `WAN (em1)` by default. During the "Assign interfaces" step (OPNsense console, option 1), this display convention was followed without verification — **but the actual card detection order didn't match this default display**. Specifically, the card physically wired to `intnet-users` (meant to become LAN) was declared WAN, and the Bridged card (meant to be WAN) was declared LAN.

## Root cause

WAN and LAN interfaces swapped during initial assignment: the interface named "LAN" by OPNsense was actually the Bridged card (connected to the Internet, not the client network), and "WAN" was actually wired to `intnet-users`. The DHCP service (expected on LAN) was therefore running on the interface physically connected to the Internet — unreachable from `PC-user1`.

## Fix

Reassigned via the OPNsense console menu (option 1, "Assign interfaces"):
- `em0` → WAN (instead of LAN)
- `em1` → LAN (instead of WAN)

Then reconfigured the IP address on LAN (option 2): `10.10.10.1/24`, DHCP restarted with the `10.10.10.100`-`10.10.10.200` range.

## Verification

```
ipconfig
```
on `PC-user1` → IP `10.10.10.101/24`, gateway `10.10.10.1` correctly obtained.

## Lesson learned

Don't trust OPNsense's default display on first boot to determine which physical card maps to which logical interface. When in doubt, check with `ifconfig` in shell (Diagnostics > Shell, or console menu option 8) to confirm the actual state (IP, link status) of each interface before confirming an assignment.
