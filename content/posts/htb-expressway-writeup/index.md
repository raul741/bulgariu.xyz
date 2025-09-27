+++ 
draft = false
date = 2025-09-27T10:14:44+02:00
title = "Guía de Expressway en HackTheBox"
description = "Un tutorial para la máquina HTB \"Expressway\""
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="./files/title.png" alt="">}}

Buenas noches y bienvenidos a otro artículo de HackTheBox, hoy revisaremos la máquina "Expressway" de dificultad fácil.

# Reconocimiento básico

Al intentar escanear la máquina, nos encontraremos con un resultado sorprendentemente vacío, que solo nos muestra un puerto SSH abierto:

{{<betterfigure src="./files/nmap.png" alt="">}}

No hay nada que se pueda hacer contra la versión específica de OpenSSH en nuestro objetivo, sin embargo, mirando el logotipo de la máquina, podemos ver que incluye un "500".

Mientras que el puerto `500/tcp` se muestra claramente cerrado en la máquina, intentemos escanear el puerto `500/udp` en su lugar:

{{<betterfigure src="./files/nmap1.png" alt="">}}

Con el escaneo terminado, descubrimos que la máquina aloja un servicio ikeVPN dentro del puerto UDP abierto.

# Ganando un punto de apoyo

Después de [investigar formas de atacar el servicio](https://book.hacktricks.wiki/en/network-services-pentesting/ipsec-ike-vpn-pentesting.html), podemos usar el siguiente comando para intentar obtener información sobre el servidor VPN:

```bash
sudo ike-scan -M 10.10.11.87
```

{{<betterfigure src="./files/ikescan1.png" alt="">}}

Más importante aún, durante la investigación podemos detectar que el servidor usa autenticación basada en PSK, lo que significa que podríamos intentar obtener un hash del servidor y crackearlo usando el siguiente comando:

```bash
sudo ike-scan -P -M -A -n fakeID 10.10.11.87
```

{{<betterfigure src="./files/ikescan2.png" alt="">}}

Como puede ver, el servidor ha respondido con un hash a pesar de que usamos un valor de grupo inexistente *(fakeID)*, por lo que ahora podemos intentar usar Hashcat junto con este hash:

```bash
hashcat -O -m 5400 hash.txt rockyou.txt
```

{{<betterfigure src="./files/hashcat.png" alt="">}}

Una vez que finalmente tengamos acceso a la contraseña de la VPN, es posible que esté pensando en intentar conectarse al servicio VPN con ella, sin embargo, durante el escaneo es posible que haya visto la línea `ike@expressway.htb`, y si recuerda desde el comienzo de la guía, había un servidor SSH disponible en la máquina:

{{<betterfigure src="./files/user.png" alt="">}}

Usando la contraseña de IKE VPN podemos iniciar sesión en el servidor como el usuario `ike`, lo que nos otorga acceso a la bandera `user.txt`.

# PrivEsc:

Mientras explora la máquina desde adentro, podemos verificar la versión de `sudo` que se ejecuta en el interior, siendo la versión **1.9.17**:

{{<betterfigure src="./files/sudover.png" alt="">}}

Al buscar exploits, descubrirá que esta versión es vulnerable a CVE-2025-32463, por lo que podemos buscar un [Proof-of-Concept](https://github.com/K1tt3h/CVE-2025-32463-POC/) en Internet y ejecutarlo dentro de la máquina:

{{<betterfigure src="./files/root.png" alt="">}}

Después de descargar el exploit en la máquina de destino y ejecutarlo, obtenemos acceso inmediato a `root` y finalmente podemos acceder a la bandera `root.txt`, concluyendo esta máquina.
