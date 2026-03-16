Challenge link: https://tryhackme.com/room/digdug

In this challenge, you can see that the flag is hidden inside a DNS TXT record.

Normally, you would query the TXT record using one of the following commands:

```xml
dig txt <MACHINE IP>
```

or

```xml
dig txt <domain>
```

However, in this challenge, the server only responds to **requests made in a specific way** for the domain [givemetheflag.com](http://givemetheflag.com).

So instead of just entering the IP, you need to specify both the domain and the DNS server (i.e., the machine IP) explicitly in your request.

The exact command is as follows:

```xml
dig txt <domain> @<MACHINE IP>
```

For example:

```xml
dig txt givemetheflag.com @<MACHINE_IP>
```

Running this will let you find the flag in the TXT record included in the response.

![1](https://github.com/jaejun835/hacking-notes/blob/main/Photo/Tryhackme%20challenge_KR/TryHackMe%20%7C%20Dig%20Dug%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.TryHackMe%20%7C%20Dig%20Dug%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)
