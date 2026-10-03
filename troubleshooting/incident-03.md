# Incident 03 — Conflit DNS Dnsmasq / Unbound (forwarding company.local en échec)

## Catégorie
Service / conflit de port

## Objectif initial

Permettre à tous les postes du réseau (utilisant OPNsense comme DNS unique) de résoudre les noms internes du domaine Active Directory `company.local`, en redirigeant ces requêtes spécifiques vers le contrôleur de domaine (`10.10.20.20`), sans reconfigurer chaque client individuellement.

## Configuration mise en place (a priori correcte)

`Services > Dnsmasq DNS & DHCP > Domains` : entrée `company.local → 10.10.20.20` (conditional forwarding).

## Symptôme

- `nslookup WIN-A5T5IRJE99D.company.local 10.10.20.20` (interrogation directe du contrôleur de domaine) → **succès**
- `nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1` (via OPNsense) → **échec** :
```
*** OPNsense.internal can't find WIN-A5T5IRJE99D.company.local: Non-existent domain
```

## Démarche de diagnostic

1. Vérification de la configuration Dnsmasq (champs Domain / IP or Host) → syntaxe correcte, rien à corriger.
2. Forcer "Apply" et redémarrage manuel du service Dnsmasq (bouton de contrôle du service) → aucun changement observé.
3. **Capture tcpdump côté OPNsense**, sur l'interface SERVERS, pendant un nouveau test :
```
tcpdump -i em2 port 53 -n
```
Résultat : **silence total** — aucun paquet DNS sortant vers `10.10.20.20` pendant la tentative de résolution. Contrairement à l'incident 02 (où le paquet arrivait mais n'était pas traité), ici la requête n'était **même pas émise** par OPNsense vers le contrôleur — signe que la requête cliente n'atteignait jamais le bon processus en interne sur OPNsense.
4. Vérification du champ `Services > Dnsmasq DNS & DHCP > General > Listen port` : configuré sur **`53053`**, pas le port DNS standard (`53`). Un client interroge par défaut le port 53 ; Dnsmasq, en écoute sur 53053, ne recevait donc jamais la requête initiale du client.
5. Vérification de `Services > Unbound DNS` : service **activé** en parallèle de Dnsmasq.

## Cause

**Deux serveurs DNS actifs simultanément sur OPNsense** — Dnsmasq (port personnalisé 53053) et Unbound DNS (port standard 53), sans coordination entre eux. Les clients, interrogeant le port 53 standard par défaut, recevaient en réalité les réponses d'**Unbound** — qui ignore totalement la règle de forwarding configurée dans Dnsmasq, d'où le "Non-existent domain". La configuration de forwarding, pourtant syntaxiquement correcte, était appliquée au mauvais service — celui qui n'était jamais réellement sollicité par le trafic client.

## Correction

1. Désactivation d'Unbound DNS (`Services > Unbound DNS > General > Enable` décoché, Save)
2. Remise de Dnsmasq sur le port standard : `Services > Dnsmasq DNS & DHCP > General > Listen port` = `53`, Save
3. Redémarrage forcé du service Dnsmasq

## Vérification

```
nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1
```
→ réponse correcte (`10.10.20.20`).

## Leçon retenue

Sur OPNsense, plusieurs services DNS (Dnsmasq, Unbound, Kea DHCP) peuvent coexister et être activés indépendamment, sans avertissement de conflit. Il faut s'assurer qu'un seul service DNS "fait réellement autorité" sur le port standard 53 — sinon une configuration par ailleurs correcte sur un service n'a aucun effet sur le trafic réellement traité par l'autre. C'est un piège silencieux qui ne se révèle qu'en croisant la configuration (Listen port) avec une capture réseau montrant l'absence totale de requête sortante.
