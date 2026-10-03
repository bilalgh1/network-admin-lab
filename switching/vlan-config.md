# Segmentation réseau (VLANs équivalents)

## Approche retenue

Ce lab étant construit en virtualisation pure (VirtualBox), la segmentation n'est pas réalisée via un switch physique/virtuel avec trunking 802.1Q, mais via des **réseaux internes VirtualBox isolés** ("Internal Network"), chacun branché sur sa propre interface réseau d'OPNsense. Chaque réseau interne joue le rôle d'un VLAN : les machines qui y sont connectées ne peuvent communiquer qu'entre elles et avec OPNsense, jamais directement avec un autre segment — tout trafic inter-segment passe obligatoirement par le firewall.

| Réseau interne VirtualBox | Rôle (VLAN équivalent) | Interface OPNsense | Device |
|---|---|---|---|
| `intnet-users` | Users (VLAN 10 équivalent) | LAN | em1 |
| `intnet-servers` | Servers (VLAN 20 équivalent) | SERVERS | em2 |
| `intnet-it` | IT / Admin (VLAN 30 équivalent) | IT | em3 |
| *(carte Bridged/NAT)* | WAN — sortie Internet | WAN | em0 |

*Limitation assumée et documentée dans [addressing/ip-plan.md](../addressing/ip-plan.md) : 4 réseaux au lieu de 5 prévus initialement, en raison de la limite de 4 cartes réseau par VM dans l'interface graphique VirtualBox.*

## Assignation des interfaces sur OPNsense

L'assignation des interfaces (`em0`-`em3` vers WAN/LAN/SERVERS/IT) se fait via le menu console d'OPNsense (option 1, "Assign interfaces"), puis l'attribution d'adresse IP (option 2, "Set interface IP address").

Chaque interface a ensuite été activée et nommée via l'interface web (`Interfaces > [nom]`) :
- **Enable Interface** coché
- **Description** renommée (LAN, SERVERS, IT) pour plus de lisibilité dans tout le reste de l'interface OPNsense (menus Firewall, Services, etc.)
- **Static IPv4**, adresse correspondant au plan d'adressage

## Point d'attention — inversion WAN/LAN au premier démarrage

Voir [troubleshooting/incident-01.md](../troubleshooting/incident-01.md) : lors du tout premier assign, les interfaces WAN et LAN ont été inversées par erreur (la carte physiquement reliée à `intnet-users` a été déclarée WAN, et inversement). OPNsense affiche par défaut `LAN (em0)` au tout premier boot avant toute configuration, ce qui peut induire en erreur sur l'ordre réel des cartes — toujours vérifier avec `ifconfig` en shell (Diagnostics > Shell) en cas de doute, plutôt que de se fier à l'ordre d'affichage initial.

## Interface VPN (WireGuard) comme segment logique supplémentaire

En Phase 8, l'instance WireGuard (`wg0`) a été assignée comme une interface logique à part entière (`Interfaces > Assignments`), apparaissant comme `OPT3` puis renommée `WIREGUARD`. Cette interface se comporte exactement comme les interfaces physiques (SERVERS, IT) du point de vue du firewall : deny by default tant qu'aucune règle n'est créée. Voir [services/vpn.md](../services/vpn.md).

## Pistes d'amélioration

Pour pratiquer le vrai trunking 802.1Q, le VLAN tagging, le Spanning Tree Protocol (STP) et l'EtherChannel/LACP comme prévu dans le brief original, la suite logique serait d'introduire un switch Cisco IOS virtualisé (GNS3 ou EVE-NG) entre OPNsense et les VMs, avec un port trunk vers le firewall et des ports access par VLAN vers chaque machine — non réalisé dans cette itération par contrainte de temps.
