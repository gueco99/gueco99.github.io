---
title: Lame
parent: Easy
nav_order: 1
---

# Walkthrough — Lame (Hack The Box)
{: .fs-9 }

> Primera máquina publicada en Hack The Box, ideal para iniciarse en el pentesting de sistemas Linux.
{: .fs-6 .fw-300 }

---

## Introducción

**Lame** es una máquina de nivel *Easy* en Hack The Box. El objetivo es identificar vulnerabilidades en servicios comunes y explotarlas para obtener acceso inicial y escalado de privilegios.

- SO: Linux
- Dificultad: Fácil
- Herramientas comunes: `nmap`, `msfconsole`, `smbclient`

---

## Enumeración

```bash
nmap -sC -sV -p- -T4 10.10.10.3
