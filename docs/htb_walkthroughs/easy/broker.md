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

