# CTF – CodeTwo (Writeup)

## Contexte

Dans ce challenge, je fais face à une application web exposant un éditeur de code JavaScript accessible après création d’un compte.  
L’éditeur permet d’exécuter du code côté serveur.  
L’objectif est d’obtenir un accès initial, puis d’escalader mes privilèges jusqu’à obtenir un accès **root**.

Toutes les informations sensibles ont été volontairement anonymisées pour publication publique.

---

## Reconnaissance

Je commence par un scan réseau classique afin d’identifier les services exposés :

    nmap -sC -sS -sV 10.10.11.82

Le scan révèle notamment :
- un service HTTP exposé sur le **port 8000**
- une application web permettant l’exécution de code JavaScript via un éditeur intégré

Après création d’un compte, j’accède à l’éditeur et constate que le code soumis est exécuté côté serveur.

---

## Accès initial – Exécution de code via js2py

En analysant l’environnement, j’identifie une vulnérabilité connue liée à **js2py**, permettant l’exécution de code arbitraire via un payload JavaScript malveillant.

Je prépare un listener Netcat sur ma machine :

    nc -lvnp 4444

Puis j’injecte un payload exploitant la vulnérabilité js2py directement dans l’éditeur de code.  
L’exécution du payload me permet d’obtenir un shell sur le serveur distant.

Pour stabiliser le shell :

    script /dev/null

---

## Récupération de données sensibles

Une fois sur la machine cible, je démarre un serveur HTTP local afin d’exposer les fichiers accessibles :

    python3 -m http.server 8080

Depuis ma machine, je télécharge une base de données trouvée sur le serveur :

    wget http://10.10.11.82:8080/users.db

---

## Extraction et craquage des identifiants

En analysant la base `users.db`, je découvre des identifiants utilisateurs, dont un compte nommé **marco** avec un hash de mot de passe.

Je soumets le hash à CrackStation, ce qui me permet de retrouver le mot de passe en clair :

    sweetangelbabylove

Je peux alors me connecter en SSH :

    ssh marco@10.10.11.82

---

## Énumération locale

Une fois connecté en tant que `marco`, je récupère le premier flag (`user.txt`), puis je poursuis l’énumération des privilèges.

Je vérifie les droits sudo :

    sudo -l

Je constate que l’utilisateur `marco` peut exécuter un binaire spécifique nommé **npbackup** avec des privilèges élevés.

---

## Analyse de npbackup

J’affiche l’aide du binaire afin de comprendre son fonctionnement :

    sudo npbackup --help

Le binaire permet d’exécuter des sauvegardes basées sur un fichier de configuration au format **YAML**.

Cela ouvre une possibilité d’abus via une configuration contrôlée par l’utilisateur.

---

## Escalade de privilèges via npbackup

Je crée un fichier de configuration minimal malveillant dans `/tmp`, permettant l’exécution d’une commande arbitraire lors de la sauvegarde.

Lors de l’exécution de la commande npbackup avec cette configuration, un fichier exécutable `/tmp/rootbash` est créé.

Je l’exécute avec l’option `-p` afin de conserver les privilèges :

    /tmp/rootbash -p

Je deviens alors **root** et je récupère le flag final dans `/root/`.

---

## Conclusion

Ce challenge illustre :
- les risques liés à l’exécution de code utilisateur côté serveur,
- l’exploitation de vulnérabilités connues (js2py),
- l’impact critique d’identifiants stockés dans des bases accessibles,
- les dangers des binaires exécutables via sudo reposant sur des fichiers de configuration contrôlables.

L’exploitation combine **RCE web**, **exfiltration de données**, **craquage de mots de passe** et **escalade de privilèges** jusqu’au **root**.

---

## Compétences démontrées

- Reconnaissance réseau (Nmap)
- Exploitation RCE via éditeur JavaScript
- Reverse shell et stabilisation
- Exfiltration de fichiers (HTTP / wget)
- Analyse de bases de données
- Craquage de mots de passe (hash)
- Énumération locale (sudo -l)
- Escalade de privilèges via binaire sudo mal configuré
