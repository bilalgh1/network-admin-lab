# Incident 03 — DNS conflict between Dnsmasq and Unbound (company.local forwarding failure)

## Category
Service / port conflict

## Initial goal

Allow every host on the network (using OPNsense as the single DNS server) to resolve internal names from the `company.local` Active Directory domain, by forwarding those specific queries to the domain controller (`10.10.20.20`), without reconfiguring each client individually.

## Configuration put in place (seemingly correct)

`Services > Dnsmasq DNS & DHCP > Domains`: entry `company.local → 10.10.20.20` (conditional forwarding).

## Symptom

- `nslookup WIN-A5T5IRJE99D.company.local 10.10.20.20` (querying the domain controller directly) → **success**
- `nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1` (via OPNsense) → **failure**:
```
*** OPNsense.internal can't find WIN-A5T5IRJE99D.company.local: Non-existent domain
```

## Diagnostic process

1. Checked the Dnsmasq configuration (Domain / IP or Host fields) → syntax correct, nothing to fix.
2. Forced "Apply" and manually restarted the Dnsmasq service (service control button) → no change observed.
3. **tcpdump capture on OPNsense**, on the SERVERS interface, during a fresh test:
```
tcpdump -i em2 port 53 -n
```
Result: **complete silence** — no DNS packet going out toward `10.10.20.20` during the resolution attempt. Unlike incident 02 (where the packet arrived but wasn't processed), here the request wasn't even **sent** by OPNsense to the controller — a sign that the client request never reached the right internal process on OPNsense in the first place.
4. Checked `Services > Dnsmasq DNS & DHCP > General > Listen port`: set to **`53053`**, not the standard DNS port (`53`). A client queries port 53 by default; Dnsmasq, listening on 53053, never received the initial client request.
5. Checked `Services > Unbound DNS`: service **enabled** in parallel with Dnsmasq.

## Root cause

**Two DNS servers active simultaneously on OPNsense** — Dnsmasq (custom port 53053) and Unbound DNS (standard port 53), with no coordination between them. Clients, querying the standard port 53 by default, were actually getting answers from **Unbound** — which completely ignores the forwarding rule configured in Dnsmasq, hence the "Non-existent domain". The forwarding configuration, syntactically correct, was applied to the wrong service — the one never actually queried by client traffic.

## Fix

1. Disabled Unbound DNS (`Services > Unbound DNS > General > Enable` unchecked, Save)
2. Moved Dnsmasq back to the standard port: `Services > Dnsmasq DNS & DHCP > General > Listen port` = `53`, Save
3. Forced a restart of the Dnsmasq service

## Verification

```
nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1
```
→ correct answer (`10.10.20.20`).

## Lesson learned

On OPNsense, several DNS services (Dnsmasq, Unbound, Kea DHCP) can coexist and be enabled independently, with no conflict warning. You need to make sure only one DNS service is actually "authoritative" on the standard port 53 — otherwise a correct configuration on one service has zero effect on traffic actually handled by the other. This is a silent trap that only shows up by cross-referencing the configuration (Listen port) with a packet capture showing a total absence of outbound requests.
