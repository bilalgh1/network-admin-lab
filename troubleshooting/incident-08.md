# Incident 08 — Mauvaise gateway

## Catégorie
Routage

## Contexte

Incident volontairement provoqué pour illustrer un cas classique de troubleshooting réseau (exemple cité dans le brief initial du projet).

## Scénario

Sur `PC-user1`, passage de la carte réseau en IP fixe avec une passerelle volontairement erronée :
- IP : `10.10.10.150`
- Masque : `255.255.255.0`
- Gateway : `10.10.10.99` (au lieu de la vraie, `10.10.10.1`)

## Symptôme

```
ping 10.10.10.1   → ✅ succès (réseau local)
ping 8.8.8.8      → ❌ échec attendu (Internet)
```

## Premier test faussé

Au premier essai, `ping 8.8.8.8` réussissait malgré tout, contrairement à ce qu'impliquait la configuration de gateway erronée.

### Diagnostic
```
ipconfig
```
a révélé la cause : une **seconde carte réseau** ("Ethernet 2"), reliquat d'un test SSH précédent (branchée sur `intnet-it`), était toujours active avec une IP et une gateway valides sur `10.10.30.0/24`. Windows utilisait cette route alternative pour sortir vers Internet, masquant complètement l'effet de la gateway cassée sur la première carte.

### Point retenu
Sur une machine disposant de plusieurs cartes réseau actives, un problème de gateway sur une interface peut être invisible si une autre interface offre une route de secours valide. Toujours vérifier `ipconfig` en entier (pas uniquement la carte modifiée) avant de conclure qu'un test de panne n'a pas produit l'effet escompté.

## Test propre (après désactivation de "Ethernet 2")

```
ping 10.10.10.1
```
→ 0% perte, succès normal (réseau local, résolution ARP directe, la gateway n'intervient pas pour une communication sur le même sous-réseau).

```
ping 8.8.8.8
```
→ 75% perte, `Destination host unreachable` renvoyé par la machine elle-même (pas un timeout réseau classique) — la machine sait immédiatement qu'elle ne peut pas router vers l'extérieur, faute de gateway valide dans sa table de routage.

## Diagnostic complémentaire

```
ping 10.10.10.99
```
(la gateway configurée) → timeout, gateway inexistante sur le réseau — confirme que la gateway elle-même est injoignable, cause directe du problème.

## Correction

Remise de la bonne gateway (`10.10.10.1`), ou repassage en DHCP automatique.

## Vérification

```
ping 8.8.8.8
```
→ de nouveau réussi après correction.

## Leçon retenue

Un ping réussi vers le réseau local n'implique pas un routage fonctionnel vers l'extérieur — la gateway n'intervient que pour le trafic destiné à un réseau distant. En cas de panne Internet avec réseau local fonctionnel, tester systématiquement la gateway elle-même (`ping <IP gateway>`) pour confirmer ou écarter cette piste rapidement, et vérifier l'ensemble des interfaces actives sur la machine testée.
