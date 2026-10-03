# Incident 10 — DNS service stopped on the domain controller

## Category
Application layer / DNS

## Scenario

Deliberately staged incident: stopped the Windows DNS service on `DC-Server1`.
```
net stop DNS
```

## Symptom

From `PC-user1`:
```
ping 10.10.20.20
```
→ 0% loss, the server stays fully reachable at the network level.

```
nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1
```
→ failure:
```
DNS request timed out.
    timeout was 2 seconds.
*** Request to OPNsense timed-out
```

## Analysis

Same logic as incident 09: the network layer (ICMP) stays healthy, only the specific application service (DNS, port 53) on the domain controller is at fault. OPNsense correctly forwards the query to the configured forwarder (`10.10.20.20`, see [services/dns.md](../services/dns.md)) but gets no reply, hence the timeout observed on the client side.

## Diagnosis (server side)

Checked the DNS service status via the `dnsmgmt.msc` console, or from the command line:
```
sc query DNS
```
→ confirms the service is stopped.

## Fix

```
net start DNS
```

## Verification

```
nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1
```
→ correct answer restored (`10.10.20.20`).

## Lesson learned

This incident illustrates the full DNS resolution chain built in this project: client → OPNsense (Dnsmasq, `company.local` forwarding) → domain controller (Windows DNS). An outage at any link in this chain breaks resolution end to end, even if every earlier link stays fully functional. Hence the value of testing each step separately (direct resolution on the DC vs. via OPNsense, as done in incident 03) to pinpoint exactly which link is at fault, rather than assuming a global cause.
