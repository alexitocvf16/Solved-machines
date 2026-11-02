# Máquina Analyst — SOC-LAB // Ticket #PGN-2026-0417

## Descripción

Máquina de **DockerLabs** orientada a **Blue Team**, centrada en **análisis forense de tráfico de red con Wireshark**. No se trata de una máquina de explotación: no hay acceso en vivo al servidor comprometido, toda la investigación se realiza sobre una captura de red (`.pcap`) entregada como evidencia, tal y como ocurriría en un caso real de respuesta a incidentes.

El escenario sitúa al analista como parte de un equipo de respuesta a incidentes, investigando el hackeo sufrido por la intranet corporativa de una tienda online ficticia ("Reviews & Co."). El equipo de desarrollo del cliente detectó un archivo sospechoso en el directorio de subida de imágenes de la sección de reseñas de producto, y poco después un administrador reportó que ya no podía autenticarse con la cuenta de servicio `pinguinito`.

## Objetivo

A partir del `.pcap` proporcionado (`incidente_pinguino.pcap`), reconstruir la cadena de ataque completa y determinar qué información llegó a verse comprometida.

## Metodología aplicada

1. **Reconocimiento inicial** con `nmap` sobre la máquina del laboratorio, identificando los servicios SSH (22) y HTTP (80) expuestos.
2. **Obtención de la evidencia**: la web del laboratorio entrega el ticket del incidente junto con el archivo de captura de tráfico.
3. **Análisis de la captura con Wireshark**, aplicando filtros de forma progresiva para reconstruir el ataque:
   - `http.request.method=="POST"` → localizar las peticiones de subida de archivos e identificar la IP del atacante.
   - **Follow → HTTP Stream** sobre cada petición POST para inspeccionar el contenido real enviado y la respuesta del servidor.
4. **Identificación del intento de subida bloqueado**: un primer intento de subir `shell.php` es rechazado por el servidor al no permitir la extensión `.php`.
5. **Identificación del bypass**: un segundo intento camufla la webshell como imagen (`image.jpg.php`), logrando saltarse la validación de extensión y quedando alojada en `/reviews/uploads/image.jpg.php`.
6. **Localización de la reverse shell**, filtrando los paquetes SYN salientes desde el servidor hacia la IP del atacante (`ip.src==10.10.2.15 and ip.dst==198.51.100.23 and tcp.flags.syn==1 and tcp.flags.ack==0`).
7. **Reconstrucción de la sesión de la reverse shell** mediante Follow → HTTP/TCP Stream, recuperando todos los comandos ejecutados remotamente por el atacante.

## Hallazgos clave

- **IP del atacante**: `198.51.100.23`.
- **Vector de entrada**: subida de archivo malicioso (`upload.php`) con bypass de filtrado de extensiones mediante doble extensión (`image.jpg.php`).
- **Webshell** alojada en `/reviews/uploads/image.jpg.php`, usada para establecer una reverse shell.
- **Post-explotación**: el atacante enumeró el sistema (`id`, `uname -a`, `/etc/passwd`), comprobó privilegios con `sudo -l` y encontró un binario ejecutable como root sin contraseña (`/opt/scripts/reset_user_pass.sh`).
- **Escalada de privilegios**: abuso de ese script para resetear la contraseña del usuario `pinguinito`, obteniendo así credenciales válidas de una cuenta con shell interactiva (`/bin/bash`), lo que explica por qué el administrador perdió el acceso con dicha cuenta.

## Conclusión

El incidente se resume en: subida de webshell mediante bypass de validación de extensión → ejecución remota de comandos vía reverse shell → abuso de un privilegio `sudo` mal configurado para resetear la contraseña de la cuenta de servicio `pinguinito` → acceso persistente a la máquina como ese usuario, con capacidad de escalar a `root` mediante `sudo su`.

## Herramientas utilizadas

- `nmap`
- `Wireshark` (filtros de visualización, Follow TCP/HTTP Stream)

➡️ Writeup completo con capturas paso a paso: [Analyst.md](./Analyst.md)
