---
title: "Picklerick - TryHackMe"
layout: post
date: 2023-12-05 22:10
tags:
  - TryHackMe
  - Picklerick
  - information-disclosure
  - RCE
image: /assets/images/picklericky.jpeg
headerImage: true
projects: false
hidden: false
description: "Picklerick - TryHackMe"
category: blog
author: CAHOn3R
externalLink: true
---
# Fase Reconocimiento

Empezamos la **fase de reconocimiento** escaneando los puertos de la máquina para ver por donde podemos atacar...

![[Pasted image 20231205184846.png]]
```bash
-p- # para escanear los 65535 puertos que existen
-sS # para realizar un escaneo tipo "SYN scan"
--min-rate 5000 # para indicarle no se envien menos de 5k paquetes por segundo
--open # para que solo reporte puertos abiertos
-vvv # para que muestre por consola lo máximo posible y lo antes posible
-n # para realizar un escaneo prescindiendo de la resolución de nombres de dominio
-Pn # para que no realice la detección del estado del host antes de iniciar el escaneo
-oG # para especificar un formato de salida en modo greppable en el archivo allPorts
```

Lo siguiente que hago es utilizar la herramienta de [s4vitar](https://github.com/s4vitar?tab=repositories) **extractPorts** para que me copie los puertos en la clipboard y poder realizar mas cómodamente la detección de versiones y servicios para estos puertos de la siguiente manera
![[Pasted image 20231205190034.png]]
```bash
-sC # Activa los scripts de secuencia de comandos predeterminados (también conocidos como "scripts de inicio") en Nmap. Estos scripts realizan una variedad de tareas para detectar servicios y vulnerabilidades comunes en un sistema.
-sV # Intenta determinar las versiones de servicios que están en ejecución en los puertos abiertos. 
```
como podéis apreciar ambos parámetros se pueden fusionar

![[Pasted image 20231205190437.png]]
observamos que hay 2 puertos abiertos **22 ssh y 80 http**, también podemos apreciar que nos enfrentamos a una una maquina **Linux**. En lo primero que me fijo es en la version de openSSH ya que es bastante antigua y puede ser vulnerable **user enumeration**, lo tendré en mente para un futuro...

lo siguiente es echar un vistazo a la web

![[Pasted image 20231205191224.png]]
Parece que Monty esta en apuros y tenemos que ayudarle, así que tenemos un nombre de usuario o al menos un posible nombre de usuario y algo que una pista **burpsuite?**

hago **ctrl+u** para ver el código fuente y nos encontramos con un comentario
![[Pasted image 20231205191516.png]]
Esto si que parece un nombre de usuario **R1ckRul3s** me lo guardo en notas y empezamos con el fuzzing a la web, no encuentro ninguna ruta interesante así que **fuzzeo** archivos con extensiones típicas
![[Pasted image 20231205192330.png]]
rápidamente vemos archivos que llaman la atención, también me llama la atención que **namp** no haya reportado el robots.txt 🤷‍♂️🤷‍♂️
Accedo al robots.txt y veo lo que parece ser una password **Wubbalubbadubdub** mi primera idea es conectarme por **ssh** pero como también acabamos de ver un **login.php** para allá que voy...
![[Pasted image 20231205192730.png]]
lo que vemos es un panel de comandos a si que intento entablar una **reverse shell** pero no me deja
![[Pasted image 20231205193335.png]]
Así que hago un **ls -la** y veo un archivo un tanto sospechoso 🤦‍♂️ **Sup3rS3cretPickl3Ingred.txt** pongo el archivo en la url y ahí tenemos la primera flag de la maquina 1/3.

Mientras buscaba alguna manera de entrar a la máquina me encuentro con el segundo ingrediente en el directorio de rick, puesto que no me deja hacerle un **cat** por que hay comandos capados le hago un **less** y tenemos el segundo ingrediente de esta máquina la segunda flag 2/3.

hago un **sudo -l** para comprobar los permisos que tenemos como www-data y resulta que podemos ejecutar cualquier comando como el usuario **root**, así que ejecuto el siguiente comando para listar el directorio **/root** y ahí vemos el tercer ingrediente...
![[Pasted image 20231205194705.png]]
Hago un **less /root/3rd.txt** y tenemos la tercera flag 3/3.

# conclusión
Una maquina muy sencillita, no está mal para muy principiantes pero obviamente no es nada realista...
