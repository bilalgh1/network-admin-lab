# Règles Firewall — Matrice de sécurité inter-réseaux

## Principe général : deny by default

OPNsense bloque par défaut tout trafic sur une interface tant qu'aucune règle explicite ne l'autorise. **LAN fait exception** : deux règles "Default allow" (IPv4 et IPv6) sont créées automatiquement à l'installation. Les interfaces ajoutées manuellement (SERVERS, IT, WIREGUARD) n'ont, elles, **aucune règle par défaut** — constaté à plusieurs reprises en testant un simple ping vers la gateway avant toute règle créée (échec systématique tant qu'aucune règle Pass n'existe).

## Ordre d'évaluation des règles

OPNsense évalue les règles d'une interface **de haut en bas** ; la première règle qui correspond au trafic s'applique, les suivantes sont ignorées pour ce paquet. Une règle de blocage placée *après* une règle large ("allow all") n'a donc aucun effet. Ce point a été découvert concrètement lors de la création de la règle `Block Users → IT` (voir ci-dessous) — la règle a dû être repositionnée au-dessus des règles "Default allow LAN" par glisser-déposer pour devenir effective.

## Matrice de règles implémentées

| # | Interface | Action | Source | Destination | Port/Protocole | Description / Justification |
|---|---|---|---|---|---|---|
| 1 | LAN | **Block** | LAN network | IT network | any | `Users → IT DENY` — un poste utilisateur compromis ne doit pas pouvoir atteindre le réseau d'administration. Placée en 1ère position (au-dessus des règles "allow all"). |
| 2 | LAN | **Pass** | LAN network | 10.10.20.10 (host) | TCP/80 | `Users → Servers ALLOW` restreint au strict nécessaire : accès au serveur web uniquement sur le port HTTP, pas d'accès complet au serveur (pas de SSH, pas d'autre port). Principe du moindre privilège. |
| 3 | LAN | **Pass** (par défaut install) | LAN network | any | any | Règle "Default allow LAN to any" — permet notamment la sortie Internet (NAT). Conservée, avec les règles plus spécifiques (1, 2) placées au-dessus pour primer sur les cas concernés. |
| 4 | IT | **Pass** | IT network | any | any | `IT → infrastructure ALLOW` — accès large volontaire pour le réseau d'administration, cohérent avec le besoin de gérer/dépanner l'ensemble de l'infrastructure. |
| 5 | SERVERS | **Pass** | SERVERS network | any | any | Autorise la sortie du serveur (mises à jour système, résolution DNS externe, etc.). Sans cette règle, aucune sortie n'est possible (deny by default). |
| 6 | WIREGUARD | **Pass** | WireGuard net (10.10.99.0/24) | IT network | any | VPN admin distant restreint à IT uniquement — complète la restriction déjà imposée au niveau du tunnel (`AllowedIPs`), en défense en profondeur. |
| 7 | WAN | **Pass** | any | WAN address | UDP/51820 | Autorise le handshake WireGuard entrant depuis l'extérieur — nécessaire pour que le VPN fonctionne (deny by default s'applique aussi sur WAN). |

*Guests → Internet ALLOW / Guests → Servers DENY / Guests → IT DENY* : non implémentées dans cette itération, le VLAN Guest n'ayant pas été créé (voir [switching/vlan-config.md](../switching/vlan-config.md) pour la justification).

## Tests de validation effectués

| Test | Résultat attendu | Résultat obtenu |
|---|---|---|
| `ping 10.10.30.1` depuis PC-user1 (Users → IT) | Échec (règle Block) | ✅ 100% perte |
| `ping 8.8.8.8` depuis PC-user1 (Users → Internet) | Succès | ✅ 0% perte |
| `http://10.10.20.10` depuis PC-user1 (Users → Servers, port 80) | Succès | ✅ Page nginx affichée |
| `ssh user@10.10.20.10` depuis un poste sur IT | Succès (règle IT → tout) | ✅ Connexion établie |
| `ping 10.10.30.1` depuis PC-Admin-Remote (VPN → IT) | Succès | ✅ 0% perte |
| `ping 10.10.20.10` / `ping 10.10.10.1` depuis PC-Admin-Remote (VPN → Servers/LAN) | Échec | ✅ 100% perte chacun |

## Piège rencontré — règle qui ne se sauvegardait pas

Voir [troubleshooting/incident-05.md](../troubleshooting/incident-05.md) : lors de la création de la règle `Block Users → IT`, le bouton "Save" ne semblait produire aucun effet. Cause réelle : la VM avait été éteinte puis rallumée entre-temps, la session web/API n'était plus valide silencieusement (pas de message d'erreur explicite affiché). Un simple rechargement de la page a résolu le problème.

## Alternative non utilisée mais documentée — Floating Rules

Si le glisser-déposer pour réordonner une règle échoue dans l'interface, une **Floating Rule** avec l'option "Quick" cochée est évaluée avant les règles d'interface classiques, indépendamment de leur ordre dans le tableau — une alternative fiable pour garantir qu'une règle de blocage prime sur une règle large, sans dépendre du réordonnancement manuel.
