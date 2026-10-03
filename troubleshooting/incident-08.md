# Incident 08 — Wrong gateway

## Category
Routing

## Context

Deliberately staged incident to illustrate a classic network troubleshooting case (example cited in the project's original brief).

## Scenario

On `PC-user1`, switched the network adapter to a static IP with a deliberately wrong gateway:
- IP: `10.10.10.150`
- Mask: `255.255.255.0`
- Gateway: `10.10.10.99` (instead of the real one, `10.10.10.1`)

## Symptom

```
ping 10.10.10.1   → ✅ success (local network)
ping 8.8.8.8      → ❌ expected failure (Internet)
```

## First misleading test

On the first attempt, `ping 8.8.8.8` still succeeded, contrary to what the wrong gateway configuration implied.

### Diagnosis
```
ipconfig
```
revealed the cause: a **second network adapter** ("Ethernet 2"), leftover from an earlier SSH test (connected to `intnet-it`), was still active with a valid IP and gateway on `10.10.30.0/24`. Windows used this alternate route to reach the Internet, completely masking the effect of the broken gateway on the first adapter.

### Takeaway
On a multi-adapter machine, a gateway problem on one interface can be invisible if another interface offers a valid fallback route. Always check `ipconfig` in full (not just the adapter being modified) before concluding a failure test didn't produce the expected effect.

## Clean test (after disabling "Ethernet 2")

```
ping 10.10.10.1
```
→ 0% loss, normal success (same local subnet, direct ARP resolution, the gateway isn't involved).

```
ping 8.8.8.8
```
→ 75% loss, `Destination host unreachable` returned by the machine itself (not a regular network timeout) — the machine immediately knows it can't route outbound, lacking a valid gateway in its routing table.

## Complementary diagnosis

```
ping 10.10.10.99
```
(the configured gateway) → timeout, gateway doesn't exist on the network — confirms the gateway itself is unreachable, the root cause of the problem.

## Fix

Restored the correct gateway (`10.10.10.1`), or switched back to automatic DHCP.

## Verification

```
ping 8.8.8.8
```
→ succeeds again after the fix.

## Lesson learned

A successful ping to the local network doesn't imply working routing to the outside world — the gateway only comes into play for traffic headed to a remote network. When troubleshooting an Internet outage with a working local network, systematically test the gateway itself (`ping <gateway IP>`) to quickly confirm or rule out this path, and check every active interface on the machine under test.
