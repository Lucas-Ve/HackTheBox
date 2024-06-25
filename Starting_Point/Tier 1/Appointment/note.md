# Note détaillée du déroulement du document "Appointment Write-up"

## Introduction

"Appointment" est une machine principalement orientée vers les applications web. Plus précisément, nous allons apprendre à réaliser une injection SQL contre une application web avec une base de données SQL. Notre cible est un site web avec des capacités de recherche dans une base de données backend contenant des éléments consultables vulnérables à ce type d'attaque. Tous les éléments de cette base de données ne doivent pas être visibles par tous les utilisateurs, donc différents privilèges sur le site web donneront des résultats de recherche différents.

Hypothétiquement, un administrateur du site web pourra rechercher des utilisateurs, leurs e-mails, informations de facturation, adresses de livraison, etc. En revanche, un simple utilisateur ou un visiteur non authentifié ne pourra chercher que les produits en vente. Ces tables d'informations seront séparées. Cependant, pour un attaquant ayant des connaissances en vulnérabilités des applications web, notamment l'injection SQL, cette séparation entre les tables ne signifie rien, car ils pourront exploiter l'application web pour interroger directement n'importe quelle table de la base de données SQL du serveur web.

Un excellent exemple de fonctionnement typique d'un service SQL est le processus de connexion utilisé pour tout utilisateur. Chaque fois que l'utilisateur souhaite se connecter, l'application web envoie les informations de connexion (combinaison nom d'utilisateur/mot de passe) au service SQL, les comparant avec les entrées stockées dans la base de données pour cet utilisateur spécifique. Si le nom d'utilisateur et le mot de passe spécifiés correspondent à une entrée dans la base de données, le service SQL le renvoie à l'application web, qui connecte alors l'utilisateur, lui donnant accès aux parties restreintes du site web. Après la connexion, l'application web attribue à l'utilisateur une permission spéciale sous forme de cookie ou de jeton d'authentification associant sa présence en ligne à sa présence authentifiée sur le site. Ce cookie est stocké à la fois localement, sur le stockage du navigateur de l'utilisateur, et sur le serveur web.

Ensuite, si l'utilisateur souhaite rechercher des éléments listés sur la page, il saisira le nom de l'objet dans une barre de recherche, ce qui déclenchera le même service SQL pour exécuter la requête SQL au nom de l'utilisateur. Si une entrée pour l'objet recherché existe dans la base de données, généralement sous une table différente, les informations associées sont récupérées et envoyées à l'application web pour être présentées à l'utilisateur sous forme d'images, de textes, de liens, etc.

Les sites web utilisent des bases de données telles que MySQL, MariaDB ou d'autres car les données qu'ils collectent ou servent doivent être stockées quelque part. Les données peuvent être des noms d'utilisateur, des mots de passe, des publications, des messages ou des ensembles plus sensibles tels que les informations personnellement identifiables (PII), protégées par les lois internationales sur la confidentialité des données. Toute entreprise qui néglige de protéger les PII de ses utilisateurs se voit infliger de lourdes amendes par les régulateurs internationaux et les agences de protection des données.

L'injection SQL est une méthode courante d'exploitation des pages web utilisant des instructions SQL pour récupérer et stocker des données utilisateur. Si elle est mal configurée, cette attaque permet d'exploiter la vulnérabilité d'injection SQL bien connue, qui est très dangereuse. De nombreuses techniques permettent de se protéger contre les injections SQL, comme la validation des entrées, les requêtes paramétrées, les procédures stockées et la mise en œuvre d'un pare-feu d'application web (WAF) sur le périmètre du réseau du serveur. Cependant, il existe des instances où aucune de ces solutions n'est en place, d'où la prévalence de ce type d'attaque, selon la liste des 10 principales vulnérabilités web de l'OWASP.

## Énumération

Tout d'abord, nous effectuons un scan nmap pour trouver les ports ouverts et disponibles ainsi que leurs services. Si aucun autre indicateur n'est spécifié dans la syntaxe de la commande, nmap scannera les 1000 ports TCP les plus courants pour les services actifs. Cela nous convient dans notre cas.

De plus, nous aurons besoin des privilèges super-utilisateur pour exécuter la commande ci-dessous avec les indicateurs -sC ou -sV. En effet, le scan de script (-sC) et la détection de version (-sV) sont considérés comme des méthodes plus intrusives de scan de la cible. Cela augmente la probabilité d'être détecté par un dispositif de sécurité périmétrique sur le réseau de la cible.

La seule porte ouverte détectée est la porte 80 TCP, qui exécute le serveur Apache httpd version 2.4.38.

## Connexion au serveur Web

Nous allons ensuite naviguer directement vers l'adresse IP de la cible à partir de notre navigateur, dans notre instance Pwnbox ou Virtual Machine. En saisissant l'adresse IP de la cible dans le champ URL de notre navigateur, nous faisons face à un site web contenant un formulaire de connexion. Les formulaires de connexion sont utilisés pour authentifier les utilisateurs et leur donner accès aux parties restreintes du site web en fonction du niveau de privilège associé au nom d'utilisateur saisi. Comme nous ne connaissons pas de justificatifs spécifiques que nous pourrions utiliser pour nous connecter, nous vérifierons s'il existe d'autres répertoires ou pages utiles pour nous dans le processus d'énumération.

Nous devons tester le formulaire de connexion pour une éventuelle vulnérabilité d'injection SQL.

## Obtention d'un Point d'Appui

### 1. Connexion avec injection SQL
Pour tester l'injection SQL, nous pouvons essayer d'insérer une séquence SQL malveillante dans le champ du nom d'utilisateur du formulaire de connexion.

Par exemple :
- Nom d'utilisateur : `admin'--`
- Mot de passe : `n'importe quel mot de passe`

Ce type d'injection SQL ferme la requête et commente le reste de l'instruction SQL, permettant ainsi de contourner la vérification du mot de passe.

### 2. Exploration du système

Une fois connecté avec succès en utilisant l'injection SQL, nous pouvons explorer les répertoires du serveur web pour trouver des informations sensibles.

### 3. Téléchargement de fichiers

Utilisez des outils comme `wget` ou `curl` pour télécharger les fichiers trouvés qui peuvent contenir des informations précieuses.

### 4. Soumission de la Flag

Copiez la valeur de hachage obtenue et collez-la sur la page du lab Starting Point pour obtenir la propriété de la machine.
