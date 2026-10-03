# Incident 02 — DHCP n'écoute pas sur l'interface IT

## Catégorie
Service / configuration à deux niveaux

## Contexte

Ajout d'une 2e carte réseau sur `PC-user1`, branchée sur `intnet-it`, pour simuler un poste admin et tester l'accès SSH depuis le réseau IT.

## Symptôme

La nouvelle carte ("Ethernet 2") reste en IP APIPA (`169.254.x.x`). `ipconfig /renew "Ethernet 2"` échoue :
```
unable to contact your DHCP server. Request has timed out.
```

## Hypothèses testées (dans l'ordre)

1. **Carte réseau VirtualBox désactivée** côté OPNsense → vérification de `Configuration > Réseau > Carte 4` d'`OPNsense-FW` : "Activer l'interface réseau" était effectivement décoché → corrigé, mais le problème persistait après correction et redémarrage. Hypothèse partiellement confirmée mais insuffisante.
2. **Service Dnsmasq arrêté** → vérifié via `service dnsmasq status` en shell OPNsense (Diagnostics > Shell) : service actif (`running as pid ...`). Hypothèse écartée.
3. **Trafic bloqué en chemin** (firewall, routage) → testé par capture réseau (voir ci-dessous).

## Commandes / Wireshark-tcpdump

Capture directement sur l'interface IT d'OPNsense :
```
tcpdump -i em3 port 67 or port 68 -n
```
Résultat : les requêtes DHCP Discover/Request du client **arrivent bien** jusqu'à l'interface (paquets `BOOTP/DHCP, Request from ...` visibles), mais **aucune réponse (Offer/ACK) n'est émise** par OPNsense.

## Analyse

Le paquet arrive à destination mais n'est pas traité par le service DHCP — signe d'un problème de configuration du service lui-même, pas d'un problème réseau/câblage.

Vérification de `Services > Dnsmasq DNS & DHCP > General > Interface` : ce champ (liste à sélection multiple déterminant sur quelles interfaces le service **écoute** réellement) ne contenait que **LAN**. L'interface IT n'y avait jamais été ajoutée — alors que la plage DHCP pour IT (`Services > Dnsmasq DNS & DHCP > DHCP ranges`) était, elle, parfaitement configurée et visible dans la liste des plages.

## Cause

Deux niveaux de configuration distincts sur Dnsmasq, faciles à confondre :
- (a) la liste globale des interfaces écoutées (`General > Interface`)
- (b) les plages DHCP par interface (`DHCP ranges`)

Une plage peut être correctement configurée dans (b) et pourtant rester totalement inopérante si l'interface correspondante n'est pas cochée dans (a). Le service tournait, recevait les paquets, mais les ignorait silencieusement car IT n'était pas dans sa liste d'écoute.

## Correction

Ajout de `IT` dans le champ `Interface` (General), en conservant `LAN` déjà présent. Save, puis redémarrage forcé du service Dnsmasq.

## Vérification

```
ipconfig /renew "Ethernet 2"
```
→ IP `10.10.30.89/24`, gateway `10.10.30.1` obtenue correctement.

## Leçon retenue

Sur OPNsense avec le service unifié Dnsmasq DNS & DHCP, une configuration apparemment correcte (plage DHCP bien définie) peut ne produire aucun effet si l'interface n'est pas explicitement ajoutée à la liste d'écoute globale du service. C'est un piège silencieux, sans message d'erreur — seule une capture réseau (tcpdump) permet de distinguer avec certitude "le paquet n'arrive pas" de "le paquet arrive mais n'est pas traité", et donc d'orienter le diagnostic vers la bonne cause.
