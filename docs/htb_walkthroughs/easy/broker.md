# HTB Walkthrough – Broker

**Fecha de resolución:** 17 de julio de 2025  
**Autor:** [gueco99](https://github.com/gueco99)

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

![Resultado de Nmap](../../assets/images/broker/01-nmap.png)
