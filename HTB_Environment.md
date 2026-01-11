# CTF – Environment (Writeup)

## Contexte

Dans ce challenge Hack The Box, la machine cible est accessible via le domaine `environment.htb`.  
L’objectif est d’obtenir un accès initial à l’application web, puis d’escalader mes privilèges jusqu’à obtenir un accès **root**.

---

## Reconnaissance
Je commence par un nmap :

    nmap -sC -sV -sS IP
    
Je commence par de l’énumération et du fuzzing sur le domaine :

    environment.htb

Je découvre rapidement une route `/login`. En inspectant les réponses et certains messages de debug côté backend, je remarque une logique conditionnelle liée à l’environnement d’exécution de l’application.

Les commentaires indiquent que :
- si l’application tourne en environnement **preprod**
- l’utilisateur est automatiquement connecté en tant qu’administrateur (`user_id = 1`)

---

## Accès initial – Bypass d’authentification via paramètre d’environnement

J’intercepte une requête de connexion et je teste l’ajout d’un paramètre non documenté :

    ?--env=preprod

Ce paramètre force l’application à se comporter comme si elle tournait en environnement de préproduction.  
Résultat :
- l’authentification est contournée
- je suis connecté automatiquement au portail en tant qu’utilisateur **Hish**

Il s’agit d’un **auth bypass logique** basé sur une mauvaise gestion des environnements.

---

## Découverte d’une fonctionnalité d’upload

Une fois connecté, je constate que le portail permet la modification de la photo de profil via un endpoint `/upload`.

Je teste les contrôles en place et constate que :
- le type MIME est faiblement vérifié
- les fichiers sont stockés dans `/storage/files/`

---

## Accès initial – Upload polyglotte et exécution de commandes

Je crée un fichier **polyglotte** :
- image PNG valide
- contenant également du code PHP

Le fichier est accepté par l’application et sauvegardé dans :

    /storage/files/

En accédant directement au fichier uploadé et en ajoutant un paramètre de type :

    ?cmd=<commande>

le code PHP est interprété par le serveur.  
Cela me permet d’exécuter des commandes arbitraires et d’obtenir un **webshell**.

---

## Obtention d’un shell interactif

À partir du webshell, je stabilise l’accès et j’obtiens un shell interactif sur la machine.

Je suis connecté en tant que l’utilisateur :

    hish

---

## Énumération locale

Je commence par vérifier les privilèges sudo :

    sudo -l

Résultat intéressant :
- l’utilisateur `hish` peut exécuter `/usr/bin/systeminfo` avec `sudo`
- les variables d’environnement **ENV** et **BASH_ENV** sont conservées

C’est un point critique exploitable pour une escalade de privilèges.

---

## Escalade de privilèges – Abus de BASH_ENV

Le binaire `systeminfo` est un script shell.  
Lorsque la variable `BASH_ENV` est définie, Bash charge et exécute automatiquement le fichier référencé.

Je crée donc un script malveillant :

    /tmp/test.sh

Contenu du script :
- copie `/bin/bash` vers `/tmp/rootbash`
- applique le bit **SUID**

J’exécute ensuite la commande suivante :

    sudo BASH_ENV=/tmp/test.sh /usr/bin/systeminfo

Le script est interprété avec les privilèges **root**, ce qui crée :

    /tmp/rootbash (SUID root)

---

## Accès root

Il ne me reste plus qu’à exécuter :

    /tmp/rootbash -p

Je récupère alors un shell **root** sur la machine.

---

## Chaîne d’attaque récapitulative

- Découverte du paramètre `--env=preprod`
- Bypass d’authentification (connexion admin)
- Upload d’un fichier polyglotte PHP
- Exécution de commandes via webshell
- Accès shell en tant que `hish`
- Abus de `sudo` + `BASH_ENV`
- Création d’un binaire SUID
- Obtention d’un shell root

---

## Conclusion

Ce challenge met en évidence :
- les dangers d’une mauvaise gestion des environnements (prod / preprod)
- les risques liés aux uploads insuffisamment filtrés
- l’impact critique des variables d’environnement conservées avec sudo
- l’importance de la configuration sécurisée des scripts exécutés en root

L’exploitation combine :
- bypass logique d’authentification
- RCE via upload de fichiers
- escalade de privilèges locale
- abus de mécanismes internes du shell

---

## Compétences démontrées

- Fuzzing et énumération web
- Analyse de logique applicative
- Bypass d’authentification
- Exploitation d’upload de fichiers
- Webshell et stabilisation
- Énumération sudo
- Abus de variables d’environnement (BASH_ENV)
- Escalade de privilèges Linux
