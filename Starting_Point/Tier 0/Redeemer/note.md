# Note détaillée du déroulement du document "Redeemer Write-up"

## Introduction

Les bases de données sont des collections d'informations organisées qui peuvent être facilement consultées, gérées et mises à jour. Dans la plupart des environnements, les systèmes de bases de données sont très importants car ils communiquent des informations liées à vos transactions de vente, à l'inventaire des produits, aux profils des clients et aux activités marketing.

Il existe différents types de bases de données et l'une d'entre elles est Redis, qui est une base de données en mémoire. Les bases de données en mémoire sont celles qui reposent essentiellement sur la mémoire principale pour le stockage des données (c'est-à-dire que la base de données est gérée dans la RAM du système), contrairement aux bases de données qui stockent les données sur le disque ou les SSD. Comme la mémoire principale est beaucoup plus rapide que la mémoire secondaire, le temps de récupération des données dans le cas des bases de données en mémoire est très court, offrant ainsi des temps de réponse très efficaces et minimaux.

Les bases de données en mémoire comme Redis sont généralement utilisées pour mettre en cache les données fréquemment demandées afin de les récupérer rapidement. Par exemple, si un site web affiche des prix sur sa page d'accueil, le site peut d'abord vérifier si les prix nécessaires sont dans Redis, sinon, il vérifie la base de données traditionnelle (comme MySQL ou MongoDB). Lorsque la valeur est chargée depuis la base de données, elle est ensuite stockée dans Redis pour une période plus courte (secondes, minutes ou heures), afin de traiter toutes les demandes similaires qui arrivent pendant cette période. Pour un site avec beaucoup de trafic, cette configuration permet une récupération beaucoup plus rapide pour la majorité des demandes, tout en ayant un stockage stable à long terme dans la base de données principale.

Ce lab se concentre sur l'énumération d'un serveur Redis à distance puis sur le vidage de sa base de données afin de récupérer le flag. Dans ce processus, nous apprenons à utiliser l'utilitaire en ligne de commande `redis-cli` qui aide à interagir avec le service Redis. Nous apprenons également quelques commandes de base de `redis-cli`, utilisées pour interagir avec le serveur Redis et la base de données clé-valeur.

## Introduction au Pentesting

### 1. Étape de l'Énumération
**Objectif** : Documenter l'état actuel de la cible pour en apprendre le plus possible.
- Utilisation de `nmap` pour scanner les ports ouverts et déterminer les services en cours d'exécution.

## Énumération Détailée

### 1. Ping de la cible
- Utilisez `ping {IP_de_la_cible}` pour vérifier la connexion. Arrêtez avec CTRL+C après deux réponses réussies.

### 2. Scannez les ports ouverts
- **Commande** : `nmap -sV -p- {IP_de_la_cible}` pour scanner tous les ports et identifier les services en cours d'exécution.
- **Exemple** : Port 6379/tcp ouvert, utilisant le service Redis.

## Obtention d'un Point d'Appui

### 1. Connexion Redis
- **Redis** : Redis (REmote DIctionary Server) est une base de données NoSQL avancée open-source utilisée comme base de données, cache et courtier de messages.
- **Fonctionnement** : Redis stocke les données sous forme de paires clé-valeur dans la RAM pour un accès rapide. Il sauvegarde également les données sur disque pour garantir la persistance.
- **Sécurité** : Redis nécessite une authentification pour accéder aux données stockées. Les erreurs de configuration peuvent permettre des connexions non sécurisées.

### 2. Utilisation des scripts et options de commande
- Pour en savoir plus sur les capacités d'un script ou d'une commande, tapez son nom suivi du commutateur `-h` ou `--help`. Par exemple, `redis-cli --help`.
- Pour spécifier l'hôte ciblé pour la requête de connexion, utilisez `redis-cli -h {IP_de_la_cible}`.

### 3. Connexion au serveur Redis
En exécutant la commande ci-dessus, nous nous connectons au serveur Redis. Voici quelques commandes utiles :
- **info** : Retourne des informations et des statistiques sur le serveur Redis.
- **select {index}** : Sélectionne la base de données logique Redis par son index.
- **keys *** : Liste toutes les clés présentes dans la base de données sélectionnée.
- **get {clé}** : Affiche la valeur stockée pour une clé donnée.

### 4. Exemples de commandes Redis
- Pour se connecter : `redis-cli -h {IP_de_la_cible}`
- Pour obtenir des informations sur le serveur : `info`
- Pour sélectionner la base de données : `select 0`
- Pour lister les clés : `keys *`
- Pour obtenir la valeur d'une clé : `get {clé}`

## Accès au Système

### 1. Exploration du système
- Utilisez la commande `keys *` pour lister les clés dans la base de données sélectionnée.
- Utilisez la commande `get {clé}` pour obtenir la valeur de chaque clé. Par exemple, `get flag` pour obtenir la valeur de la clé flag contenant la valeur de hachage (flag) nécessaire pour valider la tâche sur Hack The Box.

### 2. Soumission de la Flag
- Copiez la valeur de hachage obtenue et collez-la sur la page du lab Starting Point pour obtenir la propriété de la machine.
