+++ 
draft = false
date = 2023-06-11T16:53:52+02:00
title = "Ventoy usage tutorial"
description = "How to install and use Ventoy to create a multi-iso bootable USB"
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial"]
categories = []
externalLink = ""
series = []
+++

A while back in my [Arch Linux installation tutorial](/en/posts/archlinux-install/), I mentioned that I would be eventually taking the time to make a simple guide
to install and use Ventoy, since this is a much simpler process, it won't be as long as the last tutorial, let's get started:

# Downloading
Head up to the [releases section](https://github.com/ventoy/Ventoy/releases) and download the package for your operating system:

{{<betterfigure src="files/ventoydownload.png" alt="Title card">}}

Once downloaded, don't forget to plug in the USB you intend to use for Ventoy.
>Please remember to backup any important files within the USB as installing Ventoy will format and erase all the existing data.

# Installation (Linux)
If you're on Linux, unpack the tar.gz file with `tar -xvf ventoy*.tar.gz` or use your GUI of choice, and open a terminal within the extracted folder, then run `sudo ./VentoyWeb.sh`, you should see something like this:
{{< betterfigure src="files/webscript.png" >}}
Head to the URL it indicates and you'll be able to see the following screen:
{{< betterfigure src="files/ventoyweb.png" >}}
***If you already backed up your files***, then all you have to do now is just press "Install", and that's it! After finishing, 
you should notice two new devices pop up:
{{< betterfigure src="files/devices.png" >}}

Ignore "VTOYEFI", you'll want to copy your .iso files to the "Ventoy" partition, and that should be about it.

# Installation (Windows)
Windows is a tad bit easier, when decompressing the .zip file you'll just want to run "Ventoy2Disk.exe" and you'll get a 
window similar to this one:
{{< betterfigure src="files/windowsvent.png" >}}
Same as last time, ***make sure your USB files are backed up***, hit "Install" and that's it. Once done, drag your .iso files into the "Ventoy" partition, not the "VTOYEFI" one.


# Finishing up
The hardest part now will be booting into the USB itself, which implies [figuring out the button for your computer brand's BIOS](https://www.tomshardware.com/reviews/bios-keys-to-access-your-firmware,5732.html), after you manage to reach the BIOS you'll want
to search for the option to boot off your USB drive, if when booting, you get a security error, you'll want to first look for the 
"Secure Boot" option in your BIOS and turn it off.

>Secure boot just prevents people from booting up any non-authorized Operating Systems off your computer.

Once turned off, give it another go and, if you did add a few .iso files already, you should see a screen similar to this:
{{< betterfigure src="files/ventoy.png" >}}
Now you can just move around with the arrow keys and hit enter to select which .iso file you wish to boot, and believe me, stock up
on .iso files for different situations because ***this will save you from many headaches***.
