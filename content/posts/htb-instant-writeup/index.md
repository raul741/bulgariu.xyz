+++ 
draft = false
date = 2024-10-20T09:50:54+02:00
title = "Guía de Instant en HackTheBox"
description = "Un tutorial para la máquina HTB \"Instant\""
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="files/title.png" alt="Title card">}}

Saludos, y bienvenidos a mi segunda guía de HackTheBox. Hoy os estaré guiando a través de la máquina "Instant" de HTB.

# Reconocimiento básico

Ejecutar un escaneo básico de Nmap nos mostrará que solo hay dos puertos abiertos, HTTP y SSH, por lo que inmediatamente nos indica que tendremos que atacar el sitio web disponible.

{{<betterfigure src="files/scan.png" alt="Nmap scan">}}

Una vez escaneado, podemos seguir adelante y asociar el nombre de dominio de la máquina a su IP.

{{<betterfigure src="files/hosts.png" alt="Hosts file">}}

# Ganando un punto de apoyo

Lo primero que notarás de la web es que ofrece una aplicación bancaria polivalente en forma de APK, ya que parece un sitio completamente estático, descargaremos el APK.

{{<betterfigure src="files/apk.png" alt="APK Download">}}

## Descompilando el APK

Si bien es posible que tengáis la tentación de intentar instalar el APK en vuestro teléfono móvil, no sacaréis mucho de ello.
Lo que nos gustaría saber es cómo se comunica esta aplicación con el servidor backend, ya que no tenemos nada de información sobre esta máquina más allá de su sitio web estático, y para ello utilizaremos la herramienta ***apktool***.

{{<betterfigure src="files/decomp.png" alt="Decompiled APK">}}

Después de descompilar, veréis que hay una gran cantidad de archivos, aproximadamente alrededor de 8200 diferentes, entonces, ¿cómo podemos tratar de encontrar información útil?
Como dije anteriormente, queremos saber cómo interactúa esta aplicación con el servidor backend de la máquina, por lo que, naturalmente, asumiríamos que el nombre de dominio debe haber sido codificado en algún lugar del código fuente para realizar cualquier llamada a la API.

{{<betterfigure src="files/grepscreen.png" alt="Grepping for domains">}}

Usando Grep, encontramos instantáneamente varias líneas de código que detallan dos subdominios de la máquina: *"swagger-ui"* y *"mywalletv1"*, siendo este último claramente la dirección para realizar llamadas a la API.
Sin embargo, el más interesante es "swagger-ui", ya que este es un programa popular utilizado para crear ***documentación de API***.

{{<betterfigure src="files/swagtee.png" alt="Adding swagger-ui to /etc/hosts">}}

{{<betterfigure src="files/swaggerui.png" alt="swagger-ui dashboard">}}

## ¿Qué diablos es un "JWT"?

Estamos llegando a alguna parte, el sitio de documentación de la API incluso nos permite realizar peticiones de API desde la comodidad de nuestro navegador al permitirnos editar el cuerpo json de cada petición.
No obstante, como podréis ver, no hay mucho que se pueda hacer como usuario no autenticado, lo que requiere que creemos una cuenta e iniciemos sesión, iniciar sesión nos dará un token **"JWT"**.

{{<betterfigure src="files/jwt.png" alt="Login token">}}

> Los tokens JWT son cadenas json codificadas en base64 compuestas por un encabezado, un cuerpo y una cola separados por un ".", con el encabezado detallando los métodos de cifrado, el cuerpo que contiene la información sobre el usuario que inicia sesión y la cola es una firma criptográfica de tanto el encabezado como el cuerpo combinados.

> ***Ya veréis por qué esto es importante más adelante.***

{{<betterfigure src="files/jwtdecoded.png" alt="Login token decoded">}}

Una vez obtenido, finalmente podemos agregar nuestro token de inicio de sesión a la cabecera de autorización dentro del sitio web de documentación de la API y darnos cuenta de que podemos... En realidad, no hacer mucho.
Como te daréis cuenta, todas las llamadas a la API administrativa están prohibidas para los usuarios habituales como nosotros *(y tampoco parece haber ninguna llamada a la API mal configurada/explotable disponible)*, pero como recordaréis, los tokens JWT parecen compartir un encabezado común para manejar los métodos de cifrado.
Entonces, ¿qué pasa si intentamos buscar uno de esos encabezados en el código base APK descompilado?

## El Rabbit R1 acaba de llamar, quiere su llave API hardcodeada de vuelta

{{<betterfigure src="files/jwtadmin.png" alt="Hardcoded admin JWT token">}}

Bingo, si probamos esta clave JWT podremos comprobar que efectivamente funciona y ya tenemos acceso a todas las llamadas administrativas de la API. Pero, ¿cómo podemos usar esto para ganar un punto de apoyo dentro de la máquina?
Es posible que hayais visto dos de las llamadas especiales a la API que giran en torno a los logs. Si intentamos mostrar los logs disponibles veremos lo siguiente:

{{<betterfigure src="files/availablelogs.png" alt="Available logs">}}

Lo primero de lo que probablemente os daréis cuenta es el directorio donde se encuentran los logs, justo dentro de una carpeta de usuario /home llamada "shirohige", y mientras al mismo tiempo nos permite dar información sobre qué archivo nos gustaría consultar, *adivinad cómo podemos explotar esto*.

{{<betterfigure src="files/LFI.png" alt="LFI">}}

Si adivinasteis la inclusión de archivos locales *(o LFI)* a través del parámetro "log_file_name", ¡estabais en lo cierto! Pero como se puede ver, solo podemos consultar los archivos existentes sin poder verificar el contenido de los directorios, por lo que tratar de navegar por el sistema de archivos para encontrar cualquier credencial no va a ser una tarea fácil.
***¿O sí?***

Como mencioné, lo primero que de lo que probablemente os disteis cuenta sobre los comandos de registro disponibles fue el hecho de que estaban ubicados dentro de la carpeta de inicio del usuario, lo que significa que tenemos acceso de lectura al contenido de este directorio. ¿Y qué tipo de credenciales podemos encontrar dentro de la carpeta de inicio de un usuario?***

{{<betterfigure src="files/idrsa.png" alt="Private SSH key">}}

Después de limpiar la llave privada a un formato SSH utilizable, podemos iniciar sesión en el usuario "shirohige" de la máquina y recuperar la bandera user.txt!

{{<betterfigure src="files/usertxt.png" alt="user.txt">}}

# Escalada de privilegios

Una vez dentro, os daréis cuenta de que tampoco hay mucho que podamos hacer como este usuario, ya que al iniciar sesión con la llave SSH, no tenemos acceso a la contraseña necesaria para ejecutar algo como 'sudo -l' por ejemplo.
Después de navegar por el sistema de archivos durante un rato, es posible que os encontréis con el contenido de la carpeta /opt y su archivo "sessions-backup.dat".

{{<betterfigure src="files/opt.png" alt="/opt contents">}}

Después de investigar un poco, aprenderéis que este es un archivo que contiene datos críticos sobre una sesión SSH en el programa "Solar-PuTTY". Después de indagar un poco, podréis encontrar un repositorio como [este](https://github.com/RainbowCache/solar_putty_crack) que puede ayudaros a abrir el archivo de sesión.

Una vez descarguéis la herramienta de descifrado de sesiones Solar-PuTTY de vuestra elección, exfiltrad el archivo "sessions-backup.dat" a través de cualquier medio que consideréis conveniente y suministrad a la herramienta nuestra buena lista de contraseñas rockyou.txt.

{{<betterfigure src="files/cracked.png" alt="Cracked session password">}}

Como podéis ver aquí, tenemos acceso en texto plano a la contraseña de root de la máquina, por lo que ahora podemos volver a nuestra sesión SSH como "shirohige", ejecutar 'su -' y probar la contraseña que encontramos en la sesión de Solar-PuTTY:

{{<betterfigure src="files/roottxt.png" alt="Root flag">}}

¡Y ahí lo tenéis! Habéis completado la máquina HackTheBox "Instant".
