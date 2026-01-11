# CTF – Planning (Writeup)

## Contexte

Dans ce challenge Hack The Box, la machine cible est accessible à l’adresse `10.10.11.68` sous le domaine `planning.htb`.  
Des identifiants initiaux sont fournis, mais l’accès au site principal sur le port 80 n’est pas possible.

L’objectif est d’obtenir un accès initial via un service exposé, puis d’escalader mes privilèges jusqu’à obtenir un accès **root**.

---

## Informations initiales

Identifiants fournis au départ :

- Utilisateur : admin  
- Mot de passe : 0D5oT70Fq13EvB5r  

---

## Reconnaissance

Je commence par tester l’accès au site web sur le port 80, sans succès.  
Je décide donc de rechercher des sous-domaines associés à `planning.htb`.

Je lance un fuzzing DNS avec ffuf :

    ffuf -w /usr/share/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt \
         -u http://10.10.11.68/ \
         -H "Host: FUZZ.planning.htb" \
         -fs 178

Ce fuzzing me permet d’identifier le sous-domaine suivant :

- `grafana.planning.htb`

---

## Accès à Grafana

En accédant à `http://grafana.planning.htb`, je découvre une interface **Grafana**.

Je tente les identifiants initiaux fournis :

- admin / 0D5oT70Fq13EvB5r  

La connexion est **réussie**, ce qui me donne un accès authentifié à l’interface Grafana.

---

## Accès initial – Exploitation de Grafana

Après identification de la version, je constate que Grafana est en version **11.0**, connue pour être vulnérable à une faille permettant l’exécution de commandes arbitraires avec des identifiants valides.

Je trouve sur GitHub un script Python exploitant cette vulnérabilité.  
À l’aide de ce script, j’envoie une commande permettant d’ouvrir un **reverse shell** vers ma machine.

Je reçois alors un shell sur la machine cible.

---

## Post-exploitation – Variables d’environnement

Une fois connecté, j’exécute la commande suivante afin d’inspecter les variables d’environnement :

    env

Je découvre des identifiants stockés en clair :

- Utilisateur : enzo  
- Mot de passe : RioTecRANDEntANT!  

---

## Accès SSH utilisateur

Je teste immédiatement ces identifiants en SSH :

    ssh enzo@10.10.11.68

La connexion est réussie.  
Je récupère alors le **flag utilisateur** (`user.txt`).

---

## Énumération locale

Je tente d’énumérer les privilèges sudo :

    sudo -l

Cette commande n’est pas autorisée pour l’utilisateur `enzo`.  
Je décide donc de lancer **linPEAS** afin d’identifier d’autres vecteurs d’escalade.

L’énumération met en évidence la présence d’un binaire intéressant :

- `/tmp/bash`

---

## Escalade de privilèges

Le binaire `/tmp/bash` possède le bit **SUID** positionné.  
Je l’exécute avec l’option `-p` afin de conserver les privilèges :

    /tmp/bash -p

Cette commande me donne immédiatement un shell avec les privilèges **root**.

Je peux alors accéder au fichier :

    /root/root.txt

et récupérer le **flag root**.

---

## Note alternative

Si le binaire `/tmp/bash` n’avait pas été présent, une autre piste aurait été exploitable :  
un **service cron** exécuté en boucle avec les privilèges root.

Dans ce scénario, il aurait été possible de :
- exploiter le cron
- créer un tunnel
- copier `/bin/bash`
- et positionner le bit SUID pour obtenir un shell root

---

## Chaîne d’attaque récapitulative

- Accès initial impossible sur le port 80
- Découverte du sous-domaine `grafana.planning.htb`
- Authentification Grafana avec creds fournis
- Exploitation Grafana 11.0 (RCE)
- Reverse shell
- Récupération de credentials via variables d’environnement
- Accès SSH utilisateur
- Découverte d’un binaire SUID (`/tmp/bash`)
- Escalade de privilèges et accès root

---

## Conclusion

Ce challenge met en évidence :
- l’importance du fuzzing de sous-domaines
- les risques liés aux services administratifs exposés
- le danger des identifiants stockés en clair dans les variables d’environnement
- l’impact critique des binaires SUID laissés accessibles

L’exploitation combine :
- reconnaissance web
- exploitation d’un service tiers vulnérable
- post-exploitation
- escalade de privilèges locale jusqu’au **root**

---

## Compétences démontrées

- Fuzzing DNS (ffuf)
- Reconnaissance web
- Exploitation de Grafana
- Reverse shell
- Analyse des variables d’environnement
- Accès SSH
- Énumération locale Linux
- Exploitation de binaires SUID
- Escalade de privilèges
