# Incident 01 — Interfaces WAN/LAN inversées

## Catégorie
Configuration réseau / assignation d'interfaces

## Symptôme

`PC-user1` (branché sur le réseau `intnet-users`, destiné à devenir LAN) reçoit une adresse APIPA (`169.254.x.x`) au lieu d'une IP DHCP valide. `ipconfig /release` puis `/renew` échoue avec :
```
An error occurred while renewing interface Ethernet : unable to contact your DHCP server.
```

## Hypothèses testées

1. **Mismatch de nom de réseau interne** entre les deux VMs (OPNsense et PC-user1) — comparaison caractère par caractère des champs "Name" dans VirtualBox des deux côtés → noms identiques (`intnet-users`) confirmés des deux côtés. Hypothèse écartée.

## Commandes / vérifications

- `ipconfig /release` / `ipconfig /renew` côté client (Windows)
- Comparaison visuelle des réglages réseau VirtualBox (`Configuration > Réseau`) des deux VMs

## Analyse

Au tout premier démarrage d'OPNsense, avant toute configuration manuelle, la console affiche par défaut `LAN (em0)` et `WAN (em1)`. Lors de l'étape "Assign interfaces" (console OPNsense, option 1), cette convention d'affichage a été suivie sans vérification — **mais l'ordre réel de détection des cartes par le système ne correspondait pas à cet affichage par défaut**. Concrètement, la carte physiquement branchée sur `intnet-users` (censée devenir LAN) a été déclarée WAN, et la carte en mode Bridged (censée être WAN) a été déclarée LAN.

## Cause

Interfaces WAN et LAN inversées lors de l'assignation initiale : l'interface nommée "LAN" par OPNsense était en réalité la carte Bridged (connectée à Internet, pas au réseau client), et "WAN" était en réalité branchée sur `intnet-users`. Le service DHCP (attendu sur LAN) tournait donc sur l'interface physiquement connectée à Internet — inaccessible pour `PC-user1`.

## Correction

Réassignation via le menu console OPNsense (option 1, "Assign interfaces") :
- `em0` → WAN (au lieu de LAN)
- `em1` → LAN (au lieu de WAN)

Puis reconfiguration de l'adresse IP sur LAN (option 2) : `10.10.10.1/24`, DHCP relancé avec la plage `10.10.10.100`-`10.10.10.200`.

## Vérification

```
ipconfig
```
sur `PC-user1` → IP `10.10.10.101/24`, gateway `10.10.10.1` obtenue correctement.

## Leçon retenue

Ne pas se fier à l'affichage par défaut d'OPNsense au tout premier boot pour déterminer quelle carte physique correspond à quelle interface logique. En cas de doute, vérifier avec `ifconfig` en shell (Diagnostics > Shell, ou option 8 du menu console) pour confirmer l'état réel (IP, lien actif) de chaque interface avant de valider une assignation.
