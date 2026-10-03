# Service DNS

## Architecture de résolution

```
Client (LAN/SERVERS/IT)
      │  DNS = 10.10.10.1 (OPNsense)
      ▼
OPNsense — Dnsmasq DNS & DHCP (port 53)
      │  requêtes *.company.local → forward
      ▼
DC-Server1 — DNS Windows (Active Directory intégré)
      10.10.20.20
```

Les clients du réseau utilisent OPNsense comme DNS unique. OPNsense résout lui-même les requêtes génériques (vers Internet) et **redirige conditionnellement** les requêtes concernant le domaine `company.local` vers le contrôleur de domaine Active Directory, qui fait autorité sur ce domaine.

## Configuration

- **OPNsense** : `Services > Dnsmasq DNS & DHCP > Domains` — entrée `company.local → 10.10.20.20` (conditional forwarding)
- **DC-Server1** : rôle DNS Server installé automatiquement avec Active Directory Domain Services lors de la promotion en contrôleur de domaine (domaine racine `company.local`, NetBIOS `COMPANY`)

## Incident majeur — conflit entre deux services DNS actifs

Le forwarding, bien que configuré correctement dès le départ, ne fonctionnait pas. La cause réelle : **deux services DNS actifs simultanément sur OPNsense** — Dnsmasq (configuré sur un port non standard, `53053`, visible dans `General > Listen port`) et **Unbound DNS**, activé en parallèle et répondant réellement sur le port standard 53. Les clients interrogeant le port 53 standard recevaient donc les réponses d'Unbound, qui ignorait totalement la règle de forwarding configurée dans Dnsmasq.

Diagnostic complet, capture tcpdump à l'appui, dans [troubleshooting/incident-03.md](../troubleshooting/incident-03.md).

**Correction** : désactivation d'Unbound DNS, remise de Dnsmasq sur le port standard 53.

## Tests de validation

| Test | Résultat |
|---|---|
| `nslookup WIN-A5T5IRJE99D.company.local 10.10.20.20` (direct sur le DC) | ✅ Résout correctement |
| `nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1` (via OPNsense, avant correction) | ❌ "Non-existent domain" |
| `nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1` (via OPNsense, après correction) | ✅ Résout correctement (`10.10.20.20`) |
| `ping google.com` depuis PC-user1 et SRV-Ubuntu1 (résolution externe) | ✅ Fonctionnelle de bout en bout |

## Résilience testée

Le service DNS de `DC-Server1` a été volontairement arrêté (`net stop DNS`) pour observer le comportement en cas de panne — voir [troubleshooting/incident-10.md](../troubleshooting/incident-10.md). Confirme la dépendance de toute la chaîne de résolution interne à ce service : une panne casse la résolution `company.local` de bout en bout, même si OPNsense et le reste du réseau restent fonctionnels.
