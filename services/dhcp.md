# Service DHCP

## Technologie

OPNsense 26.x remplace l'ancien menu DHCPv4 (configuré séparément par interface) par un service unifié : **Dnsmasq DNS & DHCP** (`Services > Dnsmasq DNS & DHCP`), qui gère DHCP et DNS ensemble. C'est le service retenu pour ce lab plutôt que l'alternative Kea DHCP également disponible.

## Configuration par réseau

| Réseau | DHCP activé | Plage | Justification |
|---|---|---|---|
| LAN (Users) | ✅ Oui | 10.10.10.100 → 10.10.10.200 | Postes utilisateurs, attribution dynamique standard |
| SERVERS | ❌ Non | — | IP fixes attribuées manuellement (voir [addressing/ip-plan.md](../addressing/ip-plan.md)) — un serveur doit conserver une adresse stable et prévisible |
| IT | ✅ Oui | 10.10.30.50 → 10.10.30.100 | Postes d'administration, plage réduite (peu d'équipements attendus) |

## Fonctionnement attendu (séquence DORA)

```
DHCP Discover (client)
       ↓
DHCP Offer (Dnsmasq)
       ↓
DHCP Request (client)
       ↓
DHCP ACK (Dnsmasq)
```

Le client obtient : adresse IP, masque, passerelle, serveur(s) DNS.

## Point de configuration à deux niveaux — piège rencontré

Sur OPNsense avec Dnsmasq, il existe **deux réglages distincts et faciles à confondre** :
1. `Services > Dnsmasq DNS & DHCP > General > Interface` — la liste globale des interfaces sur lesquelles le service **écoute** réellement
2. `Services > Dnsmasq DNS & DHCP > DHCP ranges` — les plages DHCP configurées par interface

Une plage peut être parfaitement configurée dans (2) et pourtant rester totalement inopérante si l'interface correspondante n'est pas cochée dans (1). Voir le détail complet de cet incident, diagnostiqué par capture tcpdump, dans [troubleshooting/incident-02.md](../troubleshooting/incident-02.md).

## Tests de validation

| Test | Résultat |
|---|---|
| `PC-user1` sur LAN → `ipconfig` | IP `10.10.10.101/24`, gateway `10.10.10.1` obtenue correctement |
| Carte réseau secondaire sur IT → `ipconfig /renew` | IP `10.10.30.89/24`, gateway `10.10.30.1` obtenue correctement (après correction de l'incident 02) |
| SERVERS (Ubuntu, Windows Server) | Pas de DHCP — IP fixes configurées manuellement (Netplan sur Ubuntu, Panneau de configuration réseau sur Windows) |
