# Oopsie Write-up

## Introduction

Lorsqu'on effectue une évaluation web incluant des mécanismes d'authentification, il est toujours conseillé de vérifier les cookies, les sessions et de comprendre comment fonctionne le contrôle d'accès. Dans de nombreux cas, une attaque d'exécution de code à distance et un point d'appui sur le système peuvent ne pas être réalisables par elles-mêmes, mais plutôt après avoir enchaîné différents types de vulnérabilités et d'exploits. Dans cette machine, nous allons apprendre que les types de vulnérabilités de divulgation d'informations et de contrôle d'accès cassé, même s'ils ne semblent pas très importants, peuvent avoir un grand impact lors de l'attaque d'un système, et donc pourquoi même les petites vulnérabilités comptent.

## Énumération

Nous allons commencer notre énumération en recherchant des ports ouverts à l'aide de l'outil Nmap :

- **Commande** : `nmap -sC -sV {TARGET_IP}`

Nous remarquons que les ports 22 (SSH) et 80 (HTTP) sont ouverts. Nous visitons l'IP en utilisant le navigateur web où nous faisons face à un site web pour l'automobile.

Sur la page d'accueil, il est possible de trouver des informations intéressantes sur la façon d'accéder aux services via la connexion.

Selon ces informations, le site web devrait avoir une page de connexion. Avant de procéder à l'énumération des répertoires et des pages, nous pouvons essayer de mapper le site web en utilisant le proxy Burp Suite pour explorer passivement le site web. Burp Suite est une application de test de sécurité puissante qui peut être utilisée pour effectuer des requêtes web sur des applications web, des applications mobiles et des clients lourds. Burp offre de multiples capacités telles que web crawler, scanner, proxy, repeater, intruder et bien d'autres.

### Utilisation de Burp Suite

Pour commencer, nous allons démarrer Burp Suite et configurer le navigateur pour envoyer le trafic via le proxy. Pour accéder aux paramètres du proxy dans Mozilla Firefox, cliquez sur le menu de Firefox et allez dans Préférences.

Ensuite, nous tapons dans la barre de recherche "proxy" et les paramètres réseau sont présentés. Nous sélectionnons Paramètres...

Puis nous choisissons la configuration manuelle du proxy où nous entrons comme proxy HTTP l'IP 127.0.0.1 et le port 8080 où Burp Proxy écoute.

Note : Il est conseillé de vérifier également l'option "Utiliser ce proxy pour FTP et HTTPS" afin que toutes les requêtes passent par Burp.

Nous devons désactiver l'interception dans Burp Suite car elle est activée par défaut. Naviguez vers l'onglet Proxy, et sous l'onglet Intercept, sélectionnez le bouton où Intercept est activé pour le désactiver.

Maintenant que tout est correctement configuré, nous actualisons la page dans notre navigateur et passons à Burp Suite sous l'onglet Target puis sur l'option Sitemap.

Il est possible de repérer certains répertoires et fichiers qui n'étaient pas visibles lors de la navigation. L'un des répertoires très intéressants est /cdn-cgi/login.

Nous pouvons le visiter dans notre navigateur et nous sommes effectivement présentés avec la page de connexion :

Après avoir essayé quelques combinaisons de nom d'utilisateur/mot de passe par défaut, nous n'avons pas réussi à accéder. Mais il y a aussi une option pour se connecter en tant qu'invité. En essayant cela, nous sommes présentés avec quelques nouvelles options de navigation alors que nous sommes connectés en tant qu'invité :

Après avoir navigué sur les pages disponibles, nous remarquons que la seule intéressante semble être la page des téléchargements (Uploads). Cependant, il n'est pas possible d'y accéder car nous devons avoir des droits de super administrateur :

Nous devons trouver un moyen d'escalader nos privilèges de l'utilisateur invité au rôle de super administrateur. Une façon de tenter cela est de vérifier si les cookies et les sessions peuvent être manipulés.

### Manipulation des Cookies

Il est possible de voir et de modifier les cookies dans Mozilla Firefox en utilisant les outils de développement.

Pour accéder au panneau des outils de développement, nous devons cliquer avec le bouton droit de la souris sur le contenu de la page web et sélectionner Inspecter l'élément(Q).

Ensuite, nous pouvons naviguer vers la section Stockage où les cookies sont présentés. Comme on peut le voir, il y a un cookie role=guest et user=2233, ce qui nous laisse supposer que si nous connaissions le numéro de super administrateur pour la variable utilisateur, nous pourrions accéder à la page de téléchargement.

Nous vérifions l'URL dans la barre de notre navigateur où il y a un identifiant pour chaque utilisateur :

- **URL** : `http://10.129.95.191/cdn-cgi/login/admin.php?content=accounts&id=2`

Nous pouvons essayer de changer la variable id par quelque chose d'autre, par exemple 1, pour voir si nous pouvons énumérer les utilisateurs :

- **URL** : `http://10.129.95.191/cdn-cgi/login/admin.php?content=accounts&id=1`

En effet, nous avons une vulnérabilité de divulgation d'informations que nous pourrions exploiter. Nous connaissons maintenant l'ID d'accès de l'utilisateur administrateur, nous pouvons donc essayer de changer les valeurs dans notre cookie via les outils de développement pour que la valeur utilisateur soit 34322 et la valeur du rôle soit admin. Ensuite, nous pouvons revisiter la page des téléchargements.

### Obtention d'un accès

Nous avons finalement accès au formulaire de téléchargement.

Maintenant que nous avons accès au formulaire de téléchargement, nous pouvons tenter de télécharger un shell inverse PHP. Au lieu de créer le nôtre, nous allons utiliser un shell existant.

Dans Parrot OS, il est possible de trouver des webshells sous le dossier /usr/share/webshells/, cependant, si vous ne l'avez pas, vous pouvez le télécharger à partir d'ici.

Pour cet exercice, nous allons utiliser /usr/share/webshells/php/php-reverse-shell.php.

Bien sûr, nous devons modifier le code ci-dessus pour qu'il corresponde à nos besoins. Nous allons changer les variables $ip et $port pour correspondre à nos paramètres, puis nous tenterons de télécharger le fichier.

### Brute force des répertoires

Nous avons finalement réussi à le télécharger. Maintenant, nous pourrions avoir besoin de brute-forcer les répertoires pour localiser le dossier où les fichiers téléchargés sont stockés, mais nous pouvons aussi deviner. Le répertoire `uploads` semble être une supposition logique. Nous confirmons cela en exécutant également l'outil `gobuster`.

- **Commande** : `gobuster dir --url http://{TARGET_IP}/ --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php`

Le gobuster a immédiatement trouvé le répertoire /uploads. Nous n'avons pas la permission d'accéder au répertoire, mais nous pouvons essayer d'accéder à notre fichier téléchargé.

### Connexion Netcat

Tout d'abord, nous devons configurer une connexion Netcat :

- **Commande** : `nc -lvnp 1234`

Ensuite, nous demandons notre shell via le navigateur :

- **URL** : `http://{TARGET_IP}/uploads/php-reverse-shell.php`

Et vérifiez notre écouteur.

Note : Si notre shell n'est pas là, il a peut-être été supprimé, donc nous devons le télécharger à nouveau.

Nous avons obtenu un shell inverse ! Pour avoir un shell fonctionnel, nous pouvons émettre la commande suivante :

- **Commande** : `python3 -c 'import pty;pty.spawn("/bin/bash")'`

## Mouvement latéral

En tant qu'utilisateur www-data, nous ne pouvons pas faire grand-chose car le rôle a un accès restreint au système. Étant donné que le site web utilise PHP et SQL, nous pouvons énumérer davantage le répertoire web pour des divulgations potentielles ou des mauvaises configurations. Après quelques recherches, nous pouvons trouver des fichiers PHP intéressants sous le répertoire /var/www/html/cdn-cgi/login. Nous pouvons examiner manuellement le code source de toutes les pages ou essayer de rechercher des chaînes intéressantes à l'aide de l'outil grep. Grep est un outil qui recherche des MOTIFS dans chaque FICHIER et imprime les lignes qui correspondent aux motifs. Nous pouvons utiliser `cat *` pour lire tous les fichiers tout en pipant la sortie vers grep où nous fournissons le motif d'une chaîne qui commence par le mot passw et suivi de n'importe quelle chaîne comme par exemple les mots passwd ou password. Nous pouvons également utiliser l'option -i pour ignorer les mots sensibles à la casse comme Password.

- **Commande** : `cat * | grep -i passw*`

Nous avons en effet obtenu le mot de passe : MEGACORP_4dm1n!!. Nous pouvons vérifier les utilisateurs disponibles sur le système en lisant le fichier /etc/passwd afin de tenter une réutilisation de ce mot de passe :

- **Commande** : `cat /etc/passwd`

Nous avons trouvé l'utilisateur robert. Pour se connecter en tant que cet utilisateur, nous utilisons la commande su :

- **Commande** : `su robert`

Malheureusement, ce n'était pas le mot de passe pour l'utilisateur robert. Lisons maintenant les fichiers un par un. Nous allons commencer par db.php qui semble intéressant :

Maintenant que nous avons le mot de passe, nous pouvons nous connecter avec succès et lire le flag user.txt qui se trouve dans le répertoire personnel de robert :

## Escalade de privilèges

Avant d'exécuter un script d'escalade de privilèges ou d'énumération, vérifions les commandes de base pour élever les privilèges comme sudo et id :

Nous observons que l'utilisateur robert fait partie du groupe bugtracker. Voyons s'il y a un binaire dans ce groupe :

- **Commande** : `find / -group bugtracker 2>/dev/null`

Nous avons trouvé un fichier nommé bugtracker. Vérifions quels privilèges et quel type de fichier c'est :

- **Commande** : `ls -la /usr/bin/bugtracker && file /usr/bin/bugtracker`

Il y a un bit suid sur ce binaire, ce qui est une piste d'exploitation prometteuse.

### Exploitation du binaire bugtracker

Nous allons exécuter l'application pour observer son comportement :

L'outil accepte l'entrée de l'utilisateur en tant que nom du fichier qui sera lu à l'aide de la commande cat, cependant, il ne spécifie pas le chemin complet du fichier cat et nous pourrions donc être en mesure d'exploiter cela.

Nous allons naviguer vers le répertoire /tmp et créer un fichier nommé cat avec le contenu suivant :

`/bin/sh`

Nous allons ensuite définir les privilèges d'exécution :

- **Commande** : `chmod +x cat`

Pour exploiter cela, nous pouvons ajouter le répertoire /tmp à la variable d'environnement PATH.

- **Commande** : `export PATH=/tmp:$PATH`

Ensuite, nous vérifions le $PATH :

- **Commande** : `echo $PATH`

Enfin, nous exécutons le bugtracker depuis le répertoire /tmp :

La racine du drapeau peut être trouvée dans le dossier /root :

Nous avons obtenu les deux flags, félicitations !
