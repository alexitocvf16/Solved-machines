# Solved Machines 🕵️‍♂️

Repositorio donde voy subiendo los **writeups** de las máquinas vulnerables que resuelvo, tanto de **Red Team** (explotación y escalada de privilegios) como de **Blue Team** (análisis forense, respuesta a incidentes, análisis de tráfico).

La idea es compartir mi forma de trabajar y que, de paso, le sirvan a alguien de la comunidad en el futuro 🙂

## 📂 Máquinas

| Máquina | Plataforma | Enfoque | Writeup |
|---|---|---|---|
| **Analyst** | SOC-LAB (Blue Team) | Análisis forense de tráfico de red con Wireshark — investigación de un incidente de intrusión (webshell + bypass de extensión + escalada de privilegios) a partir de una captura `.pcap` | [📄 Writeup](./Máquina%20analyst/README.md) |
| **Sedition** | HackerLabs | Enumeración SMB, explotación y escalada de privilegios | [📄 Writeup](./Maquina%20sedition%20hackerlabs/Máquina%20Sedition%20Hackerlabs.md) |
| **Canto** | HackMyVM | Enumeración de servicios, explotación web y escalada de privilegios | [📄 Writeup](./Máquina%20canto%20hackmyvm/Máquina%20Canto.md) |
| **Darkweb** | — | Explotación de servicios web y escalada de privilegios hasta root | [📄 Writeup](./Máquina%20darkweb/Darkweb.md) |
| **Grooti** | Propia (creada por mí) | Mi primera máquina vulnerable propia — explotación de web y base de datos | [📄 Writeup](./Máquina%20grooti/Máquina%20grooti.md) |
| **Ofuspingu** | HackerLabs | Fuzzing web, desofuscación de JavaScript y explotación | [📄 Writeup](./Máquina%20ofuspingu%20hackerslabs/Máquina%20ofuspingu.md) |
| **Usersearch** | DockerLabs | Enumeración de servicios, explotación web y escalada de privilegios | [📄 Writeup](./Máquina%20usersearch%20dockerlabs/Máquina%20usersearch%20dockerlabs.md) |
| **Spiderport** | DockerLabs | Enumeración web, explotación y escalada de privilegios | [📄 Writeup](./Spiderport%20Dockerlabs/SPIDERPORT.md) |

## 🎯 Tipos de contenido

- **🔴 Red Team**: explotación de servicios, abuso de vulnerabilidades web, escalada de privilegios, obtención de `root`/`system`.
- **🔵 Blue Team**: análisis forense de capturas de red, reconstrucción de cadenas de ataque, identificación de IOCs y respuesta a incidentes.

## 🛠️ Herramientas habituales

`nmap` · `gobuster` · `Wireshark` · `smbmap` · bypass de filtros web · escalada de privilegios en Linux (`sudo -l`, binarios mal configurados, etc.)

## 📌 Nota

Cada carpeta corresponde a una máquina e incluye su writeup paso a paso con las capturas de pantalla del proceso. En las máquinas de análisis forense (Blue Team) se incluye además un `README.md` propio con el contexto del caso y los hallazgos principales.
