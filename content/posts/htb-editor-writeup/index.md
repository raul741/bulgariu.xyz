+++ 
draft = false
date = 2025-08-06T17:24:53+02:00
title = "Guía de Editor en HackTheBox"
description = "Un tutorial para la máquina HTB \"Editor\""
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="./files/title.png" alt="">}}

Bienvenidos una vez más, hoy os guiaré a través de la máquina "Editor":

# Reconocimiento básico

Ejecutando un escaneo nmap, nos encontraremos con un conjunto relativamente normal de puertos, 22, 80 y sorprendentemente 8080:

{{<betterfigure src="./files/nmap.png" alt="">}}

Comprobando dentro del puerto HTTP, no hay nada fuera de lo normal, aparte de un enlace de **"Docs"**.
Puede que tengas la tentación de intentar utilizar ingeniería inversa contra los paquetes de editores de texto ofrecidos, pero no te llevarán a ninguna parte.

{{<betterfigure src="./files/1.png" alt="">}}

Dirigiéndonos al enlace **"Docs"**, descubrimos que la página de documentación está ejecutando XWiki en el backend tras el nombre de subdominio ***"xwiki.editor.htb"***, y mirándolo más de cerca, ¡incluso nos dice la versión de XWiki que se está utilizando!

{{<betterfigure src="./files/2.png" alt="">}}

{{<betterfigure src="./files/3.png" alt="">}}

# Gaining a foothold

Buscando exploits relacionados con la versión de XWiki en línea, [encontramos un ***PoC*** para una vulnerabilidad RCE](https://github.com/dollarboysushil/CVE-2025-24893-XWiki-Unauthenticated-RCE-Exploit-POC) relacionada con [CVE-2025-24893](https://www.offsec.com/blog/cve-2025-24893/) que podemos usar contra la máquina.

{{<betterfigure src="./files/4.png" alt="">}}

> Recuerda estabilizar la shell inversa con los siguientes comandos:
> ```bash
> python3 -c "import pty;pty.spawn('/bin/bash')"
> export TERM=xterm
> # Press Ctrl+Z
> stty raw -echo; fg
> ```

Como puedes ver, obtenemos inmediatamente una shell al ejecutar el exploit, sin embargo, solo somos el usuario "xwiki", lo que significa que aún no tenemos acceso a la bandera **user.txt**.

Un buen lugar para empezar a buscar pistas es la base de datos, pero para eso primero necesitamos las credenciales de la base de datos MySQL que ejecuta el servicio XWiki. Y después de un rato buscando, finalmente encontramos un archivo que contiene exactamente estas mismas credenciales en `/etc/xwiki/hibernate.cfg.xml`.

{{<betterfigure src="./files/5.png" alt="">}}

Si tu primer instinto fue intentar comprobar las bases de datos MySQL, probablemente te diste cuenta de que no había nada útil dentro de estas, y eso es porque se trata de un caso de reutilización de credenciales, lo que significa que la contraseña de la base de datos es la misma que la contraseña del usuario.

{{<betterfigure src="./files/6.png" alt="">}}

Como puedes ver, podemos entrar directamente y obtener la bandera **user.txt**.

# PrivEsc

Después de realizar comprobaciones preliminares como `sudo -l`, puede que te des cuenta de que nuestro usuario está dentro del grupo "netdata", y al intentar buscar archivos relacionados con ese grupo encontramos esto:

{{<betterfigure src="./files/7.png" alt="">}}

Investigando un poco en Internet sobre qué es eso de "netdata", descubrimos que es un servicio de métricas y que se aloja regularmente en el **puerto 19999**.

{{<betterfigure src="./files/8.png" alt="">}}

{{<betterfigure src="./files/9.png" alt="">}}

Comprobando la salida de `netstat -tln` podemos ver el puerto exacto visto anteriormente expuesto a localhost, lo que significa que necesitaremos usar SSH forwarding para acceder al servicio usando el siguiente comando:

```bash
ssh -L 19999:127.0.0.1:19999 oliver@editor.htb
```

Inmediatamente después de acceder a `localhost:19999`, nos aparece el servicio y una advertencia muy particular sobre un "nodo obsoleto":

{{<betterfigure src="./files/10.png" alt="">}}

{{<betterfigure src="./files/11.png" alt="">}}

Mirando la alerta, parece que esta versión de netdata está desactualizada y es potencialmente vulnerable, así que empezamos a buscar exploits y encontramos [este exploit](https://github.com/AzureADTrent/CVE-2024-32019-POC) originado en [CVE-2024-32019](https://securityvulnerability.io/vulnerability/CVE-2024-32019).

Tras seguir las instrucciones del repositorio, escalamos con éxito nuestros privilegios y obtenemos la bandera **root.txt**!

{{<betterfigure src="./files/12.png" alt="">}}

