---
title: Blue
parent: Medium
nav_order: 1
---

# Walkthrough — Blue (Hack The Box)
{: .fs-9 }

Máquina de nivel **Medium** basada en Windows, conocida por su vulnerabilidad al exploit **EternalBlue** (MS17-010), una de las más famosas de los últimos años. hola
{: .fs-6 .fw-300 }

---

## Introducción

- Sistema: Windows 7
- Dificultad: Media
- IP típica: `10.10.10.40`
- Vulnerabilidad principal: **EternalBlue / MS17-010**
- Herramientas usadas: `nmap`, `msfconsole`, `smbclient`

---

## Enumeración

```bash
nmap -sC -sV -p- -T4 10.10.10.40
