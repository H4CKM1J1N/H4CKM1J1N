# 🛡️ ARTEX: Análisis Técnico y Validación de un Red Team Autónomo con IA Multi-Agente

![Status](https://img.shields.io/badge/Status-Completed-success)
![Environment](https://img.shields.io/badge/Environment-Isolated_Lab-blue)
![Engine](https://img.shields.io/badge/LLM-DeepSeek_V4_Pro-orange)
![Framework](https://img.shields.io/badge/Framework-ARTEX-red)

## 📌 Contexto y Motivación

Recientemente, agencias de seguridad internacionales (incluyendo el FBI) y múltiples firmas de Threat Intelligence emitieron alertas sobre el uso malicioso de **ARTEX**, una plataforma de pruebas de penetración autónoma impulsada por Inteligencia Artificial.

El objetivo de este repositorio es puramente **defensivo y educativo**. Documenta mi investigación en un laboratorio aislado para entender cómo operan estos agentes ofensivos, cómo razonan, cómo evaden defensas y cómo podemos preparar nuestras infraestructuras (Blue Team) para detectar y mitigar este nuevo paradigma de amenazas.

> **Disclaimer:** Toda la investigación, comandos y exploits documentados aquí fueron ejecutados en un entorno de laboratorio estrictamente controlado y autorizado. No se realizaron pruebas contra infraestructuras de producción ni sistemas de terceros.

---

## 🏗️ Arquitectura del Laboratorio

Para evitar el uso de imágenes Docker precompiladas de dudosa procedencia (algunas reportadas con backdoors), ARTEX fue compilado desde el código fuente auditado. 

El ecosistema se desplegó en una red Docker aislada (`artlab-net`), permitiendo la comunicación segura entre el contenedor de ARTEX y la máquina objetivo (`bola_container`).

### Configuración del "Cerebro" (LLM)
*   **Motor Principal:** Se configuró **DeepSeek V4 Pro** a través de una API compatible con OpenAI. DeepSeek destacó por su excelente equilibrio entre coste, velocidad y su capacidad de razonamiento lógico ofensivo (crucial para cadenas de ataque complejas).
*   **Orquestación Multi-Agente:** ARTEX no es un solo modelo; utiliza un enjambre de agentes. Un **Planner** (Estratega) descompone el objetivo, mientras que múltiples **Workers** (Ejecutores) realizan las pruebas en paralelo.

### Integraciones (MCP - Model Context Protocol)
*   **Browser MCP:** Se habilitó un navegador Chromium real (Playwright) para interactuar con aplicaciones web modernas y JavaScript pesado.
*   **ScopeSentry MCP:** Conectado para simular la sincronización de activos y el contexto de un cliente real.

### Conocimiento Inyectado (Skills)
Se cargaron módulos avanzados de la comunidad (ej. `api-security`, `llm-security`, `firmware-pentest`, `cloud-k8s`) que inyectan metodologías estandarizadas (PTES, OWASP, FSTM) directamente en el prompt de los agentes, evitando alucinaciones.

![Configuración del Goal Decomposer con DeepSeek](assets/img1_goal_decomposer.png)
*Fig 1. Configuración del motor DeepSeek V4 Pro como LLM global para la descomposición de objetivos.*

---

## ⚔️ El Caso de Estudio: Hacking Caja Negra (Black-Box)

El prompt de la tarea se diseñó siguiendo metodologías estrictas:
> *"Realiza una evaluación de seguridad de caja negra sobre el objetivo 172.21.0.3. Sigue la metodología PTES y OWASP. 1) Reconocimiento activo... 2) Mapeo web... 3) Identificación de vulnerabilidades... 4) EXPLOTACIÓN... 5) POST-EXPLOTACIÓN... 6) Documentación con evidencia."*

### 🚦 Reglas de Intercepción (Human-in-the-Loop)
Antes de iniciar, se configuraron estrictas `Interception Rules` para garantizar la seguridad del entorno. Si la IA intentaba ejecutar ataques de fuerza bruta (`ffuf`, `hydra`), inyecciones agresivas o escaladas con posibles efectos secundarios, la ejecución se pausaba automáticamente solicitando aprobación humana.

---

## 📊 La Cadena de Ataque Autónoma (Step-by-Step)

La IA demostró una capacidad de adaptación asombrosa, encadenando herramientas y razonando sobre los resultados.

### 1. Reconocimiento y Superficie de Ataque
El agente lanzó un escaneo inicial con `nmap`. Al recibir un bloqueo ICMP (ping), en lugar de fallar, **razonó** y aplicó el flag `-Pn` (tratando el host como activo). Descubrió el puerto `22` (SSH) y el `12345` (una API en Python/Werkzeug).

![Asset Coverage de ARTEX](assets/img2_asset_coverage.png)
*Fig 2. Mapeo automático de la superficie de ataque y cobertura de pruebas.*

### 2. Enumeración y Descubrimiento de Vulnerabilidades
Al inspeccionar la API, la IA notó una respuesta JSON con el mensaje `{"message": "User not found"}`. Razonó que esto era un fuerte indicio de una superficie de ataque **BOLA/IDOR** (Broken Object Level Authorization). Desplegó workers en paralelo:
*   Enumeración de directorios con `ffuf`.
*   Pruebas de inyección SQL/NoSQL.
*   Manipulación de IDs en la API (`/user/{id}`).

### 3. Explotación y Post-Explotación (Root)
*   **Hallazgo 1 & 2:** Confirmó el IDOR en `GET /user/{id}` (enumeración sin autenticación) y un fallo en `POST /login` (ignoraba la contraseña).
*   **Hallazgo 3:** Encontró un depurador de Werkzeug expuesto.
*   **Hallazgo 4 & 5:** Encontró credenciales débiles de SSH (`steven:steven`). Ya dentro del sistema, ejecutó `sudo -l` y descubrió una mala configuración: `sudo unzip`. La IA explotó esto de forma autónoma para escalar privilegios hasta obtener **`root`** (`uid=0`), demostrando el impacto con comandos mínimos y no destructivos (`id`, `whoami`).

![Resumen del Planner de ARTEX](assets/img3_planner_summary.png)
*Fig 3. Resumen final generado por el agente Planner detallando la cadena de ataque y la escalada a root.*

---

## 🧠 Visualizando la Mente de la IA: The Exploration Graph

Una de las características más fascinantes de ARTEX es su UI. El **Exploration Graph** genera un Grafo Acíclico Dirigido (DAG) que mapea visualmente todo el proceso de pensamiento del agente.

Cada nodo representa un comando ejecutado, una inferencia lógica o una vulnerabilidad confirmada. Al hacer clic en cada nodo, el operador humano puede auditar exactamente qué comando se lanzó, cuál fue la salida de la terminal y cómo la IA conectó ese hallazgo con el siguiente paso. Esta característica es invaluable para la fase de reporte y para replicar la prueba manualmente.

![Exploration Graph de ARTEX](assets/img4_exploration_graph.png)
*Fig 4. Grafo de exploración que mapea la máquina de estados de la IA durante toda la campaña.*

---

## 📝 Generación de Reportes y Auditoría

El agente `Report Writer` de ARTEX se encarga de consolidar la evidencia. Para llevar la documentación al siguiente nivel (y que sirva como material de estudio), se modificó su prompt inyectando la siguiente directiva:

> *"Incluye una sección 'Step-by-Step Attack Chain'. Para cada paso, indica la Fase (PTES), el Comando exacto ejecutado, la Salida/Resultado obtenido y el Razonamiento de la IA para dar el siguiente paso. No omitas los pasos intermedios."*

Esto permite generar informes técnicos que no solo muestran el "qué" (la vulnerabilidad), sino el "cómo" (la cadena de explotación exacta), facilitando la remediación y el estudio táctico.

---

## 🛡️ Conclusiones y Lecciones para Defensores (Blue Team)

1.  **Velocidad y Encadenamiento:** La IA no solo escanea; *razona*. Es capaz de encadenar un fallo de autenticación con la exposición de un depurador y una mala configuración de `sudo` en cuestión de minutos, algo que a un humano le tomaría horas.
2.  **Adaptabilidad a Defensas:** Bloqueos de ICMP o WAFs básicos no detienen a estos agentes; cambian de táctica automáticamente.
3.  **La Importancia de las Reglas:** El concepto de `Human-in-the-Loop` (aprobar comandos críticos) es vital. Sin estas reglas, un agente autónomo podría causar daños colaterales (DoS accidentales) en la infraestructura de un cliente.
4.  **Detección:** Las defensas ya no pueden basarse solo en firmas de CVEs conocidos. Debemos monitorizar patrones de comportamiento: ráfagas rápidas de escaneos lógicos, intentos secuenciales de IDOR y comandos de post-explotación automatizados.

**La IA ofensiva ya está aquí. Entender cómo piensa es el primer paso para detenerla.**

---
*Investigación realizada por [Tu Nombre/Usuario de GitHub] para fines de ciberseguridad defensiva.*
