---
title: Broker
parent: Easy
nav_order: 1
---

# Walkthrough — Broker (Hack The Box)
{: .fs-9 }

> Walkthrough para la máquina *Broker* de HTB, enfocada en enumeración de puertos y explotación básica.
{: .fs-6 .fw-300 }

---

## Introducción

**Broker** es una máquina de nivel *Easy* en Hack The Box. El enfoque principal está en identificar puertos expuestos mediante escaneo completo y explotar los servicios visibles.

- SO: Linux  
- Dificultad: Media  
- Herramientas utilizadas: `nmap`

---

## 🔎 Enumeración inicial

Lo primero que hice fue lanzar un escaneo completo de puertos usando `nmap`, incluyendo scripts por defecto y detección de versiones, para ver qué exponía la máquina. El comando fue el siguiente:

```bash
nmap -sC -sV -p- -T4 10.10.11.243
```

> `-sC`: Usa los scripts por defecto de NSE para una enumeración básica.  
> `-sV`: Detecta versiones de los servicios.  
> `-p-`: Escanea los 65535 puertos, no solo los comunes.  
> `-T4`: Acelera un poco el escaneo sin perder mucha estabilidad.

![Resultado de Nmap](/assets/images/broker/01-nmap.png)

## 🧠 Análisis de servicios detectados

Este escaneo me da una visión completa del panorama de servicios que corren en la máquina:

- **22/tcp (SSH)**: Servicio típico de administración remota.  
  👉 Por ahora no me sirve mucho, pero lo apunto para post-explotación. Si consigo credenciales, podría ser una vía rápida de entrada o escalada.  
- **80/tcp (HTTP)**: Servidor web Nginx funcionando... ¡pero nos recibe con un **401 Unauthorized**!  
  El detalle curioso es el *realm* `"ActiveMQRealm"`, que me da una pista muy jugosa: probablemente hay una instancia de **Apache ActiveMQ** detrás.

📌 *Esto nos puede llevar a un vector de ataque más adelante. Tomo nota...*

```bash
nmap -sC -sV -p- -T4 10.10.11.243
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 SHA256:a45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|   256 SHA256:cc:75:de:4a:e6:i3:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: 401 Unauthorized
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_basic realm=ActiveMQRealm
```

![Resultado de Nmap](/assets/images/broker/02-nmap.png)

## 🔐 Acceso al panel web de ActiveMQ

Al acceder vía HTTP a la IP `10.10.11.243`, el navegador me sorprendió con una ventana de autenticación básica:

![Prompt de autenticación](/assets/images/broker/03-auth-popup.png)

Esto confirma lo que ya sospechábamos desde el escaneo de puertos: ¡hay un servicio de **Apache ActiveMQ** corriendo detrás!

---

### 🤔 ¿Qué hice?

Probé con las credenciales por defecto más clásicas del universo de las malas prácticas:

```text
Username: admin  
Password: admin
```

🎯 **¡Y funcionó!**

---

Una vez dentro, fui redirigido a la interfaz de administración de ActiveMQ:

![Panel de ActiveMQ](/assets/images/broker/04-activemq-panel.png)

Desde aquí se pueden ver opciones para:
- Administrar el broker
- Visualizar demos web (¡a veces útiles para encontrar rutas inseguras!)
- Navegar documentación del framework

---

### 🧠 Conclusión

💡 El hecho de que ActiveMQ esté corriendo con las credenciales por defecto nos da acceso total al panel de administración. Esto puede abrir la puerta a:
- **Carga de archivos maliciosos**
- **Ejecución de comandos**
- **Explotación de versiones vulnerables**

¡Hora de ver qué versión corre y buscar un exploit!
