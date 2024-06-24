# Note détaillée du déroulement du document "Dancing Write-up"

## Introduction
Le document "Dancing Write-up" fournit des instructions pour débuter avec Hack The Box, une plateforme de pentesting. Il se concentre sur la machine "Dancing" et inclut des étapes détaillées pour se connecter à un réseau VPN, scanner des ports, et exploiter des services vulnérables.

## Introduction au Pentesting

### 1. Étape de l'Énumération
**Objectif** : Documenter l'état actuel de la cible pour en apprendre le plus possible.
- Utilisation de `nmap` pour scanner les ports ouverts et déterminer les services en cours d'exécution.

## Énumération Détailée

### 1. Ping de la cible
- Utilisez `ping {IP_de_la_cible}` pour vérifier la connexion. Arrêtez avec CTRL+C après quatre réponses réussies.

### 2. Scannez les ports ouverts
- **Commande** : `nmap -sV {IP_de_la_cible}` pour identifier les services en cours d'exécution.
- **Exemple** : Port 445/tcp ouvert, utilisant le service SMB.

## Obtention d'un Point d'Appui

### 1. Connexion SMB
- **SMB** : Server Message Block est un protocole réseau utilisé pour partager des fichiers, des imprimantes et des ports série entre les machines d'un réseau.
- **Fonctionnement** : SMB permet aux applications de lire, créer et mettre à jour des fichiers sur un serveur distant. Les clients SMB doivent fournir un nom d'utilisateur et un mot de passe pour accéder aux partages SMB.
- **Sécurité** : SMB nécessite une authentification. Toutefois, des erreurs de configuration peuvent permettre des connexions anonymes ou invitées.

### 2. Utilisation des scripts et options de commande
- Pour en savoir plus sur les capacités d'un script ou d'une commande, tapez son nom suivi du commutateur `-h` ou `--help`. Par exemple, `smbclient -h` ou `smbclient --help`.
- Pour sélectionner l'hôte ciblé pour la requête de connexion, utilisez `smbclient -L {IP_de_la_cible}`.

### 3. Identification des partages SMB
En exécutant la commande ci-dessus, nous voyons que quatre partages distincts sont affichés. Voici ce qu'ils signifient :
- **ADMIN$** : Les partages administratifs sont des partages réseau cachés créés par la famille de systèmes d'exploitation Windows NT qui permettent aux administrateurs système d'accéder à distance à chaque volume de disque sur un système connecté au réseau. Ces partages ne peuvent pas être supprimés de façon permanente mais peuvent être désactivés.
- **C$** : Partage administratif pour le volume de disque C:\. C'est là que le système d'exploitation est hébergé.
- **IPC$** : Le partage de communication inter-processus. Utilisé pour la communication entre processus via des canaux nommés et ne fait pas partie du système de fichiers.
- **WorkShares** : Partage personnalisé.

## Obtention d'un Point d'Appui

### 1. Connexion SMB avec les partages
- **Tentatives de connexion** : Essayez de vous connecter aux différents partages SMB identifiés (`ADMIN$`, `C$`, `IPC$`, `WorkShares`) en utilisant la commande `smbclient //{IP_de_la_cible}/{nom_du_partage}`. Si le serveur permet une connexion anonyme, laissez le champ mot de passe vide et appuyez sur Entrée.
- **Exemple réussi** : Une connexion avec le partage `WorkShares` peut réussir si le serveur SMB est mal configuré.

### 2. Exploration du système
- Utilisez la commande `ls` pour lister les fichiers dans le répertoire courant du partage connecté.
- Utilisez la commande `cd {nom_du_répertoire}` pour naviguer dans les répertoires.
- Localisez le fichier `flag.txt` contenant la valeur de hachage (flag) nécessaire pour valider la tâche sur Hack The Box.

### 3. Téléchargement du fichier
- Utilisez la commande `get {nom_du_fichier}` pour télécharger le fichier souhaité. Par exemple, `get flag.txt` pour télécharger le fichier flag.txt dans le répertoire courant de votre machine locale.

### 4. Soumission de la Flag
- Lisez le fichier avec la commande `cat flag.txt`.
- Copiez la valeur de hachage et collez-la sur la page du lab Starting Point pour obtenir la propriété de la machine.
