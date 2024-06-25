# Note détaillée du déroulement du document "Sequel Write-up"

## Introduction

Apprendre à naviguer dans les bases de données est d'une importance considérable car la plupart des données critiques y sont stockées, y compris les noms d'utilisateur et les mots de passe, qui peuvent potentiellement être utilisés pour obtenir un accès privilégié au système cible. Nous avons déjà abordé le sujet des bases de données dans des documents précédents. Cependant, dans celui-ci, nous allons apprendre à naviguer à travers elles.

Les serveurs web et autres services utilisent des bases de données telles que MySQL, MariaDB ou d'autres technologies pour stocker les données accumulées de manière facilement accessible et bien organisée. Ces données peuvent représenter des noms d'utilisateur, des mots de passe, des publications, des messages, la date exacte à laquelle les utilisateurs se sont inscrits et d'autres informations - en fonction du but du site web. Chaque base de données contient des tables, qui à leur tour contiennent des lignes et des colonnes. Par exemple, si un site web possède une petite section de réseau social et une section de commerce électronique, il y aurait besoin de plusieurs sections séparées qui ne devraient pas être accessibles entre elles :
- Une contenant des informations privées des utilisateurs, telles que des adresses e-mail, des géolocalisations, des historiques de connexion et des adresses IP associées, des noms réels, des informations de carte de crédit, et plus encore.
- Une contenant des informations publiquement consultables telles que des produits, des services, de la musique, des vidéos et d'autres types.

Un excellent exemple de fonctionnement typique d'un service SQL est le processus de connexion utilisé pour tout utilisateur. Chaque fois que l'utilisateur souhaite se connecter, l'application web envoie les informations de connexion (combinaison nom d'utilisateur/mot de passe) au service SQL, les comparant avec les entrées stockées dans la base de données pour cet utilisateur spécifique. Si le nom d'utilisateur et le mot de passe spécifiés correspondent à une entrée dans la base de données, le service SQL le renvoie à l'application web, qui connecte alors l'utilisateur, lui donnant accès aux parties restreintes du site web. Après la connexion, l'application web attribue à l'utilisateur une permission spéciale sous forme de cookie ou de jeton d'authentification associant sa présence en ligne à sa présence authentifiée sur le site. Ce cookie est stocké à la fois localement, sur le stockage du navigateur de l'utilisateur, et sur le serveur web.

Ensuite, si l'utilisateur souhaite rechercher des éléments listés sur la page, il saisira le nom de l'objet dans une barre de recherche, ce qui déclenchera le même service SQL pour exécuter la requête SQL au nom de l'utilisateur. Si une entrée pour l'objet recherché existe dans la base de données, généralement sous une table différente, les informations associées sont récupérées et envoyées à l'application web pour être présentées sous forme d'images, de textes, de liens, etc.

Dans notre cas, cependant, nous n'aurons pas besoin d'accéder au service SQL via l'application web. En scannant la cible, nous trouverons un moyen direct de "parler" au service SQL nous-mêmes.

## Énumération

Tout d'abord, nous effectuons un scan nmap pour trouver les ports ouverts et les services en cours d'exécution :

- **Commande** : `nmap -sC -sV -p- {IP_de_la_cible}` pour scanner tous les ports et identifier les services en cours d'exécution.
- **Exemple** : Port 3306 ouvert, utilisant le service MySQL 5.5.5-10.3.27-MariaDB-0+deb10u1.

## Obtention d'un Point d'Appui

### 1. Connexion MySQL
Pour communiquer avec la base de données, nous devons installer soit MySQL soit MariaDB sur notre machine locale. Pour ce faire, exécutez la commande suivante :
- `sudo apt update && sudo apt install mysql*`

### 2. Connexion au service MySQL
- Testez la connexion sans mot de passe pour vérifier s'il y a une éventuelle mauvaise configuration.
- **Commande** : `mysql -h {IP_de_la_cible} -u root`

Avec un peu de chance, notre connexion est acceptée sans mot de passe requis. Nous sommes placés dans un shell de service MySQL à partir duquel nous pouvons explorer les tables et les données disponibles. Voici quelques commandes essentielles pour la navigation :
- `SHOW databases;` : Affiche les bases de données accessibles.
- `USE {nom_de_la_base};` : Sélectionne la base de données à utiliser.
- `SHOW tables;` : Affiche les tables disponibles dans la base de données actuelle.
- `SELECT * FROM {nom_de_la_table};` : Affiche toutes les données de la table spécifiée.

### 3. Exploration du système
- Utilisez la commande `SHOW databases;` pour afficher les bases de données disponibles.
- Sélectionnez la base de données souhaitée avec `USE {nom_de_la_base};`.
- Affichez les tables de la base de données sélectionnée avec `SHOW tables;`.
- Affichez les données des tables avec `SELECT * FROM {nom_de_la_table};`.

### 4. Soumission de la Flag
- Copiez la valeur de hachage obtenue et collez-la sur la page du lab Starting Point pour obtenir la propriété de la machine.
