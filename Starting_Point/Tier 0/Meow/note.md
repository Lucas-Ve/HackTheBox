# Note détaillée du déroulement du document "Meow Write-up"

## Introduction
Le document "Meow Write-up" fournit des instructions pour débuter avec Hack The Box, une plateforme de pentesting. Il se concentre sur la machine "Meow" et inclut des étapes détaillées pour se connecter à un réseau VPN, scanner des ports, et exploiter des services vulnérables.

## Introduction au Pentesting

### 1. Étape de l'Énumération
**Objectif** : Documenter l'état actuel de la cible pour en apprendre le plus possible.
- Utilisation de `nmap` pour scanner les ports ouverts et déterminer les services en cours d'exécution.

## Énumération Détailée

### 1. Ping de la cible
- Utilisez `ping {IP_de_la_cible}` pour vérifier la connexion. Arrêtez avec CTRL+C après quatre réponses réussies.

### 2. Scannez les ports ouverts
- **Commande** : `nmap -sV {IP_de_la_cible}` pour identifier les services en cours d'exécution.
- **Exemple** : Port 23/tcp ouvert, utilisant le service telnet.

## Obtention d'un Point d'Appui

### 1. Connexion Telnet
- **Telnet** : Telnet (TELetype NETwork) est un protocole réseau utilisé pour fournir une interface de ligne de commande pour la gestion à distance des dispositifs. Il a été largement utilisé avant l'adoption de SSH (Secure Shell), qui offre une communication chiffrée.
- **Fonctionnement** : Telnet permet de se connecter à un hôte distant sur un réseau en utilisant le protocole TCP/IP. Une fois connecté, vous pouvez envoyer des commandes au serveur telnet comme si vous étiez physiquement présent sur la machine.
- **Sécurité** : Telnet n'est pas sécurisé car les données, y compris les mots de passe, sont transmises en texte clair. Cela le rend vulnérable aux interceptions et aux attaques de type "man-in-the-middle".

### 2. Brute-force manuel
- **Commande de connexion Telent** : Utilisez la commande `telnet {IP_de_la_cible}` pour vous connecter au service telnet de la cible.
- **Tentatives de connexion** : Étant donné que certains dispositifs réseau ou hôtes peuvent avoir des comptes avec des mots de passe par défaut ou vides, vous pouvez essayer de vous connecter en utilisant des noms d'utilisateur courants comme `admin`, `administrator`, et `root`.
- **Exemple réussi** : Après plusieurs tentatives, une connexion avec l'utilisateur `root` sans mot de passe peut réussir, ce qui vous donne accès au système.

## Accès au Système

### 1. Exploration du système
- Utilisez la commande `ls` pour lister les fichiers dans le répertoire courant.
- Localisez le fichier `flag.txt` contenant la valeur de hachage (flag) nécessaire pour valider la tâche sur Hack The Box.

### 2. Soumission de la Flag
- Lisez le fichier avec la commande `cat flag.txt`.
- Copiez la valeur de hachage et collez-la sur la page du lab Starting Point pour obtenir la propriété de la machine.
