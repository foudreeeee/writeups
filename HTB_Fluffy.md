# CTF – Fluffy (Writeup)

## Contexte

Dans ce challenge Hack The Box de type **Windows / Active Directory**, la machine cible est accessible à l’adresse `10.10.11.69` sous le domaine `fluffy.htb`.  
Des identifiants initiaux sont fournis, permettant de démarrer l’énumération interne du domaine.

L’objectif est d’obtenir un accès utilisateur, puis d’escalader progressivement les privilèges jusqu’à devenir **Domain Administrator**.

---

## Reconnaissance initiale

Je commence par un scan Nmap afin d’identifier les services exposés :

    nmap -sC -sV 10.10.11.69

Les résultats montrent un environnement Windows classique avec :
- LDAP
- SMB
- Kerberos
- Services Active Directory

Des identifiants initiaux sont fournis :

- Utilisateur : `j.fleischman`
- Mot de passe : `J0elTHEM4n1990!`

---

## Énumération SMB

Je commence par lister les partages SMB accessibles :

    smbclient -L 10.10.11.69 -U "fluffy.htb/j.fleischman"

Parmi les partages disponibles, le partage **IT** attire mon attention.  
Je teste l’accès :

    smbclient \\10.10.11.69\IT -U "fluffy.htb/j.fleischman"

Résultat :
- accès **lecture / écriture** autorisé

Cela représente une surface d’attaque critique.

---

## Accès initial – Exploitation CVE-2025-24071 (NTLM Leak)

Le partage IT étant accessible en écriture, j’exploite la vulnérabilité **CVE-2025-24071**.

Principe :
- upload d’un fichier `.library-ms` piégé
- accompagné d’une archive `.zip`
- lorsque la victime interagit avec le fichier, une authentification NTLM sortante est déclenchée

Je dépose les fichiers malveillants sur le partage IT et je lance **Responder** sur ma machine attaquante.

Résultat :
- fuite d’un hash **NTLMv2**
- utilisateur compromis : `p.agila`

---

## Craquage du hash et nouvel accès

Je cracke le hash NTLMv2 récupéré, ce qui me permet d’obtenir le mot de passe :

- Utilisateur : `p.agila`
- Mot de passe : `prometheusx-303`

Je dispose désormais d’un nouvel accès valide au domaine.

---

## Énumération Active Directory (BloodHound)

Avec les identifiants de `p.agila`, je lance une énumération Active Directory à l’aide de **BloodHound**.

Résultat clé :
- `p.agila` peut s’ajouter lui-même au groupe **SERVICE ACCOUNTS**
- ce groupe possède des droits **GenericWrite** sur plusieurs comptes de service :
  - `ca_svc`
  - `ldap_svc`
  - `winrm_svc`

Cela ouvre la voie à une attaque avancée via **Shadow Credentials**.

---

## Abus de Shadow Credentials (Certipy)

Je commence par ajouter `p.agila` au groupe **SERVICE ACCOUNTS**.

Ensuite, j’utilise **Certipy** pour exploiter les **Shadow Credentials** sur le compte `winrm_svc`.

Principe :
- injection d’une clé Kerberos dans l’objet AD de `winrm_svc`
- récupération du **NT hash** du compte

Grâce à ce hash, je peux me connecter à distance via **Evil-WinRM**.

---

## Accès utilisateur via Evil-WinRM

Je me connecte avec succès en tant que `winrm_svc` et je récupère le **premier flag utilisateur**.

À ce stade, j’ai un accès interactif sur la machine.

---

## Analyse des services de certificats AD (AD CS)

Je poursuis l’énumération et j’analyse l’infrastructure **Active Directory Certificate Services**.

Je découvre une vulnérabilité critique :
- **ESC16** sur l’autorité de certification :
  - `fluffy-DC01-CA`

Le compte `p.agila` possède des droits hérités sur le compte `ca_svc`, ce qui rend l’exploitation possible.

---

## Exploitation ESC16 – Abus de certificats

Étapes de l’attaque :

1. Je modifie temporairement l’UPN du compte `ca_svc` pour le définir comme :
   
       administrator

2. Je demande un certificat via un template d’authentification client valide

3. Une fois le certificat obtenu, je restaure immédiatement l’UPN original de `ca_svc` afin de limiter les traces visibles

---

## Accès Domain Administrator

À l’aide du certificat récupéré, j’utilise :

    certipy auth

Cela me permet :
- d’obtenir un TGT Kerberos
- de récupérer le **NT hash de l’administrateur du domaine**

Je dispose désormais d’un accès **Domain Admin** complet.

Je peux alors lire le **flag root**.

---

## Chaîne d’attaque récapitulative

- Accès initial via identifiants fournis
- Énumération SMB (partage IT en écriture)
- Exploitation CVE-2025-24071 (NTLM leak)
- Craquage du hash NTLMv2
- Énumération AD via BloodHound
- Abus de GenericWrite sur comptes de service
- Exploitation Shadow Credentials (Certipy)
- Accès WinRM
- Exploitation AD CS (ESC16)
- Compromission Domain Administrator

---

## Conclusion

Ce challenge met en évidence :
- les risques liés aux partages SMB mal configurés
- l’impact critique des fuites NTLM
- la dangerosité des droits AD mal délégués
- la puissance des attaques modernes contre AD CS

L’exploitation combine des techniques **réalistes**, **actuelles** et **avancées** d’attaque Active Directory.

---

## Compétences démontrées

- Énumération SMB et Active Directory
- Exploitation NTLM (Responder)
- Craquage de hash NTLMv2
- Analyse de graphes AD (BloodHound)
- Abus de permissions AD (GenericWrite)
- Shadow Credentials (Certipy)
- Attaques AD CS (ESC16)
- Accès WinRM et post-exploitation Windows
- Compromission Domain Admin
