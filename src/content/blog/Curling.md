---
title: 'Easy Machine | Curling'
description: 'Writeup de la maquina'
pubDate: 'May 14 2026'
heroImage: ''
tags: ['Reverse Shell', 'HexDump', 'Curl']
---

> Curling is an Easy difficulty Linux box which requires a fair amount of enumeration. The password is saved in a file on the web root. The username can be download through a post on the CMS which allows a login. Modifying the php template gives a shell. Finding a hex dump and reversing it gives a user shell. On enumerating running processes a cron is discovered which can be exploited for root.

Empezamos con el escaneo clasico de nmap, tenemos 2 puertos abiertos 22 y 80
```bash
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```
Realizo un segundo escaneo para ver mas detalles de estos puertos: 

![scan](/curling/scan.png)

Viendo un poco la pagina hosteada en el Apache me doy cuenta que hay un posible super usuario "Floris"

![floris](/curling/floris.png)

Inspeccionando el HTML me doy cuenta que hay algo comentado 

![secret](/curling/secret.png)

Voy al path 10.129.66.180/secret.txt y me encuentro algo encodeado en base64
```bash
Q3VybGluZzIwMTgh
```

Decodeandolo en la terminal me da la contrasena

```bash
echo Q3VybGluZzIwMTgh | base64 -d
Curling2018!
```

Logueamos con el usuario
![login](/curling/usuariofloris.png)

Vamos directamente a /admin, /administrator o derivados para tener un panel de admin:

![panel](/curling/paneladmin.png)

Dentro vemos que hay varias secciones, viendo alguna manera de obtener una reverse shell puede ser editando los templates o crear un archivo a los templates.
Encuentro en internet una reverse shell en [Github](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)


![shell](/curling/shell.png)

La subimos y  me pongo en escucha con nc y en el navegador vamos a /templates/protostar/shell.php 

![netcat](/curling/ncat.png)

Mejoramos la TTY

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

No puedo ver el user.txt pero puedo ver "password_backup" que es un archivo hexdump (una representacion en texto de datos binarios mostrados en bytes hexadecimales).

![passwd](/curling/password.png)

Averiguando que es encontre que  esos primeros bytes (BZh9) indican que es un archivo comprimido en formato bzip2, la firma es "BZh".

Asi que tengo que reconstruir la contrasena y pasa todo esto, tuve que ver una guia de "0xrick" para poder obtener la password

![rearmado](/curling/rearmado.png)

Lo que paso fue: 
  - xxd recrea el binario desde un hexdump
  - resulto ser un .bz2
  - descomprimiendo el .bz2 y viendo el file resulta ser un .gz
  - descomprimiendo el .gz vuelve a un archivo .bz2
  - y al final un .tar 
  - extraigo el archivo y me da la password.
  

```text
5d<wdCbdZu)|hCh***
```

Ahora con la contrasena puedo acceder como floris en ssh
Ya como floris obtenemos la flag user.txt

```bash
floris@curling:~$ cat user.txt 
dd9ff13edeb9d9fb79629cb89a******
```

Tengo que escalar privilegios como root para obtener la root flag, intento con `sudo -l` para ver los permisos pero no me deja, asi que tengo que tratar de otra manera.

Me llama la atencion el directorio "admin-areas" asi que ejecuto `ls -la` para ver permisos y si hay archivos ocultos, pero encuentro 2 archivos con un timestamp asi que supongo que debe estar ejecutandose cada cierto tiempo.

El archivo iinput tiene 
```text
url = http://127.0.0.1 
```
y un output que es codigo de la pagina principal del comienzo.

![time](/curling/timestamp.png)

Asi que modifico el input a ```url = "file:///root/root.txt"``` para poder leer la root flag. Espero unos segundos para ejecutar

```bash
floris@curling:~/admin-area$ cat report 
41e7ff4b9f4c041f66b9b4520c******
```
Y obtenemos la root flag!
