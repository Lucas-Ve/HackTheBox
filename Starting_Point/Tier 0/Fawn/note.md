# Note détaillée du déroulement du document "Fawn Write-up"

## Introduction
Le document "Fawn Write-up" fournit des instructions pour débuter avec Hack The Box, une plateforme de pentesting. Il se concentre sur la machine "Fawn" et inclut des étapes détaillées pour se connecter à un réseau VPN, scanner des ports, et exploiter des services vulnérables.

## Introduction au Pentesting

### 1. Étape de l'Énumération
**Objectif** : Documenter l'état actuel de la cible pour en apprendre le plus possible.
- Utilisation de `nmap` pour scanner les ports ouverts et déterminer les services en cours d'exécution.

## Énumération Détailée

### 1. Ping de la cible
- Utilisez `ping {IP_de_la_cible}` pour vérifier la connexion. Arrêtez avec CTRL+C après quatre réponses réussies.

### 2. Scannez les ports ouverts
- **Commande** : `nmap -sV {IP_de_la_cible}` pour identifier les services en cours d'exécution.
- **Exemple** : Port 21/tcp ouvert, utilisant le service FTP.

## Obtention d'un Point d'Appui

### 1. Connexion FTP
- **FTP** : File Transfer Protocol est un protocole réseau utilisé pour transférer des fichiers entre un client et un serveur sur un réseau informatique.
- **Fonctionnement** : FTP permet aux utilisateurs de télécharger et téléverser des fichiers depuis et vers un serveur.
- **Sécurité** : FTP n'est pas sécurisé car les données, y compris les mots de passe, sont transmises en texte clair. Il peut être sécurisé avec SSL/TLS (FTPS) ou remplacé par SSH File Transfer Protocol (SFTP).

### 2. Brute-force manuel
- **Commande de connexion FTP** : Utilisez la commande `ftp {IP_de_la_cible}` pour vous connecter au service FTP de la cible. Si le serveur permet une connexion anonyme, entrez `anonymous` comme nom d'utilisateur et n'importe quel mot de passe.
- **Tentatives de connexion** : Étant donné que certains dispositifs réseau ou hôtes peuvent avoir des comptes avec des mots de passe par défaut ou vides, vous pouvez essayer de vous connecter en utilisant des noms d'utilisateur courants comme `admin`, `administrator`, et `root`.
- **Exemple réussi** : Une connexion avec un compte anonyme peut réussir si le serveur FTP est mal configuré.

## Accès au Système

### 1. Exploration du système
- Utilisez la commande `ls` pour lister les fichiers dans le répertoire courant.
- Localisez le fichier `flag.txt` contenant la valeur de hachage (flag) nécessaire pour valider la tâche sur Hack The Box.

### 2. Soumission de la Flag
- Lisez le fichier avec la commande `cat flag.txt`.
- Copiez la valeur de hachage et collez-la sur la page du lab Starting Point pour obtenir la propriété de la machine.
