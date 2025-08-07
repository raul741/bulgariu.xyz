+++ 
draft = false
date = 2025-08-06T17:24:53+02:00
title = "HackTheBox Editor Writeup"
description = "A walkthrough for the HTB \"Editor\" machine"
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="./files/title.png" alt="Title card">}}

Welcome once again, today I'll be walking you through the "Editor" machine:

# Basic reconnaissance

Running an nmap scan, we'll encounter a relatively normal set of ports, 22, 80 and surprisingly 8080:

{{<betterfigure src="./files/nmap.png" alt="">}}

Checking inside the HTTP port, there's nothing too out of the ordinary apart from a **"Docs"** link.
You might be tempted to try reverse engineering the offered text editor packages but it won't take you anywhere.

{{<betterfigure src="./files/1.png" alt="">}}

Heading over to the **"Docs"** link, we find out that the documentation page is running XWiki on the backend behind the subdomain name ***"xwiki.editor.htb"***, and looking closer, it's even telling us the version of XWiki being used!

{{<betterfigure src="./files/2.png" alt="">}}

{{<betterfigure src="./files/3.png" alt="">}}

# Gaining a foothold

Looking for exploits related to the XWiki version online, we quickly [find a ***PoC*** for an RCE vulnerability](https://github.com/dollarboysushil/CVE-2025-24893-XWiki-Unauthenticated-RCE-Exploit-POC) related to [CVE-2025-24893](https://www.offsec.com/blog/cve-2025-24893/) that we can use against the machine.

{{<betterfigure src="./files/4.png" alt="">}}

> Remember to stabilize your reverse shell with the following commands:
> ```bash
> python3 -c "import pty;pty.spawn('/bin/bash')"
> export TERM=xterm
> # Press Ctrl+Z
> stty raw -echo; fg
> ```

As you can see, we immediately obtain a shell upon executing the exploit however, we're only the "xwiki" user, meaning we do not have access to the **user.txt** flag yet.

A good place to begin looking for clues is the database, but for that we first need the credentials for the MySQL database executing the XWiki service. And after some investigation, we eventually find a file containing these exact same credentials at `/etc/xwiki/hibernate.cfg.xml`.

{{<betterfigure src="./files/5.png" alt="">}}

If your first instinct was to try checking the MySQL databases, you probably realized that you couldn't quite find anything useful, and that's because this is a case of re-used credentials, meaning the database password is the same one as the user password.

{{<betterfigure src="./files/6.png" alt="">}}

As you can see, we can log right in and obtain the **user.txt** flag.

# PrivEsc

After performing any preliminary checks such as `sudo -l` you might notice that our user is inside the group "netdata", and checking for files related to that group yields this:

{{<betterfigure src="./files/7.png" alt="">}}

Doing some research on the Internet on what this "netdata" is, we find that it's a metrics service and is hosted regularly on **port 19999**.

{{<betterfigure src="./files/8.png" alt="">}}

{{<betterfigure src="./files/9.png" alt="">}}

Checking the output of `netstat -tln` we can see the exact port seen earlier exposed to localhost, meaning we'll need to use SSH forwarding to access the service using the following command:

```bash
ssh -L 19999:127.0.0.1:19999 oliver@editor.htb
```

Immediately after accessing `localhost:19999`, we're greeted with the service and a very particular warning about an outdated node:

{{<betterfigure src="./files/10.png" alt="">}}

{{<betterfigure src="./files/11.png" alt="">}}

Looking into the alert, it seems like this version of netdata is outdated and potentially vulnerable, so we start looking for exploits and find [this exploit](https://github.com/AzureADTrent/CVE-2024-32019-POC) originating from [CVE-2024-32019](https://securityvulnerability.io/vulnerability/CVE-2024-32019).

After following the repository's instructions, we successfully escalate our privileges and obtain the **root.txt** flag!

{{<betterfigure src="./files/12.png" alt="">}}

