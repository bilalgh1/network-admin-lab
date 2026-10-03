# Incident 09 — Service web arrêté (port/service inaccessible)

## Catégorie
Couche application

## Scénario

Incident volontairement provoqué : arrêt du service Nginx sur `SRV-Ubuntu1`.
```
sudo systemctl stop nginx
```

## Symptôme

Depuis `PC-user1` :
```
ping 10.10.20.10
```
→ 0% perte, succès normal (couche réseau/IP intacte, le serveur répond bien aux ICMP).

```
curl http://10.10.20.10
```
→ échec immédiat :
```
curl: (7) Failed to connect to 10.10.20.10 port 80 after 2062 ms: Could not connect to server
```

## Analyse

La différence entre les deux résultats est la clé de cet incident : un ping réussi prouve que la machine est allumée, routée correctement, et que le firewall autorise au moins le trafic ICMP — mais ne dit **rien** sur l'état d'un service applicatif précis. La couche réseau (3) et la couche application (7) doivent être testées séparément.

## Diagnostic (côté serveur)

```
sudo systemctl status nginx
```
→ statut `inactive (dead)`, confirme l'arrêt du service.

## Correction

```
sudo systemctl start nginx
```

## Vérification

```
curl http://10.10.20.10
```
→ page HTML complète "Welcome to nginx!" correctement retournée.

## Leçon retenue

Toujours distinguer les couches lors d'un diagnostic réseau : connectivité (ping, couche 3) ≠ disponibilité d'un service applicatif (couche 7). Un ping réussi ne garantit jamais qu'un service précis fonctionne sur la machine cible ; il faut tester le port/protocole concerné spécifiquement (`curl`, `telnet host port`, ou équivalent) pour confirmer l'état réel du service.
