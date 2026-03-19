# 🖥️ [NOMBRE DE LA MÁQUINA]

```bash
┌──(Harou㉿kali)-[~/writeups/nombre-maquina]
└─$ cat info.txt

  Plataforma  : The Hackers Labs
  Dificultad  : 🟢 Fácil
  Sistema     : Linux / Windows
  Fecha       : DD/MM/AAAA
  Estado      : ✅ Completada
```

---

## 🎯 `cat objetivo.txt`

> Breve descripción de la máquina y qué había que conseguir.

---

## 🔍 `./reconnaissance.sh`

### Escaneo de puertos

```bash
nmap -sV -sC -oN scan.txt <IP>
```

**Resultado:**
```
# Pega aquí el output del nmap
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22 | SSH | OpenSSH X.X |
| 80 | HTTP | Apache X.X |

---

## 🕵️ `./enumeration.sh`

> Describe qué encontraste al explorar los servicios. 
> ¿Qué había en el puerto 80? ¿Directorios ocultos? ¿Usuarios?

```bash
# Comandos que usaste
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
```

---

## ⚡ `./exploitation.sh`

> ¿Cómo conseguiste acceso inicial? Explica el vector de ataque.

```bash
# Comandos clave del exploit
```

**¿Qué vulnerabilidad explotaste?**
```
# Describe la vulnerabilidad con tus palabras
```

---

## 🚀 `./privesc.sh`

> ¿Cómo escalaste privilegios hasta root/admin?

```bash
# Comandos de escalada de privilegios
```

---

## 🏁 `cat flags.txt`

```
User flag  : ****************************
Root flag  : ****************************
```

---

## 🧠 `cat lecciones.txt`

> Esta sección es la más importante — ¿qué aprendiste?

- 💡 Lección 1: ...
- 💡 Lección 2: ...
- 💡 Lección 3: ...

---

## 🛠️ `cat herramientas.txt`

```
nmap · gobuster · burpsuite · ...
```

---

<div align="center">

*`📚 Writeup educativo — The Hackers Labs`*

</div>
