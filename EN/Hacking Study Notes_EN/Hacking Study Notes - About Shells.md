The most fundamental concept when studying hacking is the **"shell"**.

Before diving deep, a quick definition: a shell is what you use to **interact with the command line (CLI)**.

In plain terms — bash on Linux, or cmd / PowerShell on Windows.

When attacking a remote system, the goal is to **obtain one of these shells**.

There are two main techniques:

1. **Reverse Shell** — making the target server **connect back to you**
2. **Bind Shell** — **opening a port on the target** and connecting to it yourself

There are many tools for establishing a reverse or bind shell, but at a minimum you need two things:

- **Shell code (payload)** to be executed on the target system
- An **interface tool** to interact with the resulting shell

Here's a quick overview of the main tools:

**"netcat"**

netcat is a classic networking tool that can perform a wide range of network tasks manually — things like **banner grabbing** during recon, catching **reverse shells**, or connecting to **bind shells**.

However, netcat shells are notoriously unstable, so various techniques are needed to stabilize them.

**"socat"**

socat is best thought of as a straight upgrade over netcat — it does everything netcat does, and more. Shells produced by socat are far more stable. The trade-off is that socat has more complex syntax and isn't installed by default on Linux like netcat is.

**exploit/multi/handler** (Metasploit Framework)

Like socat and netcat, this module catches reverse shells. Since it's part of Metasploit, multi/handler comes fully equipped for obtaining stable shells, with various options to upgrade and improve the shell once obtained.

**Msfvenom**

Also part of Metasploit but usable as a standalone tool. Its main purpose is **generating payloads on the fly** — not just reverse and bind shells, but a wide variety of payload types.

There are other tools as well, but that covers the main ones.
