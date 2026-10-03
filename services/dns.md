# DNS Service

## Resolution architecture

```
Client (LAN/SERVERS/IT)
      │  DNS = 10.10.10.1 (OPNsense)
      ▼
OPNsense — Dnsmasq DNS & DHCP (port 53)
      │  *.company.local queries → forward
      ▼
DC-Server1 — Windows DNS (integrated with Active Directory)
      10.10.20.20
```

Clients on the network use OPNsense as their single DNS server. OPNsense resolves generic queries itself (toward the Internet) and **conditionally forwards** queries for the `company.local` domain to the Active Directory domain controller, which is authoritative for that domain.

## Configuration

- **OPNsense**: `Services > Dnsmasq DNS & DHCP > Domains` — entry `company.local → 10.10.20.20` (conditional forwarding)
- **DC-Server1**: DNS Server role installed automatically alongside Active Directory Domain Services when promoted to domain controller (root domain `company.local`, NetBIOS `COMPANY`)

## Major incident — conflict between two active DNS services

The forwarding rule, although configured correctly from the start, simply didn't work. The actual cause: **two DNS services active simultaneously on OPNsense** — Dnsmasq (configured on a non-standard port, `53053`, visible in `General > Listen port`) and **Unbound DNS**, enabled in parallel and actually answering on the standard port 53. Clients querying the standard port 53 were therefore getting answers from Unbound, which completely ignored the forwarding rule configured in Dnsmasq.

Full diagnosis, backed by a tcpdump capture, in [troubleshooting/incident-03.md](../troubleshooting/incident-03.md).

**Fix**: disabled Unbound DNS, moved Dnsmasq back to the standard port 53.

## Validation tests

| Test | Result |
|---|---|
| `nslookup WIN-A5T5IRJE99D.company.local 10.10.20.20` (direct to the DC) | ✅ Resolves correctly |
| `nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1` (via OPNsense, before fix) | ❌ "Non-existent domain" |
| `nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1` (via OPNsense, after fix) | ✅ Resolves correctly (`10.10.20.20`) |
| `ping google.com` from PC-user1 and SRV-Ubuntu1 (external resolution) | ✅ Fully functional |

## Resilience tested

The DNS service on `DC-Server1` was deliberately stopped (`net stop DNS`) to observe failure behavior — see [troubleshooting/incident-10.md](../troubleshooting/incident-10.md). Confirms the whole internal resolution chain's dependency on this service: an outage breaks `company.local` resolution end to end, even though OPNsense and the rest of the network stay fully functional.
