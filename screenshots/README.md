# Screenshots

This folder gathers screenshots taken throughout the project, as a complement to the `.md` files under `troubleshooting/` and `services/`.

## Suggested organization

```
screenshots/
├── opnsense-dashboard.png
├── interfaces-overview.png
├── firewall-rules-lan.png
├── firewall-rules-wireguard.png
├── dnsmasq-dhcp-ranges.png
├── ad-server-manager.png
├── incident-02-tcpdump-dhcp.png
├── incident-03-tcpdump-dns.png
├── incident-04-wireguard-status.png
├── vpn-peer-generator.png
└── nginx-welcome-page.png
```

## Priority screenshots to include (already captured during the project)

- OPNsense Dashboard (proof of a successful install)
- Interface list (`Interfaces > Assignments`) showing LAN/SERVERS/IT/WIREGUARD
- Firewall rules table (`Firewall > Rules`) for LAN and WIREGUARD
- `tcpdump` output from incident 02 (DHCP requests visible, no reply)
- Final WireGuard status (`VPN > WireGuard > Status`) showing a successful handshake
- "Welcome to nginx!" page shown in PC-user1's browser
- Successful login to the `COMPANY\Administrator` domain account on DC-Server1

## Naming convention

`<area>-<description>.png`, lowercase with hyphens — e.g. `firewall-block-users-it.png`, `dns-nslookup-success.png`. Makes it easy to link from the `.md` files in this repo (e.g. `![Screenshot](../screenshots/firewall-block-users-it.png)`).
