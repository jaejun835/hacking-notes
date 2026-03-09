TryHackMe link → https://tryhackme.com/room/attacktivedirectory

Today I worked through the TryHackMe Attacktive Directory room.

### [What tool will allow us to enumerate port 139/445?]

Port 139 typically runs NetBIOS; port 445 runs SMB. Think of NetBIOS as essentially an older version of the SMB service.

To enumerate both ports, `enum4linux` is the right tool:

```
enum4linux -a <IP>
```

**Answer: enum4linux**

### [What is the NetBIOS-Domain Name of the machine?]

The enum4linux scan revealed the domain name as **THM-AD**.

The NetBIOS domain name is critical for mapping Active Directory structure — it helps you find the Domain Controller (the top of the hierarchy).

![1](https://github.com/jaejun835/hacking-notes/blob/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

**Answer: THM-AD**

### [What invalid TLD do people commonly use for their Active Directory Domain?]

A domain is essentially a system that groups and manages all the computers and users within an organization.

The most commonly used (and invalid) TLD for internal AD domains is `.local`.

**Answer: .local**

### [What command within Kerbrute will allow us to enumerate valid usernames?]

To enumerate usernames, use the `userenum` subcommand:

```
kerbrute userenum --dc <IP> -d spookysec.local user.txt
```

**Answer: userenum**

### [What notable account is discovered? (These should jump out at you)]

Kerbrute found 16 results. Since AD is case-insensitive, many are duplicates — there are really only 8 unique valid accounts.

Of those, `svc-admin` is the standout account to pursue.

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/2.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

**Answer: svc-admin**

### [What is the other notable account is discovered? (These should jump out at you)]

Among the accounts, `backup` is also a high-value target.

Backup accounts tend to be privileged: they need to run automatically via scripts, so their passwords are often stored in plaintext somewhere, Kerberos pre-authentication may be disabled, or they may hold DCSync-level rights.

**Answer: backup**

### [We have two user accounts that we could potentially query a ticket from. Which user account can you query a ticket from with no password?]

This is about **AS-REP Roasting** — request a ticket without a password, crack it offline.

```
impacket-GetNPUsers spookysec.local/ -usersfile user.txt -dc-ip <IP> -format hashcat -request
```

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/3.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

Roasting confirmed `svc-admin` as the vulnerable account.

**Answer: svc-admin**

### [What mode is the hash?]

The hash prefix `$krb5asrep$23$` means hashcat mode 18200.

**Answer: 18200**

### [Now crack the hash with the modified password list provided, what is the user accounts password?]

```
hashcat -m 18200 ./hash.txt ./password.txt
```

![4](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/4.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

**Answer: management2005**

### [What utility can we use to map remote SMB shares?]

**Answer: smbclient**

### [Which option will list shares?]

The `-L` option (short for List).

**Answer: -L**

### [How many remote shares is the server listing?]

```
smbclient -L //10.48.185.32 -U svc-admin
```

![5](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/5.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

6 shares returned.

**Answer: 6**

### [There is one particular share that we have access to that contains a text file. Which share is it?]

The `backup` share contained a text file.

![6](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/6.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

**Answer: backup**

### [What is the content of the file?]

**Answer: YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw**

### [Decoding the contents of the file, what is the full contents?]

The content is Base64 — decode it:

```
cat hash_1.txt | base64 -d
```

![7](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/7.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

**Answer: backup@spookysec.local:backup2517860**

### [What method allowed us to dump NTDS.DIT?]

NTDS.DIT is the Domain Controller's core database — it holds everything.

DCSync is a legitimate DC-to-DC sync feature. Since the `backup` account holds DCSync privileges, it can impersonate a DC and request all hashes from the real one.

![8](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/8.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

![9](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/9.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

The output confirms it: `[*] Using the DRSUAPI method to get NTDS.DIT secrets`.

> Side note: with the krbtgt hash you could also launch a Golden Ticket attack.
> 

**Answer: DRSUAPI**

### [What is the Administrators NTLM hash?]

Dump format breakdown:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
              │   └─ LM hash                             └─ NTLM hash (this one matters)
              └─ RID
```

**Answer: 0e0363213e37b94221497260b0bcb4fc**

### [What method of attack could allow us to authenticate as the user without the password?]

**Answer: Pass the Hash**

### [Using a tool called Evil-WinRM what option will allow us to use a hash?]

**Answer: -H**
