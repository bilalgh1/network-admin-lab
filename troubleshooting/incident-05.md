# Incident 05 — Session web OPNsense invalidée après reboot VM

## Catégorie
Application / session

## Contexte

Création de la règle firewall `Block Users → IT` via l'interface web d'OPNsense.

## Symptôme

Le formulaire "Edit rule" est rempli correctement (Interface, Action, Source, Destination tous valides), mais le clic sur **Save** ne produit visiblement aucun effet — pas de fermeture de popup, pas de message d'erreur affiché, pas de nouvelle règle dans la liste.

## Hypothèses testées

1. Champ obligatoire manquant (Interface non sélectionnée) → vérifié et corrigé dans un premier temps, mais le souci persistait sur un essai ultérieur similaire.
2. Erreur silencieuse du formulaire, non visible dans l'interface.

## Diagnostic

La VM `OPNsense-FW` avait été **éteinte puis rallumée** entre deux sessions de travail. La session web (cookie/jeton d'authentification côté API) n'était plus valide après ce redémarrage, sans que l'interface web n'affiche d'indication claire de déconnexion ou d'expiration — le formulaire restait affiché normalement, donnant l'impression que tout fonctionnait, alors que les appels API sous-jacents échouaient silencieusement.

## Correction

Rechargement complet de la page (F5) — qui a forcé un nouveau login et rétabli une session valide.

## Vérification

Après rechargement, nouvelle tentative de création de la règle : sauvegarde réussie immédiatement, règle visible dans la liste.

## Leçon retenue

Après tout redémarrage de la VM OPNsense, recharger systématiquement la page web avant de continuer une session de configuration — une session expirée silencieusement peut faire perdre du temps de diagnostic sur un problème qui n'existe pas réellement côté configuration. Un comportement "le clic ne fait rien, sans erreur visible" doit toujours faire penser à vérifier l'état de la session/connexion avant de chercher une cause plus complexe.
