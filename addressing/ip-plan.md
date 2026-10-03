# Plan d'adressage IP

## Contexte et contrainte de conception

Le brief initial prévoyait 5 réseaux (Users, Servers, IT, Guests, Management/99). L'environnement de virtualisation retenu (VirtualBox) limite l'interface graphique à **4 cartes réseau par VM**. Plutôt que de contourner cette limite dès le départ (via `VBoxManage` en ligne de commande, ou un changement d'hyperviseur), le choix a été fait de construire le cœur de l'infrastructure sur **4 réseaux** (WAN + 3 réseaux internes), et de documenter l'extension au VLAN Guest comme piste d'amélioration plutôt que comme contrainte bloquante dès le départ.

Un 5e réseau (WireGuard VPN, `10.10.99.0/24`) a été ajouté en Phase 8, en assignant l'interface WireGuard comme une interface logique supplémentaire (OPT3) sur OPNsense — cette méthode ne consomme pas de carte réseau VirtualBox physique, donc pas de nouvelle limite rencontrée à ce stade.

## Table d'adressage

| Réseau | Fonction | Plage | Masque | Gateway | Plage DHCP |
|---|---|---|---|---|---|
| LAN | Users (postes employés) | 10.10.10.0/24 | /24 (255.255.255.0) | 10.10.10.1 | 10.10.10.100 → 10.10.10.200 |
| SERVERS | Serveurs applicatifs | 10.10.20.0/24 | /24 | 10.10.20.1 | *(aucune — IP fixes)* |
| IT | Postes et accès administrateurs | 10.10.30.0/24 | /24 | 10.10.30.1 | 10.10.30.50 → 10.10.30.100 |
| WireGuard (VPN) | Accès distant admin | 10.10.99.0/24 | /24 | 10.10.99.1 | *(peers configurés individuellement)* |

## Justification du découpage

- **Un `/24` par réseau** (254 adresses utilisables) est largement suffisant pour une PME de ~50 employés répartis sur plusieurs segments, tout en gardant les calculs de sous-réseau simples — pas de sur-ingénierie pour ce contexte.
- **SERVERS n'a volontairement pas de plage DHCP** : les serveurs doivent conserver une adresse stable et prévisible dans le temps, car des services dépendent directement de cette adresse (résolution DNS, règles firewall pointant vers une IP précise, GPO potentielles). Une IP qui changerait à chaque redémarrage casserait silencieusement ces dépendances.
- **IT dispose d'une plage DHCP réduite** (50 adresses, 50-100) : réseau destiné à un nombre restreint de postes et équipements d'administration, pas besoin d'une plage large comme sur Users.
- **Le réseau WireGuard (10.10.99.0/24)** est isolé des autres plages internes pour que la restriction d'accès VPN (routage limité au réseau IT uniquement via le champ `AllowedIPs`) soit sans ambiguïté — le préfixe `.99` est une convention courante pour un réseau de management/VPN, cohérente avec l'exemple donné dans le brief initial.

## Adresses fixes attribuées

| Machine | Rôle | Réseau | IP fixe |
|---|---|---|---|
| OPNsense-FW | Firewall / routeur | LAN / SERVERS / IT (gateway de chacun) | .1 sur chaque réseau |
| SRV-Ubuntu1 | Serveur web (Nginx) + SSH | SERVERS | 10.10.20.10 |
| DC-Server1 | Contrôleur de domaine (AD + DNS) | SERVERS | 10.10.20.20 |

Les postes clients (`PC-user1`) et les peers VPN obtiennent leur adresse dynamiquement (DHCP pour LAN/IT, configuration de tunnel pour WireGuard).
