+++ 
draft = false
date = 2024-11-04T10:57:43+01:00
title = "Guía de MonitorsThree en HackTheBox"
description = "Un tutorial para la máquina HTB \"MonitorsThree\""
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="files/title.png" alt="Title card">}}

Hola de nuevo, hoy nos ocuparemos de la máquina HackTheBox "MonitorsThree", vamos a ello, ¿de acuerdo?

# Reconocimiento básico

Una vez más, después de escanear la máquina, nos encontraremos con los dos puertos HTTP y SSH clásicos, junto con un puerto websnp al que no podemos acceder.

{{<betterfigure src="files/scan.png" alt="Nmap scan">}}

Dado que no tenemos ningún otro vector de ataque más allá del servidor web, agreguemos como siempre el nombre de dominio a nuestro archivo /etc/hosts.

{{<betterfigure src="files/hosts.png" alt="Hosts file">}}

# Ganando un punto de apoyo

Una vez dentro, podemos encontrar otra web corporativa genérica, aunque esta tiene una función de inicio de sesión que podríamos intentar explotar, aunque actualmente no tenemos credenciales a nuestro nombre que podamos probar, así que centrémonos en algo diferente por ahora.

{{<betterfigure src="files/web.png" alt="Website">}}

Al enumerar subdominios, obtenemos un solo resultado para un subdominio llamado "cacti", por lo que podemos seguir adelante y también agregarlo a nuestro archivo hosts.

> Al hacer fuzzing de subdominios con ffuf, recordad filtrar los falsos positivos comprobando el tamaño de respuesta de la mayoría de los subdominios y filtrándolos con `-fs 13560` como en este caso.

{{<betterfigure src="files/subscan.png" alt="Website">}}

> Totalmente no relacionado, pero por favor, siempre que utiliceis cualquier herramienta de fuzzing por fuerza bruta como ffuf, recordad usar un parámetro de limitación de velocidad como `-rate 100` para evitar sobrecargar la máquina, especialmente las máquinas compartidas.

{{<betterfigure src="files/cactiweb.png" alt="Cacti website">}}

Una vez que añadimos el subdominio, podemos encontrar un servidor para un programa llamado "Cacti", por suerte para nosotros hicieron todo lo posible para decirnos la versión que se está utilizando actualmente, [así que después de 5 segundos de búsqueda encontramos un exploit RCE para la versión vulnerable!](https://github.com/5ma1l/CVE-2024-25641)
***Sin embargo, un inconveniente***, el exploit requiere autenticación con el servidor de destino, y todavía no tenemos ninguna credencial que podamos probar.

Probablemente sea un buen momento para revisar ese formulario de inicio de sesión de la página inicial:

{{<betterfigure src="files/loginform.png" alt="Login form">}}

Teniendo en cuenta el hecho de que este sitio web dinámico parece estar autoprogramado en lugar de usar un CMS regular como Drupal/Wordpress, podemos suponer que puede haber llamadas SQL no desinfectadas en el formulario de inicio de sesión, por lo que enviamos una solicitud POST y la capturamos con Burpsuite.

{{<betterfigure src="files/loginformtest.png" alt="Login form injection attempt">}}

> Si bien podríamos intentar usar SQLMap para escanear automáticamente el formulario de inicio de sesión, no es confiable a menos que se proporcione una gran cantidad de contexto, por lo que es mejor probar manualmente las inyecciones SQL agregando caracteres especiales como `'` al final de un parámetro y ver cómo reacciona el servidor.

Sin embargo, en este caso parece que el sitio web no produjo ningún comportamiento anormal, por lo que el formulario de inicio de sesión parece ser mayormente seguro, aunque todavía queda un lugar que no hemos probado.

# ¿El formulario de contraseña olvidada? ¿En serio?

{{<betterfigure src="files/forgotten.png" alt="Forgotten password form">}}

Si bien el formulario de inicio de sesión puede parecer la opción más obvia, todos y cada uno de los ataques están fallando claramente, así que sigamos adelante y probemos el formulario de restablecimiento de contraseña.

{{<betterfigure src="files/forgottentest.png" alt="Password reset injection attempt">}}

Estamos llegando a alguna parte, el parámetro `username` para la solicitud de restablecimiento de contraseña es claramente vulnerable a inyección SQL, y no solo eso, el servidor nos acaba de decir el nombre del sistema de base de datos backend, "MariaDB".

## ¡Hora de encontrar algunas credenciales!

Como mencioné antes, SQLMap es en su mayoría poco confiable sin el contexto para eliminar una gran cantidad de escaneo innecesario, sin embargo, ahora tenemos el contexto que estábamos buscando:

{{<betterfigure src="files/requestfile.png" alt="Request file">}}

Ahora, simplemente agregaremos la solicitud POST original a un archivo de texto y lo leeremos con SQLMap *(`-r reset_password.txt`)*, definiremos `username` como el parámetro específico para escanear *(`-p username`)* y le diremos a SQLMap que solo ejecute consultas MariaDB, ya que el servidor web había sido tan amable con nosotros anteriormente como para decirnos el nombre del backend de la base de datos *(`--dbms=mariadb`)*.

> Los parámetros `--level 5` y `--risk 3` se utilizan para indicar a SQLMap que sea tan ruidoso como necesite ser, en la mayoría de los escenarios del mundo real, un análisis tan ruidoso como ese resultaría en vuestra dirección IP siendo bloqueada automáticamente.
> En cuanto al parámetro `--tables`, con él, la herramienta nos mostrará automáticamente todas las tablas disponibles en la base de datos si consigue explotarla adecuadamente.

{{<betterfigure src="files/tablesfound.png" alt="The tables">}}

Como podéis ver, SQLMap logró explotar con éxito la vulnerabilidad de inyección SQL y nos mostró las tablas disponibles en la base de datos, siendo la más interesante de todas la tabla "users", ¡así que sigamos adelante y volquemos su contenido!

{{<betterfigure src="files/usertable.png" alt="The user table">}}

> Tened en cuenta que algunas de estas capturas de pantalla se editan para reducir la cantidad de salida de texto en los comandos SQLMap que se utilizan por el bien del tutorial.

Como podéis ver, acabamos de encontrar las contraseñas hasheadas de varios usuarios en la base de datos, y si intentamos analizarlas en un sitio web como [este](https://hashes.com/en/tools/hash_identifier) descubriremos que están utilizando el algoritmo extremadamente inseguro MD5.

# Descifrando un hash

Naturalmente, nuestro primer instinto será intentar descifrar el hash MD5 de la cuenta de administrador usando John the Ripper, y para eso usaremos la clásica lista de palabras "rockyou.txt".

{{<betterfigure src="files/crackedhash.png" alt="The cracked hash">}}

Como era de esperar, obtendremos acceso inmediato a la contraseña de administrador. Podríamos intentar iniciar sesión en la página principal, pero ¿recordáis ese subdominio explotable para el que necesitábamos credenciales?

{{<betterfigure src="files/cactilogin.png" alt="Cacti web page">}}

¡Y bingo! Estas credenciales son válidas para el subdominio explotable, lo que significa que ahora podemos intentar usar el exploit que habíamos encontrado hace tanto tiempo.

# Alcanzando la bandera de usuario

{{<betterfigure src="files/revshell.png" alt="Exploit">}}

> Recordad modificar el archivo `./php/monkey.php` con vuestra propia IP y puerto de escucha antes de ejecutar el exploit.

Como podéis ver, la shell inversa funcionó y ahora tenemos acceso remoto a la máquina. Ahora podemos comenzar a explorar la máquina en busca de posibles vectores de ataque, preferiblemente aquellos que nos permitan acceder al usuario "marcus" dentro de la carpeta /home.

Un lugar común para buscar vectores suele ser en la raíz del servidor web, ya que generalmente es posible encontrar cosas como archivos de configuración con contraseñas de bases de datos, y como podéis ver aquí, ¡logramos encontrar un archivo que contiene exactamente eso!

{{<betterfigure src="files/dbconfig.png" alt="DB Configuration">}}

Inmediatamente nos ponemos a trabajar en el acceso a la base de datos, ya que esta parece ser completamente diferente a la que logramos explotar anteriormente.

{{<betterfigure src="files/accessdb.png" alt="Accessing the DB">}}

La primera tabla que nos llama la atención es "user_auth", y como podréis ver, encontramos las credenciales de los usuarios de Cacti, incluyendo dos credenciales pertenecientes a una misma persona tanto para la cuenta de administrador que utilizamos para ejecutar el exploit como para una cuenta personal para el usuario "marcus", ¡justo el usuario al que intentábamos acceder anteriormente!
> Por cierto, si aún no lo habíais notado, no solo está deshabilitada la autenticación basada en contraseña en el servidor SSH, sino que también las credenciales que encontramos por primera vez para ejecutar el exploit no funcionan cuando intentamos meternos por `su -` a la cuenta "marcus", aunque no os preocupéis, ***corregiremos eso pronto***.

{{<betterfigure src="files/userauth.png" alt="The Cacti user credentials">}}

Ahora podemos volver a usar John the Ripper con la lista de palabras "rockyou.txt" para descifrarla, pero no sin antes comprobar qué algoritmo de hash están utilizando las contraseñas, y [después de consultar de nuevo el sitio web](https://hashes.com/en/tools/hash_identifier), descubrimos que está usando bcrypt, así que ajustaremos a John para ello:

{{<betterfigure src="files/crackedmarcus.png" alt="Marcus' password">}}

Con el hash ahora descifrado, podemos seguir adelante e intentar meternos por `su -` a la cuenta de marcus.

{{<betterfigure src="files/su.png" alt="Access granted">}}

¡Éxito! ¡Con esto logramos alcanzar la bandera de usuario de MonitorsThree!

# Escalando nuestros privilegios

Antes de intentar nada, probablemente deberíamos encontrar alguna manera de acceder correctamente al usuario marcus sin tener que depender de una shell inversa endeble, ya que la máquina solo permite la autenticación basada en llaves públicas, tiene sentido que busquemos claves SSH privadas dentro del directorio de inicio del usuario:

{{<betterfigure src="files/id_rsa.png" alt="Access granted">}}

> ***Consejo:***
> Podéis usar netcat para exfiltrar fácilmente archivos de una máquina Linux sin abrir ningún puerto que pueda confundir a otros jugadores.

Ahí lo tenéis, una shell infinitamente más confiable, de todos modos, a medida que buscamos vectores de escalada de privilegios, es posible que os encontréis con los puertos actualmente abiertos en el servidor, con un servidor en el puerto 8200 de la interfaz loopback que destaca en particular.

{{<betterfigure src="files/netstat.png" alt="Netstat">}}

Dado que tenemos un acceso SSH más adecuado, ahora podemos usar SSH forwarding para acceder al servicio oculto desde nuestro navegador de la siguiente manera.

{{<betterfigure src="files/forwarding.png" alt="Do people even read these?">}}

{{<betterfigure src="files/duplicati.png" alt="Duplicati login page">}}

Para aquellos que no lo saben, Duplicati es un software popular para realizar copias de seguridad, pero para nosotros parece un camino hacia la bandera root. Sin embargo, ninguna de las credenciales que encontramos anteriormente funcionarán, así que veamos si podemos encontrar algún archivo de configuración útil que nos permita entrar:

{{<betterfigure src="files/sqlite.png" alt="Found sqlite file">}}

Es probable que os hayáis encontrado con el contenido de la carpeta /opt, y también es posible que tengáis la tentación de ir directamente a la carpeta de copias de seguridad, pero rápidamente os daréis cuenta de que solo se tratan de copias de seguridad de los archivos del servicio Cacti que ya hemos comprometido por completo.
Lo que *nos interesa* es el archivo de configuración de Duplicati, ya que como podéis ver, encontramos una variable `server-passphrase` y una variable `server-passphrase-salt`, sin embargo, también os daréis cuenta rápidamente de que ninguno de estos valores funciona realmente cuando intentamos iniciar sesión en la página de Duplicati.

***Entonces, ¿qué hacemos ahora?***

## El exploit más enrevesado conocido por el hombre

Naturalmente, el mejor curso de acción sería buscar formas de eludir el proceso de autenticación de Duplicati utilizando solo su `server-passphrase`, y eso os proporcionará inmediatamente un enlace a [esta issue de Github en el repositorio de Duplicati](https://github.com/duplicati/duplicati/issues/5197).

Básicamente, estas dos variables (saltedpwd y noncedpwd) que podéis encontrar aquí al inspeccionar el Javascript de la página de inicio de sesión dictan el hash de la combinación del token "nonce" generado para cada solicitud de inicio de sesión y la contraseña ingresada más la salt que encontramos anteriormente en el archivo de configuración.

> Un "nonce" es un token arbitrario que se puede usar solo una vez en una comunicación criptográfica para evitar ataques de replay.

{{<betterfigure src="files/inspect.png" alt="Javascript variables">}}

La forma en que se supone que debemos explotarlo es convirtiendo la variable `server-passphrase` que encontramos anteriormente de Base64 a Hexadecimal con una herramienta como [CyberChef](https://cyberchef.org/) y luego configurando manualmente la variable `saltedpwd` en el valor resultante.

Después de esto, volvemos a encender Burpsuite y le decimos que intercepte una solicitud de inicio de sesión, pero en lugar de enviarla inmediatamente al repetidor, le decimos que también intercepte la respuesta a la solicitud de la siguiente manera:

{{<betterfigure src="files/intercept.png" alt="Intercept the response">}}

Ahora podemos reenviar la solicitud, solo que esta vez también podremos interceptar la respuesta y verificar su contenido:

{{<betterfigure src="files/nonce.png" alt="The nonce">}}

Como podéis ver aquí, no solo recibimos el token nonce de un solo uso, sino que también vemos la misma passphrase de salt que encontramos en el archivo de configuración .sqlite para Duplicati.

Ahora, mientras la página aún siga congelada por nuestra solicitud interceptada, se supone que debemos abrir la consola y pegar el siguiente código *(y también recordar modificarlo por favor)*:
```js
var saltedpwd = 'HexOutputFromCyberChef';
var noncedpwd = CryptoJS.SHA256(CryptoJS.enc.Hex.parse(CryptoJS.enc.Base64.parse('NonceFromBurp') + saltedpwd)).toString(CryptoJS.enc.Base64);
console.log(noncedpwd);
```

A continuación, hay que reemplazar el contenido de `saltedpwd`  por la variable hexadecimal `server-passphrase` que convertimos anteriormente y `NonceFromBurp` por el valor Nonce que recibimos después de interceptar la respuesta del servidor.

Una vez editado correctamente, podemos ejecutar los comandos copiados y `console.log(noncedpwd)` imprimirá el hash calculado.

{{<betterfigure src="files/javascript.png" alt="Javascript variables">}}

Ahora se supone que debemos reenviar la solicitud que interceptamos y luego interceptaremos nuestra propia solicitud POST que incluye la contraseña hash que la página generó para nosotros, ahora tenemos que debemos reemplazar el contenido del parámetro `password` con el hash que `console.log(noncedpwd)` imprimió anteriormente:

{{<betterfigure src="files/modifiedpass.png" alt="Modified password">}}

> Recordad usar `Ctrl+U` en Burpsuite para codificar en URL la nueva contraseña con hash

Ahora, si reenviáis esta solicitud, veréis que la página os permite iniciar sesión independientemente de la contraseña que se haya utilizado.

{{<betterfigure src="files/duplicatipage.png" alt="Duplicati main webpage">}}

# Capturando la bandera root

Una vez aquí, podremos ver que Duplicati ya tiene configurada una configuración de copia de seguridad para los archivos del servidor Cacti como habíamos hablado antes, pero lo que nos gustaría es alguna forma de acceder a la bandera root.txt en la máquina.

Una forma de intentarlo es editando la configuración de copia de seguridad preexistente.

{{<betterfigure src="files/sources.png" alt="Duplicati sources">}}

> Algunos hackers perspicaces podrían haberse dado cuenta de que Duplicati se estaba ejecutando en un contenedor Docker, lo que explica la carpeta `/source` que precede a cualquier ruta de archivo dentro de la máquina de destino.

Ahora podemos seguir adelante y agregar el archivo root.txt dentro de `/sources/root` a la lista de archivos de los que se está haciendo una copia de seguridad.

{{<betterfigure src="files/sourcesroot.png" alt="Duplicati sources">}}

Después de guardar la nueva configuración, podemos ejecutar manualmente la configuración de la copia de seguridad desde la página principal y dirigirnos a la sección "Restore", donde podemos elegir restaurar individualmente el archivo root.txt que nuestra copia de seguridad manual logró copiar:

{{<betterfigure src="files/restore.png" alt="Duplicati restoration">}}

Ahora podemos elegir dónde almacenaremos la bandera, por ahora vamos a crear una carpeta temporal en `/sources/tmp/` con permisos de escritura completos para todos (`chmod 777 /tmp/.topsecret`):

{{<betterfigure src="files/restoreto.png" alt="Duplicati restoration">}}

Y una vez ejecutado el proceso de restauración, podemos dirigirnos a nuestra carpeta temporal y podremos encontrar el archivo root.txt esperándonos, marcando el fin de la máquina "MonitorsThree".

{{<betterfigure src="files/rootflag.png" alt="Root flag">}}
