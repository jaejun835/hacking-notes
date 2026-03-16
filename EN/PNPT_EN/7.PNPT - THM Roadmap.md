### Phase 1 (Individual Machines, Easy)

| Room | Topics | URL |
| --- | --- | --- |
| Blue | EternalBlue, Meterpreter | [tryhackme.com/room/blue](http://tryhackme.com/room/blue) |
| Ice | Windows service exploitation | [tryhackme.com/room/ice](http://tryhackme.com/room/ice) |
| Kenobi | SMB, PrivEsc | [tryhackme.com/room/kenobi](http://tryhackme.com/room/kenobi) |
| Basic Pentesting | Full enumeration → PrivEsc workflow | [tryhackme.com/room/basicpentestingjt](http://tryhackme.com/room/basicpentestingjt) |
| Lazy Admin | Linux initial access + PrivEsc | [tryhackme.com/room/lazyadmin](http://tryhackme.com/room/lazyadmin) |

### Phase 2 (Intermediate Individual Machines — Boot2Root)

| Room | Topics | URL |
| --- | --- | --- |
| Alfred | Windows, Jenkins RCE, token theft | [tryhackme.com/room/alfred](http://tryhackme.com/room/alfred) |
| HackPark | Windows, Hydra, PrivEsc | [tryhackme.com/room/hackpark](http://tryhackme.com/room/hackpark) |
| GameZone | SQLi, SSH tunneling | [tryhackme.com/room/gamezone](http://tryhackme.com/room/gamezone) |
| Skynet | SMB, RFI, Cronjob PrivEsc | [tryhackme.com/room/skynet](http://tryhackme.com/room/skynet) |
| Anonymous | FTP, Linux PrivEsc | [tryhackme.com/room/anonymous](http://tryhackme.com/room/anonymous) |
| Mr. Robot CTF | Web enumeration, Linux PrivEsc | [tryhackme.com/room/mrrobot](http://tryhackme.com/room/mrrobot) |
| Relevant | SMB, Windows PrivEsc | [tryhackme.com/room/relevant](http://tryhackme.com/room/relevant) |
| Overpass | Linux enumeration, PrivEsc | [tryhackme.com/room/overpass](http://tryhackme.com/room/overpass) |
| Overpass 2 | Forensics + reverse analysis | [tryhackme.com/room/overpass2hacked](http://tryhackme.com/room/overpass2hacked) |
| Overpass 3 | NFS abuse, PrivEsc | [tryhackme.com/room/overpass3hosting](http://tryhackme.com/room/overpass3hosting) |
| Startup | Linux, analyzing unusual services | [tryhackme.com/room/startup](http://tryhackme.com/room/startup) |
| Dogcat | Docker escape, PHP LFI | [tryhackme.com/room/dogcat](http://tryhackme.com/room/dogcat) |

### Phase 3 (PrivEsc Arenas — CTF Style)

| Room | Topics | URL |
| --- | --- | --- |
| Linux PrivEsc Arena | SUID/cron/passwd and more — practical | [tryhackme.com/room/linuxprivescarena](http://tryhackme.com/room/linuxprivescarena) |
| Windows PrivEsc Arena | Services/registry/DLL and more — practical | [tryhackme.com/room/windowsprivescarena](http://tryhackme.com/room/windowsprivescarena) |
| Steel Mountain | Created by Tib3rius, Windows PrivEsc | [tryhackme.com/room/steelmountain](http://tryhackme.com/room/steelmountain) |

### Phase 4 (AD + Network Practical)

| Room | Topics | Scale | URL |
| --- | --- | --- | --- |
| Attacktive Directory | Kerberoasting, ASREPRoast, DC compromise | Single DC | [tryhackme.com/room/attacktivedirectory](http://tryhackme.com/room/attacktivedirectory) |
| **Wreath** | External → pivot → internal DC compromise, ideal for PNPT | 3-machine network | [tryhackme.com/room/wreath](http://tryhackme.com/room/wreath) |
| **Holo** | Includes AV bypass, large AD network | Large network | [tryhackme.com/room/hololive](http://tryhackme.com/room/hololive) |

### Recommended Order

```bash
[Phase 1] Blue → Kenobi → Basic Pentesting
           ↓
[Phase 2] Alfred → HackPark → Skynet → Relevant → Mr. Robot
           ↓
[Phase 3] Linux PrivEsc Arena → Windows PrivEsc Arena → Steel Mountain
           ↓
[Phase 4] Attacktive Directory  ← write your first report here
           ↓
          Wreath              ← write your second report here
           ↓
          Holo (if time allows)
```

Use Phases 1–3 to build the habit of **taking screenshots + keeping notes** after every machine. Start writing reports from Phase 4 onward.
