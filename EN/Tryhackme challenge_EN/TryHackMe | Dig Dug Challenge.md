Challenge link: https://tryhackme.com/room/digdug

This challenge hides the flag inside a DNS TXT record.

Normally you'd query it with one of these:

```
dig txt <MACHINE IP>
```

or

```
dig txt <domain>
```

However, the twist here is that the server only responds to **requests made in a specific way** for the domain `givemetheflag.com`. That means you can't just throw an IP at it — you need to explicitly point dig at the machine as the DNS server while also specifying the domain.

The correct command:

```
dig txt <domain> @<MACHINE IP>
```

For example:

```
dig txt givemetheflag.com @<MACHINE_IP>
```

Run that and the flag shows up right in the TXT record of the response.  
![1](https://github.com/jaejun835/hacking-notes/blob/main/Photo/Tryhackme%20challenge_KR/TryHackMe%20%7C%20Dig%20Dug%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.TryHackMe%20%7C%20Dig%20Dug%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)
