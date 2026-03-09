Challenge link → https://tryhackme.com/room/intermediatenmap

I worked through the TryHackMe Intermediate Nmap room. The overall flow involved running recon with Nmap, finding a hint from a custom service, connecting via SSH, and capturing the flag.

First I kicked things off with the most fundamental step — a port scan.

```
sudo nmap -sS --min-rate 5000 -p- <IP>
```

With all three ports confirmed open, here's a quick rundown of what's running on each.

Port 31337 is a non-standard port. In CTF and security labs, it's common to find custom services on unusual ports that serve up hints or special information — so checking the highest port number first seemed like the most efficient approach.

```
nc <IP> 31337
```

I connected using netcat. `nc` (netcat) is a networking tool that establishes TCP/UDP connections and lets you send and receive data — perfect for inspecting raw service responses without a browser.

The custom service turned out to be outputting credentials in plaintext as `user:password`. My gut said this was exactly what I'd need for SSH authentication. CTF labs often drop hints like this intentionally.

Next, I looked at the SSH ports. Two were open — 22 and 2222 — so I needed to figure out which one would accept a login.

Port 22 (standard SSH) allowed password authentication.

Port 2222 (alternate SSH) only accepted public key auth — password login was out.

Since the credentials from port 31337 were a password, port 22 was the obvious choice.

```
ssh user@<IP> -p 22
```

I entered the password from port 31337 and got in without a hitch.

After logging in, I checked my location — `/home/ubuntu`. I moved up a level and browsed around.

```bash
pwd
cd ..
ls
```

Two directories: `ubuntu` and `user`. Headed into `user`.

```bash
cd user
ls
```

Found `flag.txt`.

```bash
cat flag.txt
```
