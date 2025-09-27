+++ 
draft = false
date = 2025-09-27T10:14:44+02:00
title = "HackTheBox Expressway Writeup"
description = "A walkthrough for the HTB \"Expressway\" machine"
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="./files/title.png" alt="">}}

Good evening and welcome to another HackTheBox writeup, today we'll be reviewing the easy difficulty "Expressway" machine.

# Basic reconnaissance

Trying to scan the machine, we'll be met with a surprisingly empty result, yielding only an SSH port:

{{<betterfigure src="./files/nmap.png" alt="">}}

There's nothing that can be done against the specific OpenSSH version in our target, however, looking at the logo of machine, we can see it include a "500".

While port `500/tcp` is clearly shown to be closed in the machine, let's try scanning the port `500/udp` instead:

{{<betterfigure src="./files/nmap1.png" alt="">}}

With the scan finished, we find out that the machine is hosting an ikeVPN service within the open UDP port.

# Gaining a foothold

After [investigating ways to attack the service](https://book.hacktricks.wiki/en/network-services-pentesting/ipsec-ike-vpn-pentesting.html), we can use the following command to try and obtain information about the VPN server:

```bash
sudo ike-scan -M 10.10.11.87
```

{{<betterfigure src="./files/ikescan1.png" alt="">}}

More importantly, during the probe we can spot that the server uses PSK-based authentication, meaning that we could attempt to obtain a server hash and brute-force it using the following command:

```bash
sudo ike-scan -P -M -A -n fakeID 10.10.11.87
```

{{<betterfigure src="./files/ikescan2.png" alt="">}}

As you can see, the server has replied with a hash despite us using a non-existing group value *(fakeID)*, so we can now attempt to use Hashcat alongside this hash:

```bash
hashcat -O -m 5400 hash.txt rockyou.txt
```

{{<betterfigure src="./files/hashcat.png" alt="">}}

Once we finally have access to the VPN password, you might be thinking of trying to connect to the VPN service with it, however, during the scan you may have noticed the line `ike@expressway.htb`, and if you remember from the start of the guide, there was an SSH server available on the machine:

{{<betterfigure src="./files/user.png" alt="">}}

Using the IKE VPN password we can log in to the server as the `ike` user, granting us access to the `user.txt` flag.

# PrivEsc:

While exploring the machine from within, you might check the version of `sudo` running inside, it being version **1.9.17**:

{{<betterfigure src="./files/sudover.png" alt="">}}

When searching for exploits, you'll find out that this version is vulnerable to CVE-2025-32463, so we can just search for a [Proof-of-Concept](https://github.com/K1tt3h/CVE-2025-32463-POC/) on the internet and execute it inside the machine:

{{<betterfigure src="./files/root.png" alt="">}}

After downloading the exploit to the target machine and executing it, we gain immediate access to `root` and can finally access the `root.txt` flag, concluding this machine.
