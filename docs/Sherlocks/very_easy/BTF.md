---
title: BFT Sherlock Investigation
parent: Sherlock
nav_order: 1
---

# Walkthrough — BFT (Very Easy)
{: .fs-9 }

> A forensic walkthrough focused on NTFS Master File Table (MFT) analysis using industry-standard tools.
{: .fs-6 .fw-300 }

---

## 🧭 Scenario Overview

In this Sherlock, you will become acquainted with MFT (Master File Table) forensics. You will be introduced to well-known tools and methodologies for analyzing MFT artifacts to identify malicious activity. During our analysis, you will utilize the MFTECmd tool to parse the provided MFT file, TimeLine Explorer to open and analyze the results from the parsed MFT, and a Hex editor to recover file contents from the MFT.

### 🔧 Tools Used

- **MFTECmd** – for parsing the MFT file.
- **Timeline Explorer** – for timeline-based analysis.
- **HxD Hex Editor** – for raw inspection of MFT entries.

---

## Task 1: Simon Stark was targeted by attackers on February 13. He downloaded a ZIP file from a link received in an email. What was the name of the ZIP file he downloaded from the link?

Para resolver esto, abrimos el archivo `mft.csv` generado por `MFTECmd` usando Timeline Explorer.

Aplicamos dos filtros importantes:
- `Created0x10 = 2024-02-13`
- `Extension contains .zip`

Esto nos arroja tres resultados, pero uno de ellos es un archivo legítimo de recolección forense (`KAPE.zip`). El archivo relevante que fue descargado desde un enlace es:

✅ **Respuesta: `Stage-20240213T093324Z-001.zip`**

---

## Task 2: Examine the Zone Identifier contents for the initially downloaded ZIP file. What is the full Host URL from where this ZIP file was downloaded?

Muchos archivos descargados desde Internet tienen un stream alternativo llamado `Zone.Identifier`. Este stream contiene el campo `HostUrl`, que nos dice exactamente de dónde se descargó el archivo.

En Timeline Explorer buscamos archivos con el nombre `Stage` y extensión `.Identifier`. Esto revela el siguiente `HostUrl` incrustado:

```
https://storage.googleapis.com/drive-bulk-export-anonymous/20240213T093324.039Z/...
```

✅ **Respuesta:**
`https://storage.googleapis.com/drive-bulk-export-anonymous/20240213T093324.039Z/4133399871716478688/...`

---

## Task 3: What is the full path and name of the malicious file that executed malicious code and connected to a C2 server?

Usamos el mismo filtro por ruta `Stage` en Timeline Explorer y buscamos archivos `.bat`, típicamente usados para ejecutar scripts maliciosos.

Descubrimos el archivo:

✅ **Respuesta:**
`C:\Users\simon.stark\Downloads\Stage-20240213T093324Z-001\Stage\invoice\invoices\invoice.bat`

---

## Task 4: Analyze the $Created0x30 timestamp for the previously identified file. When was this file created on disk?

La columna `Created0x30` refleja cuándo fue creado el archivo en disco, independientemente del valor del sistema operativo.

Buscando la entrada de `invoice.bat` vemos:

✅ **Respuesta:**
`2024-02-13 16:38:39`

Este tiempo se puede incluir en una línea de tiempo forense más amplia.

---

## Task 5: Finding the hex offset of an MFT record is beneficial in many investigative scenarios. Find the hex offset of the stager file from Question 3.

Para encontrar el offset hexadecimal de un archivo en el MFT:

1. Tomamos el **Entry Number** del archivo: `23436`
2. Lo multiplicamos por 1024 (tamaño de cada entrada MFT):
   ```
   23436 × 1024 = 23998464
   ```
3. Convertimos el resultado a hexadecimal:
   ```
   23998464 = 0x16E3000
   ```

✅ **Respuesta: `0x16E3000`**

---

## Task 6: Find the contents of the malicious stager identified in Question 3 and answer with the C2 IP and port.

El archivo `invoice.bat` es muy pequeño (286 bytes), por lo tanto, es **residente en el MFT**, lo que significa que su contenido está dentro de la propia entrada MFT.

Pasos:

1. Abrimos el archivo `$MFT` en HxD.
2. Usamos la opción “Go to” (`Ctrl + G`) y saltamos al offset hexadecimal `0x16E3000`.
3. Al llegar allí, vemos contenido en texto plano del archivo `.bat` con un script PowerShell.

El script revela la IP y puerto del servidor C2:

```
http://43.204.110.203:6666/download
```

✅ **Respuesta: `43.204.110.203:6666`**

---

## ✅ Summary Table

| Task | Finding |
|------|---------|
| Task 1 | Stage-20240213T093324Z-001.zip |
| Task 2 | HostUrl: storage.googleapis.com/... |
| Task 3 | Full path: invoice.bat |
| Task 4 | Timestamp: 2024-02-13 16:38:39 |
| Task 5 | Offset: 0x16E3000 |
| Task 6 | C2 Address: 43.204.110.203:6666 |

---

**Autor:** [gueco99](https://github.com/gueco99)   
*Category: Windows Forensics — Very Easy*