# Incident 06 — Panne d'installation Windows Server

## Catégorie
Installation / virtualisation

## Contexte

Installation de Windows Server 2022 pour la VM `DC-Server1`, destinée à devenir le contrôleur de domaine Active Directory.

## Symptôme 1 — Échec immédiat à l'installation

Dès le lancement du setup, message d'erreur :
```
Windows cannot find the Microsoft Software License Terms. Make sure the installation sources are valid and restart the installation.
```

### Diagnostic
Vérification de l'intégrité de l'ISO (taille cohérente, ~4,69 Go, téléchargé depuis le site officiel Microsoft) → fichier non corrompu. Inspection de la configuration VirtualBox de la VM : présence d'un **contrôleur Floppy avec un fichier "Unattended-....iso"**, généré automatiquement par l'assistant "New Virtual Machine" de VirtualBox 7.x via l'option **"Proceed with Unattended Installation"**, cochée par défaut.

### Cause
Le fichier de réponses (`autounattend.xml`) généré automatiquement par cette fonctionnalité ne correspondait pas exactement à l'édition Windows Server présente sur l'ISO, provoquant cet échec dès le tout début du setup.

### Correction
Suppression de la VM, recréation en **décochant explicitement "Proceed with Unattended Installation"** à l'étape de création — repasse en installation manuelle classique (choix de langue, édition, type d'installation, etc.).

## Symptôme 2 — VM figée pendant l'installation

Après correction du symptôme 1, l'installation reste bloquée à l'étape "Getting files ready for installation (17%)" sans progression visible pendant plusieurs minutes, curseur de souris hôte ne semblant plus réagir dans la fenêtre VM.

### Pistes explorées (sans cause unique confirmée)
- Vérification de la consommation CPU du processus VirtualBoxVM.exe côté hôte
- Vérification des ressources allouées à la VM (RAM, nombre de CPU)
- Vérification de l'ordre de boot (Boot Device Order)
- Tentative de sélection manuelle du périphérique de boot via le menu F12
- Vérification de l'empreinte SHA256 de l'ISO

Aucune de ces pistes n'a permis d'isoler une cause unique et certaine.

### Résolution
Suppression complète de la VM et recréation propre (plutôt que de continuer à déboguer un état potentiellement corrompu). La nouvelle installation, avec les mêmes paramètres (RAM 4096 Mo, 2 CPU, installation manuelle), s'est déroulée sans blocage jusqu'à l'écran de configuration du mot de passe administrateur.

## Vérification

Connexion réussie au bureau Windows Server, Server Manager ouvert normalement.

## Leçon retenue

Deux enseignements distincts :
1. Vérifier et désactiver les fonctionnalités d'installation "automatique"/"non assistée" d'un outil de virtualisation si l'on souhaite une installation manuelle classique et maîtrisée — leur comportement par défaut peut ne pas être compatible avec l'édition exacte d'un ISO.
2. Face à un blocage d'installateur sans cause identifiable malgré plusieurs pistes de diagnostic raisonnables, repartir d'un environnement propre (VM recréée) est parfois plus efficace en termes de temps que de s'acharner à comprendre un état incertain — un compromis pragmatique à savoir reconnaître en environnement professionnel.
