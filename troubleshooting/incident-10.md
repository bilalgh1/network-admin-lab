# Incident 10 — Service DNS arrêté sur le contrôleur de domaine

## Catégorie
Couche application / DNS

## Scénario

Incident volontairement provoqué : arrêt du service DNS Windows sur `DC-Server1`.
```
net stop DNS
```

## Symptôme

Depuis `PC-user1` :
```
ping 10.10.20.20
```
→ 0% perte, le serveur reste parfaitement joignable au niveau réseau.

```
nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1
```
→ échec :
```
DNS request timed out.
    timeout was 2 seconds.
*** Request to OPNsense timed-out
```

## Analyse

Même logique que l'incident 09 : la couche réseau (ICMP) reste saine, seul le service applicatif précis (DNS, port 53) est en cause sur le contrôleur de domaine. OPNsense transmet correctement la requête vers le forwarder configuré (`10.10.20.20`, voir [services/dns.md](../services/dns.md)) mais n'obtient aucune réponse, d'où le timeout observé côté client.

## Diagnostic (côté serveur)

Consultation du statut du service DNS via la console `dnsmgmt.msc`, ou en ligne de commande :
```
sc query DNS
```
→ confirme le service arrêté.

## Correction

```
net start DNS
```

## Vérification

```
nslookup WIN-A5T5IRJE99D.company.local 10.10.10.1
```
→ réponse correcte restaurée (`10.10.20.20`).

## Leçon retenue

Cet incident illustre la chaîne complète de résolution DNS mise en place dans ce projet : client → OPNsense (Dnsmasq, forwarding `company.local`) → contrôleur de domaine (DNS Windows). Une panne à n'importe quel maillon de cette chaîne casse la résolution de bout en bout, même si tous les maillons précédents restent pleinement fonctionnels. D'où l'intérêt de tester chaque étape séparément (résolution directe sur le DC vs via OPNsense, comme pratiqué dans l'incident 03) pour localiser précisément le maillon fautif plutôt que de supposer une cause globale.
