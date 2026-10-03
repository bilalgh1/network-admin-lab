# VPN d'accès distant (WireGuard)

## Scénario

Un administrateur travaille depuis chez lui et doit pouvoir accéder au réseau IT pour l'administration à distance. Un utilisateur VPN standard ne doit en revanche **pas** avoir accès à l'ensemble du réseau de l'entreprise (SERVERS, LAN restent inaccessibles).

## Choix technique

**WireGuard** a été retenu plutôt qu'OpenVPN ou IPsec : protocole moderne, natif dans OPNsense, configuration plus simple (paire de clés + quelques champs), et outil intégré ("Peer generator") qui génère automatiquement la configuration client complète.

## Architecture

| Élément | Valeur |
|---|---|
| Réseau VPN | 10.10.99.0/24 |
| Instance serveur (OPNsense) | `wg0`, Listen port UDP/51820, Tunnel Address 10.10.99.1/24 |
| Interface logique assignée | OPT3 → renommée `WIREGUARD` |
| Client testé | `PC-Admin-Remote`, VM isolée simulant un poste "à la maison" (hors de tous les réseaux internes du lab) |

## Restriction de sécurité — défense en profondeur

La contrainte "un utilisateur VPN standard n'accède pas à tout le réseau" est appliquée à **deux niveaux indépendants** :

1. **Niveau tunnel** : le champ `AllowedIPs = 10.10.30.0/24` sur le peer limite, au niveau du protocole WireGuard lui-même, les destinations que le client peut router à travers le tunnel — techniquement, le client ne peut même pas tenter d'envoyer du trafic vers SERVERS ou LAN via ce tunnel.
2. **Niveau firewall** : une règle sur l'interface `WIREGUARD` n'autorise explicitement que `WireGuard net → IT net`, conformément au principe deny by default déjà appliqué aux autres interfaces du lab.

## Tests de validation

| Test (depuis PC-Admin-Remote, via le tunnel) | Résultat attendu | Résultat obtenu |
|---|---|---|
| `ping 10.10.30.1` (IT) | Succès | ✅ 0% perte |
| `ping 10.10.20.10` (SERVERS) | Échec | ✅ 100% perte |
| `ping 10.10.10.1` (LAN) | Échec | ✅ 100% perte |

## Incident — échec de handshake, plusieurs causes imbriquées

La mise en place du VPN a rencontré un incident particulièrement riche, avec plusieurs causes indépendantes qu'il a fallu isoler une à une : topologie réseau WAN (NAT puis Bridged puis réseau simulé), options "Block private networks"/"Block bogon networks" bloquant silencieusement le trafic entrant sur WAN avant même l'évaluation des règles personnalisées, et configuration client obsolète après plusieurs itérations de test.

Démarche de diagnostic complète (tcpdump sur l'interface WAN, vérification du statut du peer, isolement de chaque variable) et résolution finale détaillées dans [troubleshooting/incident-04.md](../troubleshooting/incident-04.md).

## Note de sécurité pour un déploiement réel

Les options "Block private networks" et "Block bogon networks" ont été décochées sur l'interface WAN pour permettre le fonctionnement du VPN dans ce lab (où le trafic transite par des adresses privées/réservées en interne). **Sur un firewall exposé à une vraie adresse IP publique Internet, ces options doivent rester activées** — leur désactivation n'est acceptable que dans ce contexte de lab isolé.
