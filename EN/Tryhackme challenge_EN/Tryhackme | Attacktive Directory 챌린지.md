tryhackme link → https://tryhackme.com/room/attacktivedirectory

Today I worked through TryHackMe's Attacktive Directory room.

### [What tool will allow us to enumerate port 139/445?]

Port 139 is typically used for NetBIOS, and port 445 is typically used for SMB.

NetBIOS can be thought of simply as the older version of SMB.

To enumerate these two ports, you can use enum4linux.

```
enum4linux -a <IP>
```

Answer: enum4linux

### [What is the NetBIOS-Domain Name of the machine?]

From the enum4linux scan results, we can see that the domain name is THM-AD.

The NetBIOS-Domain plays a very important role when understanding the structure of Active Directory, because it allows us to identify the domain controller (the top of the hierarchy) from the results below.

![1](https://github.com/jaejun835/hacking-notes/blob/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

Answer: THM-AD

### [What invalid TLD do people commonly use for their Active Directory Domain?]

A domain is essentially a system that groups and manages all computers and users belonging to a single company or organization.

The invalid TLD that people commonly use for their Active Directory domain is .local.

Answer: .local

### [What command within Kerbrute will allow us to enumerate valid usernames?]

To enumerate usernames, add the `userenum` option to kerbrute.

```
kerbrute userenum --dc <IP> -d spookysec.local user.txt
```

Answer: userenum

### [What notable account is discovered? (These should jump out at you)]

A total of 16 accounts were found through kerbrute.

However, since AD itself is a case-insensitive system, looking closely at the output reveals duplicate emails.

In reality, only 8 unique accounts are valid, and among them, svc-admin can be leveraged to proceed to the next step.

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/2.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

### [What is the other notable account is discovered? (These should jump out at you)]

Among the accounts found with kerbrute, the backup account can also be useful.

(Due to the nature of backup tasks needing to run automatically via scripts or schedulers, the password is often stored in plaintext somewhere, Kerberos pre-authentication may be disabled, or the account holds powerful privileges like DCSync rights.)

### [We have two user accounts that we could potentially query a ticket from. Which user account can you query a ticket from with no password?]

This question refers to the AS-REP Roasting attack technique.

AS-REP Roasting can be thought of simply as obtaining a ticket without a password and then cracking it offline.

```
impacket-GetNPUsers spookysec.local/ -usersfile user.txt -dc-ip <IP> -format hashcat -request
```

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/3.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

From the roasting results, we can see that svc-admin was the vulnerable account.

Answer: svc-admin

### [What mode is the hash?]

Looking closely at the roasting result hash, it starts with `$krb5asrep$23$`, which means it will be type 18200.

Answer: 18200

### [Now crack the hash with the modified password list provided, what is the user accounts password?]

Using hashcat to crack it, we can immediately obtain the credentials.

```
hashcat -m 18200 ./hash.txt ./password.txt
```

![4](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/4.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

Answer: management2005

### [What utility can we use to map remote SMB shares?]

To map remote SMB shares, simply use smbclient.

Answer: smbclient

### [Which option will list shares?]

Using the `-L` option (short for List) lets you view the list of shared folders.

Answer: -L

### [How many remote shares is the server listing?]

Using smbclient to enumerate the share list, we can confirm a total of 6 shares.

```
smbclient -L //10.48.185.32 -U svc-admin
```

![5](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/5.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

### [There is one particular share that we have access to that contains a text file. Which share is it?]

We can confirm that there is a text file in the backup share.

![6](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/6.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

Answer: backup

### [What is the content of the file?]

As confirmed from the backup results, the file content is YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw.

Answer: YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw

### [Decoding the contents of the file, what is the full contents?]

The backup code hash is base64 type, so decoding it gives us the answer.

```
cat hash_1.txt | base64 -d
```

![7](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/7.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

Answer: backup@spookysec.local:backup2517860

### [What method allowed us to dump NTDS.DIT?]

NTDS.DIT is essentially the core database file of the domain controller.

DCSync is a legitimate feature used to synchronize data between domain controllers, but since the backup account holds DCSync privileges, it can impersonate a DC and request all hashes.

![8](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/8.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

![9](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80/9.Tryhackme%20%7C%20Attacktive%20Directory%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

※ As a side note, a golden ticket attack is also possible using krbtgt.

From the output, we can confirm that the NTLM hash dump succeeded (`[*] Using the DRSUAPI method to get NTDS.DIT secrets`), and that the DRSUAPI method was used.

Answer: DRSUAPI

### [What is the Administrators NTLM hash?]

Among the Administrator hashes, the NTLM hash is 0e0363213e37b94221497260b0bcb4fc.

```jsx
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
              │   └─ LM hash                             └─ NTLM hash (important)
              └─ RID
```

Answer: 0e0363213e37b94221497260b0bcb4fc

### [What method of attack could allow us to authenticate as the user without the password?]

The attack method that allows authenticating as a user without a password is Pass the Hash.

Answer: Pass the Hash

### [Using a tool called Evil-WinRM what option will allow us to use a hash?]

The option in Evil-WinRM that allows using a hash is `-H`.

Answer: -H
