tryhackme link → https://tryhackme.com/room/intermediatenmap

I worked through TryHackMe's Intermediate Nmap room.

The overall flow was: perform recon with Nmap, find a hint from a custom service, access via SSH, and capture the flag.

First, I ran a port scan — the most fundamental step in information gathering.

```
sudo nmap -sS --min-rate 5000 -p- <IP>
```

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

With all 3 ports confirmed open, I'll briefly explain what service is running on each port.

First, port 31337 is a non-standard port.

In CTF and security practice environments, it's common for custom services to run on non-standard ports like this, providing hints or special information.

So I judged it most efficient to start with the highest port number.

```
nc <IP> 31337
```

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/2.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

I connected using netcat.

nc (netcat) is a networking tool that can create TCP/UDP connections and send/receive data — it's well-suited for checking raw data since it can communicate directly with services without a web browser.

Upon inspection, the custom service was outputting credentials in plaintext in the format user:password.

I had a hunch this was the authentication info needed for SSH access.

In CTF practice, hints are often provided intentionally like this.

Next, I analyzed the SSH ports.

Since both port 22 and port 2222 were open, I needed to figure out which port I could actually connect through.

First, I checked the authentication method for each port.

Port 22 is the standard SSH port and password authentication was available.

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/3.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

![4](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/4.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

On the other hand, port 2222 is an alternate SSH port but only allowed public key authentication, so password authentication was not possible.

Since the credentials obtained from port 31337 were in password form, I needed to attempt connection on port 22.

![5](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/5.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

I connected to port 22 first.

```
ssh user@<IP> -p 22
```

![6](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/6.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

I entered the password obtained from port 31337 and successfully logged in.

After logging in, I checked my current location and found I was in /home/ubuntu, then navigated to the parent directory to explore files.

![7](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/7.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.jpg)

```bash
pwd
cd ..
ls
```

There were two directories: ubuntu and user.

I navigated into the user directory first to check the files.

```bash
cd user
ls
```

I found the flag.txt file.

```css
cat flag.txt
```
