# Note détaillée du déroulement du document "Crocodile Write-up"

## Introduction

Le niveau I consiste à explorer des vecteurs d'exploitation qui se chaînent pour offrir la possibilité de prendre pied sur la cible d'un service à un autre. Des identifiants peuvent être trouvés quelque part dans un dossier accessible publiquement, permettant de se connecter via une shell distante non surveillée. Un service mal configuré pourrait divulguer des informations permettant d'usurper l'identité numérique d'une victime. De nombreuses possibilités existent dans le monde réel. Cependant, nous allons commencer par des exemples plus simples.

Dans cet exemple, nous examinerons une configuration d'accès non sécurisée sur FTP et une connexion administrative pour un site web. Décomposons ce vecteur et analysons ses composants.

## Énumération

Nous commencerons par l'énumération de la cible. Notre première étape est, comme toujours, un scan nmap approfondi. En utilisant les deux options suivantes pour le scan, nous nous assurons que notre script nmap analyse le service exécuté sur chaque port trouvé en état ouvert et retourne une valeur de version de service principalement exacte dans la sortie et que tous les scripts d'analyse par défaut sont exécutés contre la cible, car nous ne sommes pas contraints sur le niveau d'intrusion de notre scan. En exécutant le scan comme mentionné, nous pouvons recevoir des résultats comme ci-dessous, avec des extraits de répertoires que le scan a même trouvés pour nous !

- **Commande** : `nmap -sC -sV -p- {IP_de_la_cible}` pour scanner tous les ports et identifier les services en cours d'exécution.
- **Exemple** : Ports ouverts 21 et 80. Le port 21 est dédié au FTP (File Transfer Protocol), ce qui signifie que son utilisation principale est de transférer des fichiers entre hôtes sur le même réseau.

## Obtention d'un Point d'Appui

### 1. Connexion FTP
- **FTP** : Le protocole de transfert de fichiers (FTP) est un protocole de communication standard utilisé pour transférer des fichiers informatiques d'un serveur à un client sur un réseau informatique.
- Les utilisateurs peuvent se connecter au serveur FTP anonymement si le serveur est configuré pour le permettre, ce qui signifie que nous pourrions l'utiliser même si nous n'avons pas de justificatifs valides. D'après notre résultat de scan nmap, le serveur FTP est en effet configuré pour permettre une connexion anonyme.

### 2. Connexion au serveur FTP
Pour se connecter au serveur FTP distant, vous devez spécifier l'adresse IP de la cible (ou le nom d'hôte), comme indiqué sur la page du laboratoire Starting Point. L'invite nous demandera ensuite nos identifiants de connexion, où nous pouvons remplir le nom d'utilisateur `anonymous`. Dans notre cas, le serveur FTP ne demande pas de mot de passe, et saisir le nom d'utilisateur `anonymous` suffit pour recevoir le code 230, connexion réussie.

- **Commande** : `ftp {IP_de_la_cible}` puis connectez-vous avec `anonymous` comme nom d'utilisateur.

Une fois connecté, vous pouvez taper la commande `help` pour vérifier les commandes disponibles.

Nous utiliserons `dir` et `get` pour lister les répertoires et manipuler les fichiers stockés sur le serveur FTP. Avec la commande `dir`, nous pouvons vérifier le contenu de notre répertoire actuel sur l'hôte distant, où deux fichiers intéressants attirent notre attention. Ils semblent être des fichiers restants de la configuration d'un autre service sur l'hôte, probablement le serveur web HTTPD.

- **Téléchargement des fichiers** : Utilisez la commande `get {nom_du_fichier}` pour télécharger les fichiers souhaités. Par exemple, `get file1.txt` et `get file2.txt`.

La terminaison de la connexion FTP peut être effectuée en utilisant la commande `exit`.

### 3. Connexion au serveur web

Après avoir obtenu les identifiants, la prochaine étape consiste à vérifier s'ils sont utilisés sur le service FTP pour un accès élevé ou sur le serveur web fonctionnant sur le port 80 découvert lors du scan nmap. La tentative de connexion avec l'un des identifiants sur le serveur FTP renvoie le code d'erreur 530, ce serveur FTP est uniquement anonyme. Pas de chance ici, donc nous pouvons quitter le shell du service FTP.

Cependant, nous avons une option restante. Lors du scan nmap, le service fonctionnant sur le port 80 a été signalé comme Apache httpd 2.4.41, un serveur HTTP Apache. En saisissant l'adresse IP de la cible dans la barre de recherche URL de notre navigateur, nous obtenons cette page web. Elle semble être une vitrine pour une entreprise d'hébergement de serveurs.

Nous pouvons utiliser un outil comme Gobuster pour trouver des pages et des répertoires intéressants. Utilisez les options suivantes pour le script afin d'obtenir les résultats les plus rapides et les plus précis :
- **Commande** : `gobuster dir --url http://{IP_de_la_cible} --wordlist /path/to/wordlist -x php,html`

Un des fichiers les plus intéressants que Gobuster a récupéré est la page /login.php. En naviguant manuellement vers l'URL sous la forme http://{IP_de_la_cible}/login.php, nous sommes confrontés à une page de connexion demandant une combinaison nom d'utilisateur/mot de passe.

Si les listes d'identifiants que nous avons trouvées avaient été plus longues, nous aurions pu utiliser un module Metasploit ou un script de force brute de connexion pour exécuter les combinaisons des deux listes plus rapidement que le travail manuel. Dans ce cas, cependant, les listes sont relativement courtes, nous permettant de tenter de nous connecter manuellement.

### 4. Connexion manuelle

Après avoir essayé plusieurs combinaisons nom d'utilisateur/mot de passe, nous parvenons à nous connecter et nous sommes confrontés à un panneau d'administration du gestionnaire de serveur. Une fois ici, un attaquant pourrait manipuler le site web de la manière qu'il souhaite, causant des ravages pour les utilisateurs et les propriétaires ou extraire plus d'informations qui l'aideraient à prendre pied sur les serveurs hébergeant la page web.

Nous avons réussi à obtenir le flag ! Il est affiché en haut du panneau d'administration.

### 5. Soumission de la Flag

Copiez la valeur de hachage obtenue et collez-la sur la page du lab Starting Point pour obtenir la propriété de la machine.
