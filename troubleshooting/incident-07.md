# Incident 07 — "Destination host unreachable" malgré une configuration IP correcte

## Catégorie
Procédure / réflexe de vérification de base

## Contexte

Première vérification de connectivité réseau depuis `DC-Server1` (Windows Server), juste après configuration de son IP fixe (`10.10.20.20/24`).

## Symptôme

```
ping 10.10.20.1
ping 10.10.20.10
```
Les deux commandes retournent immédiatement, pour chaque paquet :
```
Reply from 10.10.20.20: Destination host unreachable.
```
(réponse émise par la machine locale elle-même, pas un timeout réseau)

## Hypothèses testées

1. **Configuration IP incorrecte** → vérifiée via `ipconfig /all` : adresse (`10.10.20.20/24`), masque, gateway (`10.10.20.1`), tous corrects.
2. **Carte réseau VirtualBox mal configurée** → vérifiée par capture de l'onglet Réseau de la VM : "Activer l'interface réseau" coché, Mode "Réseau interne", nom `intnet-servers` correct, "Virtual Cable Connected" coché.

Les deux hypothèses techniques les plus probables ont été écartées après vérification rigoureuse.

## Cause réelle

**`OPNsense-FW` et `SRV-Ubuntu1` — les autres machines du réseau SERVERS, dont la gateway elle-même — étaient simplement éteintes.** Aucune configuration n'était en cause : la machine ne trouvait littéralement personne à qui parler sur le réseau.

## Correction

Démarrage d'`OPNsense-FW` et `SRV-Ubuntu1`.

## Vérification

```
ping 10.10.20.1
```
→ 0% perte, succès immédiat.

## Leçon retenue

Avant tout diagnostic réseau approfondi (vérification de configuration IP, carte réseau virtuelle, règles firewall...), toujours vérifier en premier que **tous les équipements concernés sont bien allumés**. Un réflexe de base, facile à oublier une fois concentré sur des hypothèses techniques plus complexes — ce cas illustre qu'une vérification de premier niveau, bien que triviale, doit systématiquement précéder un diagnostic plus poussé pour éviter de perdre du temps sur de fausses pistes.
