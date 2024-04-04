---
title: "Sputnik: 1 - Vulnhub"
layout: post
date: 2024-04-03 15:10
tags:
  - Vulnhub
  - Sputnik
  - Web Enumeration
  - Github Project Enumeration
  - Information Leakage
  - Splunk Enumeration
  - Splunk Exploitation
  - Abusing sudoers privilege (ed command)
image: /assets/images/Vulnhub/Sputnik/sputnik.png
headerImage: true
projects: false
hidden: false
description: Sputnik - Vulnhub
category: blog
author: CAHOn3R
externalLink: false

# Enumeración
Comenzamos la exploración de la máquina, como es habitual en nuestras pruebas locales, utilizando **arp-scan** para detectar la **IP**.
```bash
arp-scan -I eth0 --localnet --ignoredups

192.168.0.29	00:0c:29:ed:1f:ab  VMware, Inc.
```

Una vez obtenida la IP, continuamos la **enumeración** con **nmap**.
```bash
nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn -oG allPorts 192.168.0.29
nmap -sCV -p 8089,55555,61337 192.168.0.29 -oN targeted

PORT      STATE SERVICE  VERSION
8089/tcp  open  ssl/http Splunkd httpd
|_http-title: splunkd
|_http-server-header: Splunkd
| ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
| Not valid before: 2019-03-29T11:03:21
|_Not valid after:  2022-03-28T11:03:21
| http-robots.txt: 1 disallowed entry 
|_/
55555/tcp open  http     Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Flappy Bird Game
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-git: 
|   192.168.0.29:55555/.git/
|     Git repository found!
|_    Repository description: Unnamed repository; edit this file 'description' to name the...
61337/tcp open  http     Splunkd httpd
|_http-server-header: Splunkd
| http-robots.txt: 1 disallowed entry 
|_/
| http-title: Site doesn\'t have a title (text/html; charset=UTF-8).
|_Requested resource was http://192.168.0.29:61337/en-US/account/login?return_to=%2Fen-US%2F

```

Al explorar, identificamos varios servicios web. Al lanzar los scripts básicos de reconocimiento de **nmap**, descubrimos un directorio **.git** en el puerto **55555**. Extraemos este repositorio utilizando [git_dumper.py](https://github.com/arthaud/git-dumper).
```bash
./git_dumper.py http://192.168.0.29:55555 ../website
```

Con el repositorio local, podemos examinar más detenidamente el **git**. En el directorio **.git/logs**, encontramos el archivo **HEAD**, que nos dirige a un repositorio en GitHub, que también clonamos en nuestro equipo.
[<img src="/assets/images/vulnhub/Sputnik/captura1.png">](/assets/images/vulnhub/Sputnik/captura1.png)


```bash
git clone https://github.com/ameerpornillos/flappy
```

[<img src="/assets/images/vulnhub/Sputnik/captura2.png">](/assets/images/vulnhub/Sputnik/captura2.png)

Después de un análisis exhaustivo del **git**, encontramos un **secreto** que parece ser un par de **credenciales** (usuario y contraseña).

Recordando un panel de autenticación en el puerto **61337**, probamos estas credenciales y tenemos acceso...

# Explotación
Una vez dentro, investigo el software **Splunk**, que es el servicio web al que hemos accedido, y encuentro un **poc** en https://github.com/TBGSecurity/splunk_shells?tab=readme-ov-file. Este **poc** explica cómo cargar un **reverse shell** en el servidor y lanzarlo a nuestra máquina.

Primero, descargamos el archivo comprimido que contiene las diferentes **reverse shells** que necesitamos en nuestra máquina atacante.
```bash
wget https://github.com/TBGSecurity/splunk_shells/archive/1.2.tar.gz
```

Luego, en la interfaz web de **Splunk**, nos dirigimos a "**Manage Apps**" y seleccionamos "**Install app from file**".
[<img src="/assets/images/vulnhub/Sputnik/captura3.png">](/assets/images/vulnhub/Sputnik/captura3.png)

Seleccionamos nuestro archivo comprimido "**splunk_shells-1.2.tar.gz**", reiniciamos Splunk.

Navegamos nuevamente a "**Manage Apps**" y seleccionamos "**Permisos**".
[<img src="/assets/images/vulnhub/Sputnik/captura4.png">](/assets/images/vulnhub/Sputnik/captura4.png)

[<img src="/assets/images/vulnhub/Sputnik/captura5.png">](/assets/images/vulnhub/Sputnik/captura5.png)
Marcamos la casilla "**All apps**" y seleccionamos "**Save**".

[<img src="/assets/images/vulnhub/Sputnik/captura6.png">](/assets/images/vulnhub/Sputnik/captura6.png)

Luego, vamos a "**Search & Reporting**" y podemos ejecutar la **reverse shell** como se muestra en el recuadro verde, aunque también podríamos usar **Meterpreter** según el [poc](https://github.com/TBGSecurity/splunk_shells?tab=readme-ov-file), pero prefiero hacerlo todo manualmente si es posible.

Iniciamos una escucha con **netcat** en el puerto **443** y lanzamos la **reverse shell**.
```bash
nc -nvlp 443
```

Ahora estamos dentro de la máquina víctima, pero notamos que la **shell** es inestable y se desconecta rápidamente. Opto por enviar otra **shell** desde esta misma.

# Escalada de Privilegios
[<img src="/assets/images/vulnhub/Sputnik/captura7.png">](/assets/images/vulnhub/Sputnik/captura7.png)
Recibimos la **shell** de nuevo y la estabilizamos utilizando **nc mkfifo**, luego trabajamos en mejorar la **TTY** y enumerar el sistema para escalar privilegios.

Literalmente, lo primero que intentamos es un éxito: **sudo -l** nos permite ejecutar el binario **/bin/ed** como usuario **root**. Consultamos [gtfobins](https://gtfobins.github.io/gtfobins/ed/#sudo).
[<img src="/assets/images/vulnhub/Sputnik/captura8.png">](/assets/images/vulnhub/Sputnik/captura8.png)

En [gtfobins](https://gtfobins.github.io/gtfobins/ed/#sudo), filtramos por el binario **ed** y buscamos los permisos que tenemos, que en este caso es **SUID**. Nos indica que ejecutemos el binario y dentro de este, lanzamos una **shell**.
[<img src="/assets/images/vulnhub/Sputnik/captura9.png">](/assets/images/vulnhub/Sputnik/captura9.png)
Ahora somos **root** y podemos acceder a la bandera de alto privilegio.
