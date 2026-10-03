# Schéma d'architecture

Ce dossier doit contenir :
- `network-diagram.png` — export visuel du schéma réseau (capture d'écran de ton outil de diagramme, ou export du .drawio ci-dessous)
- `topology.drawio` — fichier source éditable (draw.io / diagrams.net)

## Schéma textuel de référence

```
                              INTERNET
                                  │
                           ┌──────┴──────┐
                           │  OPNsense   │
                           │ Firewall/RTR│
                           └──────┬──────┘
                                  │
                ┌─────────────────┼─────────────────┐
                │                 │                 │
           LAN (Users)      SERVERS            IT / Admin
          10.10.10.0/24    10.10.20.0/24      10.10.30.0/24
                │                 │                 │
           PC-user1         SRV-Ubuntu1         Poste admin
           (Windows)        DC-Server1          (test SSH/VPN)
                             (Nginx, SSH)
                             (AD, DNS)
                                                      │
                                              WireGuard VPN
                                             10.10.99.0/24
                                                      │
                                           PC-Admin-Remote
                                          (poste "à la maison")
```

## Comment générer le schéma visuel

1. Va sur [draw.io](https://app.diagrams.net/) (ou l'app diagrams.net)
2. Reproduis le schéma ci-dessus avec les formes réseau standards (routeur, switch, serveurs, postes)
3. Exporte en `.png` → place-le ici sous le nom `network-diagram.png`
4. Sauvegarde le fichier source `.drawio` dans ce même dossier

Alternative rapide : une capture d'écran annotée du Dashboard OPNsense (`Interfaces > Overview`) complétée d'une légende peut aussi servir de point de départ.
