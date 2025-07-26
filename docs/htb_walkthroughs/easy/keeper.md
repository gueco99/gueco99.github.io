---
title: Keeper
parent: Easy
nav_order: 2
---

# Walkthrough — Keeper (Hack The Box)
{: .fs-9 }

> Recorrido completo y detallado de la máquina **Keeper** de Hack The Box, en la que se explota una configuración por defecto de *Request Tracker*, junto con una fuga de datos sensibles en archivos KeePass.
{: .fs-6 .fw-300 }

---

## 🔰 Introducción

**Keeper** es una máquina de dificultad *fácil*, que presenta una interesante cadena de vulnerabilidades: acceso inicial mediante credenciales por defecto en un sistema de tickets (*Request Tracker*), acceso a usuario por SSH, y posterior escalada a root mediante extracción de contraseñas desde un volcado de KeePass.

- **Sistema operativo**: Linux  
- **Dificultad**: Easy  
- **Herramientas utilizadas**: `nmap`, `burpsuite`, `scp`, `keepassxc`, `puttygen`, `ssh`

---

## 🔍 Escaneo de puertos con Nmap

Iniciamos el reconocimiento con un escaneo completo de puertos usando `nmap`:

```bash
nmap -sC -sV -p- -T4 10.10.11.227
```

![Escaneo con Nmap](/assets/images/keeper/01-nmap.png)

El resultado muestra dos puertos relevantes:
- **22/tcp**: OpenSSH 8.9p1
- **80/tcp**: nginx 1.18.0

---

## 🌐 Acceso al portal web

Al visitar `http://10.10.11.227`, el navegador redirige automáticamente a `http://tickets.keeper.htb`, lo cual nos revela que el sistema utiliza subdominios. El portal web pertenece a la aplicación **Request Tracker (RT)** y muestra un formulario de login.

![Login de RT](/assets/images/keeper/02-rt-login.png)

---

## 🔑 Uso de credenciales por defecto

Basado en documentación de RT, probamos las siguientes credenciales:

- Usuario: `root`
- Contraseña: `password`

El acceso fue exitoso.

![Credenciales por defecto](/assets/images/keeper/03-default-creds.png)

---

## 🧭 Navegación por el panel RT

Una vez dentro, se nos presenta el panel principal del sistema RT.

![Dashboard de RT](/assets/images/keeper/04-rt-dashboard.png)

Desde la sección **Admin → Users**, encontramos un usuario adicional llamado `lnorgaard`.

![Usuarios en RT](/assets/images/keeper/05-users-panel.png)

---

## 🧪 Descubrimiento de contraseña inicial

Editando el perfil del usuario `lnorgaard`, encontramos el siguiente comentario:

```
Initial password set to Welcome2023!
```

![Comentario con contraseña](/assets/images/keeper/06-user-password.png)

---

## 📥 Acceso SSH como lnorgaard

Probamos por SSH con la contraseña obtenida:

```bash
ssh lnorgaard@10.10.11.227
```

¡Acceso exitoso! Dentro del sistema encontramos los archivos `user.txt` y un archivo comprimido `RT30000.zip`.

![Flag de usuario y ZIP](/assets/images/keeper/07-ssh-user-flag.png)

---

## 📤 Transferencia del archivo ZIP

Transferimos el archivo a nuestra máquina con `scp`:

```bash
scp lnorgaard@10.10.11.227:/home/lnorgaard/RT30000.zip .
```

![Transferencia con SCP](/assets/images/keeper/08-scp-zip.png)

---

## 📦 Análisis del ZIP

El archivo contiene dos ficheros:

- `KeePassDumpFull.dmp`
- `passcodes.kdbx`

![Contenido del ZIP](/assets/images/keeper/09-zip-content.png)

---

## 🧠 Extracción de clave con script

Usamos este script público:

https://github.com/matro7sh/keepass-dump-masterkey/blob/main/poc.py

```bash
python3 poc.py -d KeePassDumpFull.dmp
```

Resultado:

```
rødgrød med fløde
```

![Extracción de contraseña](/assets/images/keeper/10-keepass-crack.png)

---

## 🔐 Abrimos KeePassXC

Con `keepassxc` y la contraseña obtenida, abrimos el archivo `.kdbx`:

```bash
keepassxc passcodes.kdbx
```

Dentro hallamos credenciales para el usuario `root` y una clave privada `.ppk`.

![Abriendo base KeePassXC](/assets/images/keeper/11-keepassxc.png)
![Conversión de clave](/assets/images/keeper/12-puttygen.png)
---

## 🔁 Conversión de clave a OpenSSH

Convertimos la clave con `puttygen`:

```bash
puttygen id_rsa.ppk -O private-openssh -o id_rsa
chmod 600 id_rsa
```

![Acceso como root](/assets/images/keeper/13-ssh-root.png)

---

## 🚪 Acceso final como root

Con la clave convertida, accedemos por SSH:

```bash
ssh -i id_rsa root@10.10.11.227
```

![Flag de root](/assets/images/keeper/14-root-flag.png)

---

## 🏁 Flag final

Leemos la flag de root:

```bash
cat /root/root.txt
```



---

## 📊 Resumen

| Etapa           | Resultado                                      |
|-----------------|------------------------------------------------|
| Enumeración     | RT accesible con credenciales por defecto      |
| Explotación     | Usuario con contraseña inicial en comentarios  |
| Escalada I      | SSH como `lnorgaard`                           |
| Escalada II     | Dump de KeePass → acceso root con clave SSH    |
| Flags           | `user.txt` y `root.txt`                        |

---

**Autor:** [gueco99](https://github.com/gueco99)  
🧠 Hack The Box – *Keeper* (Easy)
