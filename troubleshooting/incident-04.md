# Incident 04 — WireGuard VPN handshake failure (multiple causes)

## Category
Network / NAT / firewall — multi-cause incident

## Context

Setting up the WireGuard VPN for remote admin access (see [services/vpn.md](../services/vpn.md)). Server instance, peer, assigned interface, and firewall rules all configured per plan.

## Initial symptom

The tunnel shows "Active" on the Windows client side, but the peer stays **red** on the OPNsense side (`VPN > WireGuard > Status`), with an empty "Handshake Age" and 0 bytes sent/received on both ends. `ping 10.10.30.1` from the VPN client fails consistently (100% loss).

## Diagnostic process (several causes identified and fixed one after another)

### 1. Incorrect WAN topology
Discovered that OPNsense's WAN card was in **NAT** mode (`10.0.2.15`, a typical VirtualBox address) rather than Bridged as originally planned. A VPN client itself behind NAT (simulating a "home" workstation) couldn't reach an address internal to another VM's NAT — the two VMs were each stuck in their own isolated NAT "bubble," with no direct path between them.

**Action**: switched WAN to Bridged → obtained a real address on the home network (`192.168.1.20`). Partial improvement (Endpoint theoretically reachable), but handshake still failing.

### 2. Missing WAN firewall rule
The WireGuard port (UDP/51820) wasn't explicitly allowed inbound on the WAN interface — deny by default also applies on WAN. Created a `Pass UDP 51820 → WAN address` rule.

**Result**: no change observed at this stage.

### 3. Promiscuous Mode on the Bridged adapter
Explored possibility: in Bridged mode, VirtualBox can by default restrict inter-VM traffic passing through the host's network bridge. Switched "Promiscuous Mode" to "Allow All."

**Result**: no change observed.

### 4. Main root cause — "Block private/bogon networks" options
OPNsense enables, by default on the WAN interface, two protections: **"Block private networks"** and **"Block bogon networks"**. These are evaluated **before** any custom rule. All traffic tested so far came from private or reserved addresses (`192.168.x.x`, then `203.0.113.0/24` during a test on a simulated internal network, then `10.0.2.x` under NAT) — consistently blocked by these two options, making the rule created in step 2 useless.

**Action**: unchecked them under `Interfaces > [WAN] > Generic configuration`.

### 5. Stale client peer
After several network topology changes made during diagnosis (Bridged, simulated internal network, back to the original NAT), a **stale peer** was still configured on the client side with an old Endpoint that no longer matched the final topology — causing repeated failures despite otherwise correct fixes at this point.

**Action**: deleted all old peers, created a single, consistent configuration (`Endpoint = 10.0.2.2:51820`, matching the restored NAT setup, with UDP/51820 port forwarding → `10.0.2.15` configured on OPNsense's NAT card).

## Final verification

`VPN > WireGuard > Status` → peer green, recent Handshake Age.

From the VPN client:
```
ping 10.10.30.1   → 0% loss (IT, allowed)
ping 10.10.20.10  → 100% loss (SERVERS, not allowed)
ping 10.10.10.1   → 100% loss (LAN, not allowed)
```

## Lesson learned

This incident stacked up **several independent causes** that had to be isolated one by one, changing one variable at a time and re-verifying after each fix: network topology (NAT/Bridged), security options silently masking custom rules (private/bogon), and client configuration state that became stale after several rounds of testing. A good example that a complex network incident doesn't always have a single root cause — methodical persistence, keeping track of each hypothesis tested and its outcome, is ultimately what isolates every factor.
