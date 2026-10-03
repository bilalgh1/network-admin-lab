# Incident 04 — VPN WireGuard, échec de handshake (causes multiples)

## Catégorie
Réseau / NAT / firewall — incident à causes imbriquées

## Contexte

Mise en place du VPN WireGuard pour l'accès admin distant (voir [services/vpn.md](../services/vpn.md)). Instance serveur, peer, interface assignée et règles firewall configurées conformément au plan.

## Symptôme initial

Le tunnel apparaît "Active" côté client Windows, mais le peer reste en statut **rouge** côté OPNsense (`VPN > WireGuard > Status`), avec "Handshake Age" vide et 0 octet envoyé/reçu des deux côtés. `ping 10.10.30.1` depuis le client VPN échoue systématiquement (100% perte).

## Démarche de diagnostic (plusieurs causes identifiées et corrigées successivement)

### 1. Topologie WAN incorrecte
Découverte que la carte WAN d'OPNsense était en mode **NAT** (`10.0.2.15`, adresse typique VirtualBox) plutôt qu'en Bridged comme initialement prévu. Un client VPN lui-même en NAT (simulant un poste "à la maison") ne pouvait pas joindre une IP interne au NAT d'une autre VM — les deux VMs étaient chacune dans leur propre "bulle" NAT isolée, sans chemin direct entre elles.

**Action** : passage de WAN en Bridged → obtention d'une IP réelle du réseau domestique (`192.168.1.20`). Amélioration partielle (Endpoint théoriquement joignable), mais handshake toujours en échec.

### 2. Règle firewall WAN manquante
Le port WireGuard (UDP/51820) n'était pas explicitement autorisé en entrée sur l'interface WAN — deny by default s'applique aussi sur WAN. Règle `Pass UDP 51820 → WAN address` créée.

**Résultat** : aucun changement observé à ce stade.

### 3. Promiscuous Mode de la carte Bridged
Piste explorée : en mode Bridged, VirtualBox peut par défaut restreindre le trafic inter-VM passant par le pont réseau de l'hôte. Passage de "Promiscuous Mode" sur "Allow All".

**Résultat** : aucun changement observé.

### 4. Cause principale — options "Block private/bogon networks"
OPNsense active par défaut, sur l'interface WAN, deux protections : **"Block private networks"** et **"Block bogon networks"**. Elles sont évaluées **avant** toute règle personnalisée. Tout le trafic testé jusqu'ici provenait d'adresses privées ou réservées (`192.168.x.x`, puis `203.0.113.0/24` lors d'un test en réseau interne simulé, puis `10.0.2.x` en NAT) — systématiquement bloqué par ces deux options, rendant inutile la règle créée à l'étape 2.

**Action** : décochées sur `Interfaces > [WAN] > Generic configuration`.

### 5. Peer client obsolète
Après plusieurs changements de topologie réseau en cours de diagnostic (Bridged, réseau interne simulé, retour au NAT d'origine), un **peer obsolète** restait configuré côté client avec un ancien Endpoint ne correspondant plus à la topologie finale — cause d'échecs redondants malgré des corrections par ailleurs correctes à ce stade.

**Action** : suppression de tous les anciens peers, création d'une configuration unique et cohérente (`Endpoint = 10.0.2.2:51820`, correspondant au NAT restauré, avec redirection de port UDP/51820 → `10.0.2.15` configurée sur la carte NAT d'OPNsense).

## Vérification finale

`VPN > WireGuard > Status` → peer au vert, Handshake Age récent.

Depuis le poste client VPN :
```
ping 10.10.30.1   → 0% perte (IT, autorisé)
ping 10.10.20.10  → 100% perte (SERVERS, non autorisé)
ping 10.10.10.1   → 100% perte (LAN, non autorisé)
```

## Leçon retenue

Cet incident a cumulé **plusieurs causes indépendantes** qu'il a fallu isoler une à une, en changeant une variable à la fois et en revérifiant après chaque correction : topologie réseau (NAT/Bridged), options de sécurité masquant silencieusement les règles personnalisées (private/bogon), et état de configuration client devenu incohérent après plusieurs itérations de test. Bon exemple pour illustrer qu'un incident réseau complexe n'a pas toujours une cause unique — la persévérance méthodique, en gardant une trace de chaque hypothèse testée et son résultat, est ce qui permet in fine d'isoler chaque facteur.
