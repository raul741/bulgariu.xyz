+++ 
draft = false
date = 2023-05-21T12:32:26+02:00
title = "Instalación UEFI encriptada de Arch Linux"
description = "Una simple guía de instalación de Arch Linux"
slug = ""
authors = ["Raul Bulgariu Suciu"]
tags = ["Linux", "Tutorial"]
categories = []
externalLink = ""
series = []
+++

Esta es una guía con la intención de ayudarte a instalar Arch Linux en un ordenador UEFI con encripción de disco completo, si tus necesidades se diferencian de alguna manera, puedes consultar la guía de instalación original [aquí](https://wiki.archlinux.org/title/Installation_guide). Vamos a empezar:

# Aviso importante
Por favor, lea cada paso con cuidado y si es posible, intente primero realizar la instalación dentro de una [máquina virtual](https://www.virtualbox.org/), no sé el alcance total de los posibles errores que esta guía podría tener, así que si detecta alguno, no dudes en [avisarme!](/contact).

# Descargando el .iso
1. Descarga el último archivo .iso de la [página de Arch Linux](https://archlinux.org/download/)

# Crea una unidad USB booteable
1. Descarga y utiliza [Ventoy](https://www.ventoy.net/en/download.html) para crear una unidad USB multi-booteable
2. Arrastra el .iso de Arch Linux a la partición "Ventoy" creada en tu USB
3. Descubre el [botón de la BIOS](https://www.tomshardware.com/reviews/bios-keys-to-access-your-firmware,5732.html) de tu marca de ordenador y utilizalo para iniciar como sistema operativo tu unidad USB
4. Una vez que llegues a la pantalla de inicio, solo selecciona las opciones por defecto
> Puede que haga un tutorial de Ventoy en el futuro, estad atentos.

# Configurar la distribución de teclado
1. Descubre el código para tu distribución de teclado, los códigos pueden ser vistos con `ls /usr/share/kbd/keymaps`
2. Ejecuta `loadkeys KB` reemplazando "KB" con el código para tu distribución correcta

> Por ejemplo, en mi caso ejecutaría `loadkeys es` para utilizar la distribución de teclado española.

# Comprobar la compatibilidad UEFI
1. Una vez funciona bien nuestro teclado, podemos seguir adelante y verificar en un segundo que tenemos soporte UEFI ejecutando ```ls /sys/firmware/efi/efivars```
2. Si el comando de arriva no ha mostrado una gran cantidad de archivos, tu ordenador no tiene UEFI activada, si después de revisar la configuración de la BIOS no tienes soporte de UEFI, consulta [la guía de instalación oficial](https://wiki.archlinux.org/title/Installation_guide), ya que una instalación de Arch Linux en BIOS supera el alcance de esta guía

# Establecer la conexión a Internet
> Si estás conectado por cable deberías tener conexión sin necesidad de configurar nada.
1. Si deseas utilizar Wi-Fi para la instalación, primero ejecuta `iwctl`
2. Después, ejecuta `station list` para descubrir el nombre de tu interfaz de red inalámbrica
3. Ahora ejecuta: `station INTERFAZ_INALAMBRICA	connect SSID` y reemplaza INTERFAZ_INALAMBRICA con tu interfaz *(por ejemplo, mi interfaz se llama `wlan0`)* y SSID con el nombre de tu red Wi-Fi
4. Una vez conectado, pulsa **Ctrl+D** para salir de iwctl

# Particionado de disco
> Por favor, recuerda que los nombres de disco son ***usualmente*** diferentes en cada ordenador, te pido que ejecutes `lsblk` y revises por tí mismo cuál es el disco que deseas particionar, especialmente si tienes un ordenador con múltiples discos duros. ***Esto te ayudará a no borrar el disco equivocado***.

> Por cierto, si estás utilizando un disco NVMe, los nombres cambiarán de sda, sdb, etc... a algo parecido a **"nvme0n1"**, `lsblk` también te ayudará aquí.

1. Ahora estamos entrando en la zona de peligro, por favor, lee el texto de arriba y asegurate de que sabes el nombre de disco correcto, a partir de ahora utilizaré /dev/sdX para referirme al disco de instalación, ***reemplaza la X con la letra correcta de tu disco***
2. Una vez estás seguro, ejecuta `cfdisk /dev/sdX`, desde aquí elige la tabla de partición GPT si cfdisk te pregunta, y en caso contrario, borraremos las particiones no deseadas.
3. Con el nuevo espacio libre, selecciona "New" y crea una partición de 1G **(Partición de inicio)**, y cambia su "Type" o tipo a ***"EFI System"***
4. Ahora selecciona el espacio libre restante, pulsa "New" otra vez y crea una nueva partición con todo el espacio restante **(Partición root)**
5. Cuando acabes, pulsa "Write" y entonces "Quit"

# Encriptando la partición root
> Si todo fue correctamente, ejecutar `lsblk` debería mostrar las nuevas particiones creadas.

1. Ahora podemos encriptar nuestro sistema ejecutando `cryptsetup luksFormat /dev/sdX2`, la herramienta te pedirá una contraseña para iniciar tu sistema operativo, por favor, ***no te olvides de esta contraseña***

> Recuerda que el comando de arriba se ejecuta sobre la partición root, no sobre el disco entero.

2. Ejecuta `cryptsetup open /dev/sdX2 crypt` para abrir tu partición encriptada

# Creando los sistemas de archivos
1. Crea el sistema de archivos para tu partición de inicio ejecutando `mkfs.vfat -F32 /dev/sdX1`
2. Crea el sistema de archivos para tu partición root ejecutando `mkfs.ext4 /dev/mapper/crypt`

# Montando los sistemas de archivos
1. Ejecuta `mount /dev/mapper/crypt /mnt` para montar tu sistema de archivos root
2. Ejecuta `mount --mkdir /dev/sdX1 /mnt/boot` para montar tu sistema de archivos de inicio
3. Ejecuta `lsblk` para asegurarte de que todo fue bien

# Crea el archivo swap
> El archivo swap es memoria de disco que será utilizada en caso de que no haya suficiente RAM, si te saltas este paso tu sistema se congelará cada vez que utilize demasiada memoria RAM.

> La cantidad de swap que necesitas dependerá de tus necesidades, si no tienes intención de configurar hibernación entonces puedes utilizar mucho menos swap, personalmente utilizo 2GB de swap con 8GB de memoria RAM.

1. Ejecuta `dd if=/dev/zero of=/mnt/swapfile bs=1M count=xxxx status=progress` y reemplaza "xxxx" con la cantidad de megabytes que quieres darle a tu swapfile
2. Ejecuta `chmod 600 /mnt/swapfile` para darle los permisos correctos
3. Ejecuta `mkswap /mnt/swapfile` para darle formato de swap
4. Ejecuta `swapon /mnt/swapfile` para activar la swapfile

# Pacstrap
1. Ahora para instalar los archivos de Arch Linux, ejecuta `pacstrap -K /mnt base base-devel linux linux-firmware neovim`
> Puedes reemplazar neovim con tu editor de texto en terminal preferido

# Generando /etc/fstab
> Este es un paso bastante importante, este archivo le dice a tu sistema operativo que particiones montar y donde al iniciar.
1. Ejecuta `genfstab -U /mnt >> /mnt/etc/fstab` para generar una archivo fstab utilizando UUIDs de particiones

# Haciendo chroot en el nuevo entorno
1. Ejecuta `arch-chroot /mnt` para cambiar de terminal a tu nuevo entorno Arch Linux

# Configuraciones regionales
1. Ejecuta `ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime` *(reemplaza /Europe/Madrid con tu zona horaria correcta)*
2. Ejecuta `hwclock --systohc`
3. Edita /etc/locale.gen con tu editor preferido y descomenta las regiones que deseas utilizar *(personalmente utilizo en_US y es_ES)*
4. Ejecuta `locale-gen` para generar los datos de región
5. Ejecuta `echo 'LANG=en_US.UTF-8' > /etc/locale.conf`
> El comando de arriba cambiará el idioma de visualización de tu sistema operativo, si deseas utilizar español o lo que sea, modifica el comando acordemente.
6. Ejecuta `echo 'KEYMAP=es' > /etc/vconsole.conf`
> Este se encarga de que distribución de teclado será utilizada por defecto en las TTYs, esto te ahorrará un dolor de cabeza después cuando inicies tu instalación, y como siempre, si utilizas una distribución diferente, modifica acordemente.

# Configurar el nombre de host
1. Ejecuta `echo 'genesis' > /etc/hostname` y reemplaza genesis con tu nombre de host preferido
2. Modifica /etc/hosts con tu editor preferido e inserta las siguientes líneas:
```bash
127.0.0.1   localhost
::1         localhost
```

# Configurar initramfs para poder iniciar sistemas encriptados
1. Modifica /etc/mkinitcpio.conf con tu editor preferido y en la línea `HOOKS`, añade `encrypt` entre `block` y `filesystems` para que se parezca a algo así:
```
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
```
2. Ejecuta `mkinitcpio -P`

# Instalando el bootloader
> Reemplaza amd-ucode con intel-ucode si tienes un procesador Intel.
1. Ejecuta `pacman -S grub efibootmgr amd-ucode` para instalar el bootloader y el microcódigo de CPU
2. Ejecuta `echo "GRUB_CMDLINE_LINUX=cryptdevice=UUID=$(blkid -s UUID -o value /dev/sdX2):crypt" >> /etc/default/grub`
> Recuerda reemplazar sdX2 con la partición de disco correcta, en caso contrario no podrás iniciar tu sistema!
3. Después de ejecutar el comando de arriba, abre `/etc/default/grub` con tu editor preferido y reemplaza el "GRUB_CMDLINE_LINUX" original con el que has insertado dentro del archivo
4. Sin cerrar tu editor, por favor recuerda descomentar la línea `GRUB_ENABLE_CRYPTODISK=y` dentro del archivo
5. Ejecuta `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
6. Ejecuta `grub-mkconfig -o /boot/grub/grub.cfg`

# Configurando la contraseña root
1. Ejecuta `passwd` para configurar tu contraseña root

# Toques finales
Eso es todo! Ahora ***podríamos*** reiniciar si queremos y tendríamos un sistema en "funcionamiento", pero hay algunas cosas de las que deberíamos encargarnos mientras sigamos aquí:

## Instalando NetworkManager / iwd
Estos dos nos ayudarán a conectarnos al Internet cuando inicies Arch Linux, ***pero solo puedes escoger uno de ellos***, NetworkManager es más amistoso para novatos pero un poco pesado en recursos, mientras que iwd es un daemon de red mucho más minimalista, si no sabes que elegir, solo escoge NetworkManager:

### NetworkManager
1. Ejecuta `pacman -S networkmanager` para instalar el daemon de red
2. Una vez terminado, ejecuta `systemctl enable NetworkManager` para que el daemon empieze al reiniciar el sistema
> Por cierto, recuerda las letras mayúsculas al tratar con NetworkManager, si ejecutas `systemctl enable networkmanager` no hará nada.
> Una vez que reinicies el sistema, lo único que tienes que hacer es ejecutar `nmtui`, que mostrará un menú de texto gráfico para conectarte a tu red inalámbrica.

### iwd
1. Ejecuta `pacman -S iwd` para instalarlo
2. Ejecuta `systemctl enable iwd`
> Para conectarte a Wi-Fi después del reinicio, solo tienes que seguir los mismos pasos del principio de la guía, `iwctl`, `station wlan0 connect`, etc...

## Creando una cuenta de usuario
> Recuerda reemplazar "raul" con tu nombre de usuario preferido!
1. Ejecuta `useradd -m -G wheel,games,network,audio,video -s /bin/bash raul`
2. Ejecuta `EDITOR=nvim visudo` y descomenta la línea `%wheel ALL=(ALL:ALL) ALL`
> Reemplaza nvim arriba con tu editor preferido, este paso le dará a tu usuario creado privilegios administrativos.
3. Ejecuta `passwd raul` o cual sea tu usuario, y dale a tu cuenta una contraseña también

## Reduciendo la swappiness
> Swappiness es la frecuencia con la cual tu sistema hará uso de la memoría de disco, a no ser que tengas 4 GB de RAM, querrás reducir este valor para incrementar el rendimiento de tu sistema, ajusta el valor a lo que se adapte bien a tu sistema.
1. Ejecuta `echo 'vm.swappiness=20' > /etc/sysctl.d/99-swappiness.conf`

# Terminando
Eso debería ser todo! Ahora pulsa Ctrl+D para salir de la sesión chroot y ejecuta `reboot` para iniciar tu nueva instalación de Arch Linux *(recuerda desconectar la unidad USB después de reiniciar)*, una vez atravieses la pantalla de inicio de sesión te darás cuenta de que no hay nada más que una terminal, eso es porque aquí es donde empieza la aventura de verdad, probablemente querrás un entorno de escritorio para utilizar tu ordenador.

> Por cierto, recuerda el consejo de antes de utilizar `nmtui` para conectarte a tu red Wi-Fi.

Mientras que esto vaya en contra de la esencia de construir tu propio entorno de trabajo, si quieres algo que funcione y ya está, puedes ejecutar el siguiente comando:
```bash
sudo pacman -Syu xorg xorg-server ffmpeg4.4 ffmpegthumbnailer tumbler gvfs ttf-roboto ttf-roboto-mono xfce4 xfce4-goodies lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings pulseaudio pulseaudio-alsa pulseaudio-jack && sudo systemctl enable lightdm
```
Y reinicia tu sistema! XFCE es un gran entorno de escritorio muy ligero y que *"funciona y ya está"*.
