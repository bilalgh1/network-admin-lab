# Screenshots

Ce dossier rassemble les captures d'écran prises pendant le projet, utiles en complément des fichiers `.md` du dossier `troubleshooting/` et `services/`.

## Organisation suggérée

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

## Captures prioritaires à inclure (déjà prises pendant le projet)

- Dashboard OPNsense (preuve de l'installation réussie)
- Liste des interfaces (`Interfaces > Assignments`) montrant LAN/SERVERS/IT/WIREGUARD
- Table des règles firewall (`Firewall > Rules`) pour LAN et WIREGUARD
- Résultat `tcpdump` de l'incident 02 (requêtes DHCP visibles, sans réponse)
- Statut WireGuard final (`VPN > WireGuard > Status`) montrant le handshake réussi
- Page "Welcome to nginx!" affichée depuis le navigateur de PC-user1
- Connexion réussie au domaine `COMPANY\Administrator` sur DC-Server1

## Convention de nommage

`<domaine>-<description>.png`, en minuscules avec tirets — ex. `firewall-block-users-it.png`, `dns-nslookup-success.png`. Facilite les liens relatifs depuis les fichiers `.md` du repo (ex. `![Capture](../screenshots/firewall-block-users-it.png)`).
