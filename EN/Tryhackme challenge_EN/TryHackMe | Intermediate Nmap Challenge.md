# Tryhackme | Intermediate Nmap Challenge

Challenge link → https://tryhackme.com/room/intermediatenmap

I worked through the TryHackMe Intermediate Nmap room. The overall flow involved running recon with Nmap, finding a hint from a custom service, connecting via SSH, and capturing the flag.

First I kicked things off with the most fundamental step — a port scan.

```
sudo nmap -sS --min-rate 5000 -p- <IP>
```

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/2.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

With all three ports confirmed open, here's a quick rundown of what's running on each.

Port 31337 is a non-standard port. In CTF and security labs, it's common to find custom services on unusual ports that serve up hints or special information — so checking the highest port number first seemed like the most efficient approach.

```
nc <IP> 31337
```

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/3.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

![4](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/4.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

I connected using netcat. `nc` (netcat) is a networking tool that establishes TCP/UDP connections and lets you send and receive data — perfect for inspecting raw service responses without a browser.

The custom service turned out to be outputting credentials in plaintext as `user:password`. My gut said this was exactly what I'd need for SSH authentication. CTF labs often drop hints like this intentionally.

Next, I looked at the SSH ports. Two were open — 22 and 2222 — so I needed to figure out which one would accept a login.

First I checked the authentication method on each port.

Port 22 (standard SSH) allowed password authentication.

![5](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/5.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

Port 2222 (alternate SSH), on the other hand, only accepted public key authentication — password login was out.

Since the credentials from port 31337 were a password, port 22 was the obvious choice.

![6](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/6.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

I connected to port 22 first.

```
ssh user@<IP> -p 22
```

![7](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/7.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.jpg)

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
