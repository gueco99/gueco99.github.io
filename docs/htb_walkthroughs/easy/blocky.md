# HTB - Blocky Walkthrough

## Introducción
En esta máquina de HackTheBox llamada **Blocky**, el objetivo era conseguir acceso inicial al sistema y escalar privilegios hasta root.  
Durante el proceso aproveché credenciales expuestas en un archivo `.jar` y una configuración insegura de `sudo`.

---

## Reconocimiento

Primero lancé un escaneo completo con Nmap para descubrir puertos y servicios:

```bash
nmap -sC -sV -p- -T4 10.10.10.37
```
![Nmap scan](/assets/images/blocky/01-nmap.png)

El resultado me mostró varios servicios interesantes:
- **21/tcp** – FTP (ProFTPD 1.3.5a)
- **22/tcp** – SSH (OpenSSH 7.2p2)
- **80/tcp** – HTTP (Apache 2.4.18)
- **25565/tcp** – Minecraft Server

---

## Enumeración Web

Al visitar `http://blocky.htb` me encontré con una web tipo blog de Minecraft, aún en construcción.

![BlockyCraft](/assets/images/blocky/02-web.png)

---

## Descubrimiento de directorios

Ejecuté Gobuster para buscar rutas ocultas:

```bash
gobuster dir -u http://blocky.htb/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 50 -x php,txt,html
```
![Gobuster](/assets/images/blocky/03-gobuster.png)

Esto me reveló un directorio `/plugins/` con acceso público.

---

## Análisis de plugins `.jar`

En `/plugins/` encontré dos ficheros `.jar`. Me descargué **BlockyCore.jar** para revisarlo.

![Plugins](/assets/images/blocky/04-plugins.png)

Con **JD-GUI** abrí el archivo y dentro encontré credenciales de base de datos en texto plano:

```java
public String sqlUser = "root";
public String sqlPass = "8YsqfCTnvxAUeduzjNSXe22";
```
![Credenciales](/assets/images/blocky/05-jdgui.png)

---

## Acceso a phpMyAdmin

Probé esas credenciales en `http://blocky.htb/phpmyadmin` y funcionaron:

```
Usuario: root
Contraseña: 8YsqfCTnvxAUeduzjNSXe22
```
![phpMyAdmin](/assets/images/blocky/06-phpmyadmin.png)

---

## Extracción de credenciales de WordPress

Dentro de phpMyAdmin, en la tabla `wp_users`, encontré el usuario **notch** con su hash de contraseña:

![Hash Notch](/assets/images/blocky/07-wp_users.png)

El hash correspondía al tipo **phpass**. Al crackearlo, resultó ser la misma contraseña encontrada en el `.jar`.

---

## Acceso SSH

Con el usuario `notch` y la misma contraseña, pude iniciar sesión por SSH:

```bash
ssh notch@10.10.10.37
```
![SSH User](/assets/images/blocky/08-ssh_user.png)

---

## Escalada de privilegios

Al revisar permisos de `sudo`, vi que `notch` podía ejecutar cualquier comando como root.  
Bastó con:

```bash
sudo -i
```
![Root](/assets/images/blocky/09-root.png)

---

## Flags

- **User flag**: `cat /home/notch/user.txt`
- **Root flag**: `cat /root/root.txt`

---

## Conclusión

Esta máquina se resolvió aprovechando:
- Credenciales expuestas en un plugin `.jar`.
- Reutilización de contraseñas en varios servicios.
- Configuración insegura de `sudo`.

Fue un reto entretenido y directo, ideal para practicar enumeración web y análisis de ficheros Java.
