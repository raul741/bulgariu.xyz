+++ 
draft = false
date = 2023-06-11T16:53:52+02:00
title = "Como utilizar Ventoy"
description = "Como instalar y usar Ventoy para crear un USB bootable multi-iso"
slug = ""
authors = ["Raul Bulgariu SUciu"]
tags = ["Tutorial"]
categories = []
externalLink = ""
series = []
+++

Hace tiempo en mi [tutorial de instalación de Arch Linux](/posts/archlinux-install/), mencioné que eventualmente crearía una guía simple de como instalar y usar Ventoy, al ser un proceso mucho más simple, no tardará tanto como el anterior tutorial, vamos a empezar:

# Descargando
Dirígete a la [sección de descargas](https://github.com/ventoy/Ventoy/releases) y descarga el archivo empaquetado para tu sistema
operativo:

{{< betterfigure src="files/ventoydownload.png" >}}

Una vez descargado, no te olvides de conectar la unidad USB que desees utilizar para Ventoy.
>Por favor, recuerda respaldar cualquier archivo importante de la unidad USB ya que instalar Ventoy formateará y borrará todos los datos de este.

# Instalación (Linux)
Si estás utilizando Linux, desempaqueta el archivo tar.gz con `tar -xvf ventoy*.tar.gz` o utiliza de aplicación gráfica preferida,  abre una terminal dentro de la carpeta extraída, y ejecuta `sudo ./VentoyWeb.sh`, deberías ver algo parecido a esto:
{{< betterfigure src="files/webscript.png" >}}
Accede a la URL que indica y podrás ver la siguiente pantalla:
{{< betterfigure src="files/ventoyweb.png" >}}
***Si ya has respaldado tus archivos***, lo único que tienes que hacer es presionar "Install", y ya está! Al terminar, 
deberías ver aparecer dos nuevas unidades de almacenamiento:
{{< betterfigure src="files/devices.png" >}}

Ignora a "VTOYEFI", querrás copiar tus archivos .iso en la partición "Ventoy" y eso debería ser todo.

# Instalación (Windows)
Windows es más fácil, al descomprimir el archivo .zip, solo necesitas ejecutar "Ventoy2Disk.exe" y verás una ventana parecida a esta:
{{< betterfigure src="files/windowsvent.png" >}}
Lo mismo que la última vez, ***asegurate de que tus archivos están respaldados***, presiona "Install" y ya está. Una vez terminado, arrastra tus archivos .iso a la partición "Ventoy", no a "VTOYEFI".


# Terminando
La parte más difícil ahora es iniciar el USB como sistema operativo, lo que implica [descubrir que botón de BIOS utiliza tu marca de ordenador](https://www.tomshardware.com/reviews/bios-keys-to-access-your-firmware,5732.html), después de llegar a la BIOS, querrás buscar la opción para iniciar tu unidad USB conectada, si al iniciar, recibes un error de seguridad, deberías buscar la opción de "Inicio seguro" o "Secure Boot" y desactivarla.

>Secure boot simplemente previene que otras personas puedan iniciar cualquier sistema operativo no autorizado en tu ordenador.

Una vez apagado, inténtalo otra vez y, si ya añadistes algunos archivos .iso, deberías ver una pantalla parecida a esta:
{{< betterfigure src="files/ventoy.png" >}}
Ahora te puedes mover utilizando las teclas de flecha y pulsar enter para seleccionar que archivo .iso deseas iniciar, y créeme, será mejor que te abastezcas de sistemas operativos para diferentes situaciones, porque ***esto te ahorrará muchos dolores de cabeza***.
