# VLFisioterapia · Centro Integral de Fisioterapia y Rehabilitación

## 1. Identificación

- **Nombre del proyecto:** VLFisioterapia · Centro Integral de Fisioterapia y Rehabilitación (Armenia, Quindío — datos clínicos ficticios de práctica).
- **Curso y periodo:** Creación de Chatbots con Inteligencia Artificial (Electiva 2) · Periodo EAM 2026-II · Entrega Final (Tercer Corte).
- **Equipo (`Equipo_Sheen_castaneda_vargas`):**
  - **Victor Sheen Arias Zuleta** — Cabecilla / Arquitecto (`api_telegram.py`, `AgendaTelegram`, `almacen_sqlite.py`, transacciones con `BEGIN IMMEDIATE` y `SAVEPOINT`, integración de canales en `main.py`).
  - **Valentín Castañeda Marín** — Observadora / QA (`test_telegram.py`, `probar_fisioterapia.py`, ejecución de la matriz de 25 pruebas T01–T25, pruebas de concurrencia multihilo T22 y validación de evidencias).
  - **Samuel Rojas Vargas** — Mano Derecha / Backend Lead (`bot_telegram.py`, `simular_respuesta_perdida.py`, `consulta_llm.py`, `chatbot_memoria.py`, modelos Pydantic v2 y documentación de despliegue).
- **Descripción breve de la solución:** Asistente conversacional omnicanal para una clínica de fisioterapia que permite a los pacientes consultar tratamientos, tarifas en pesos colombianos (COP), horarios y disponibilidad real por especialista, así como agendar, consultar, reprogramar y cancelar citas médicas desde **Telegram** (canal principal de tercer corte) y **Gradio 6** (interfaz web del segundo corte). El modelo de lenguaje (Groq `openai/gpt-oss-120b`) únicamente interpreta la intención del usuario en formato JSON; toda operación de escritura exige confirmación explícita mediante botones interactivos, valida propiedad por cuenta de Telegram, garantiza idempotencia ante fallos de red y persiste de forma atómica en **SQLite** (`reservas.db`).

---

## 2. Problema y alcance

### Usuarios y necesidad que atiende
1. **Pacientes de la clínica:** personas que necesitan conocer los tratamientos de rehabilitación física, sus precios y duraciones, verificar cupos disponibles por fisioterapeuta y gestionar sus citas (crear, consultar, modificar o cancelar) las 24 horas desde Telegram expresándose en lenguaje natural a lo largo de varios mensajes.
2. **Personal clínico y fisioterapeutas (`mariana`, `carlos`, `valeria`):** requieren que la agenda respete los horarios de apertura de cada día, la duración exacta de cada terapia (45 o 60 minutos en pasos de 30 minutos) y las especialidades asignadas a cada profesional, impidiendo cruces de horario (*double booking*) o modificaciones no autorizadas de reservas ajenas.

### Funciones implementadas
- **Consulta del catálogo clínico (`clinica.json`):** informa los 6 tratamientos disponibles (`evaluacion`, `deportiva`, `traumatologica`, `puncion_seca`, `postural`, `neurologica`) con su descripción, duración (45 o 60 min) y tarifa en COP (`$60.000` a `$85.000 COP`), además de horarios de atención, ubicación y convenios.
- **Consulta de disponibilidad real por especialista:** calcula los bloques horarios libres en pasos de 30 minutos para una fecha y tratamiento dados, filtrando por fisioterapeuta específico o unificando los cupos de los profesionales habilitados.
- **Memoria conversacional multi-turno y aislamiento por cuenta:** acumula los datos entregados en varios turnos (`cliente`, `servicio`, `fecha`, `hora`, `codigo`, `fisioterapeuta`) manteniendo historiales y propuestas pendientes completamente separados para cada usuario/chat de Telegram.
- **Resumen previo y confirmación explícita con botones inline:** antes de crear, modificar o cancelar una cita, el bot presenta un resumen con referencia única (`solicitud_id`), vigencia de 10 minutos (TTL) y botones **Confirmar** y **Descartar**.
- **Ciclo completo de reservas con propiedad e idempotencia:**
  - **Crear:** registra la cita en la tabla `reservas` con código `VF-...`, asocia el `usuario_id` de Telegram en `propietarios_telegram` y almacena el recibo en `operaciones_telegram` dentro de la misma transacción.
  - **Consultar (`/reserva CODIGO` o lenguaje natural):** devuelve los detalles de la cita solo si pertenece al usuario de Telegram solicitante (rechazando reservas ajenas o históricas de otros canales con `HTTP 404`).
  - **Modificar / Reprogramar:** verifica que la reserva pertenezca al usuario y que su huella SHA-256 (`referencia`) no haya cambiado mientras se revisaba el resumen (bloqueo optimista; si cambió o el nuevo horario se cruza, responde `HTTP 409` conservando los datos previos).
  - **Cancelar:** aplica borrado lógico (`estado = 'cancelada'`), conserva la fila para auditoría y libera inmediatamente el bloque horario.
- **Recuperación ante respuesta perdida (`/estado` y `/reintentar`):** si ocurre un corte de red tras confirmar una escritura, el bot marca la operación como `incierta` conservando su `solicitud_id` y permite recuperar el recibo desde SQLite sin duplicar la reserva.

### Límites del servicio
- **No emite diagnósticos médicos ni prescribe medicamentos:** ante consultas farmacológicas o síntomas que requieran diagnóstico clínico, advierte sus límites y deriva a valoración presencial o recepción humana (`+57 300 987 6543` / WhatsApp `+57 311 234 5678`).
- **No atiende urgencias médicas hospitalarias vitales:** casos como fracturas abiertas o dolor agudo severo se remiten a urgencias hospitalarias.
- **Consultas fuera de dominio o intentos de inyección:** preguntas ajenas a la clínica o solicitudes de revelar claves/reservas de otros usuarios son rechazadas sin ejecutar acciones ni exponer secretos.

---

## 3. Arquitectura

### Recorrido de una solicitud de principio a fin

```text
+-------------------------------------------------------------------------+
| 1. TELEGRAM MESSENGER (Cuenta A / Cuenta B)                             |
|    • Envía mensajes de texto, comandos (/start, /reserva, /estado...)   |
|    • Pulsa botones inline [Confirmar] / [Descartar]                     |
+------------------------------------+------------------------------------+
                                     | Long Polling (HTTPS)
                                     v
+-------------------------------------------------------------------------+
| 2. ADAPTADOR DEL BOT — bot_telegram.py                                  |
|    • Aísla historial y propuesta pendiente en context.chat_data         |
|    • Envía historial + mensaje a Groq (interpretar -> JSON de 7 claves) |
|    • Si es escritura: genera resumen + solicitud_id (TTL 10 min)        |
|    • ClienteAPI llama a FastAPI con X-API-Key y X-Telegram-User         |
+-----------------+----------------------------------+--------------------+
                  | HTTPS                            | HTTP REST (/telegram/*)
                  v                                  v
+------------------------------------+   +--------------------------------+
| 3. MODELO LLM — Groq API           |   | 4. API DEL NEGOCIO — main.py   |
|    (openai/gpt-oss-120b)           |   |    + Router api_telegram.py    |
|    • Solo interpreta intención y   |   |    (FastAPI, puerto 8012)      |
|      extrae campos en JSON         |   |    • Autentica X-API-Key       |
|    • Nunca escribe en la BD ni     |   |    • Autoriza X-Telegram-User  |
|      inventa disponibilidad        |   |    • Valida con Pydantic v2    |
+------------------------------------+   +---------------+----------------+
                                                         |
                                                         v
+-------------------------------------------------------------------------+
| 5. PERSISTENCIA RELACIONAL — almacen_sqlite.py + api_telegram.py        |
|    Base SQLite: reservas.db | Catálogo: clinica.json                    |
|    • Transacciones BEGIN IMMEDIATE + SAVEPOINT cambio_reserva           |
|    • Tablas: reservas (16 col.), propietarios_telegram,                 |
|      operaciones_telegram (recibos idempotentes + huella SHA-256)       |
+-------------------------------------------------------------------------+
```

### Responsabilidad de cada componente
- **Telegram:** canal de mensajería que entrega al bot la identidad verificada del usuario (`update.effective_user.id`) y los eventos de mensajes o pulsaciones de botones inline (`callback_query`).
- **Python (`bot_telegram.py`, `consulta_llm.py`, `chatbot_memoria.py`):** gestiona el historial multi-turno por conversación, valida qué datos faltan antes de armar una propuesta, controla la vigencia de los botones (10 minutos), maneja estados inciertos ante errores de red y actúa como cliente HTTP autenticado frente a FastAPI. *(El repositorio conserva también [`app_gradio.py`](app_gradio.py) como canal web del segundo corte).*
- **Modelo de lenguaje (Groq `openai/gpt-oss-120b`):** traduce lenguaje natural e historial a un objeto JSON estructurado (`accion`, `cliente`, `servicio`, `fecha`, `hora`, `codigo`, `fisioterapeuta`). No tiene acceso directo a SQLite ni autoridad para confirmar operaciones.
- **FastAPI (`main.py` y `api_telegram.py`):** única autoridad de negocio en el puerto `8012`. Expone las rutas generales (`/servicios`, `/disponibilidad`, `/citas`) y el sub-router protegido `/telegram` (`/telegram/servicios`, `/telegram/disponibilidad`, `/telegram/reservas/{codigo}`, `/telegram/operaciones`, `/telegram/operaciones/{solicitud_id}`), verificando la clave compartida `X-API-Key` en tiempo constante (`secrets.compare_digest`).
- **SQLite (`almacen_sqlite.py`, `api_telegram.py` y `reservas.db`):** garantiza persistencia en disco, control de concurrencia mediante `BEGIN IMMEDIATE`, atomicidad con `SAVEPOINT cambio_reserva`, verificación de cruces de horario por fisioterapeuta, control de propiedad en `propietarios_telegram` e idempotencia en `operaciones_telegram`.

---

## 4. Requisitos

- **Versión de Python:** verificado con **Python `3.13.9`** (compatible con **Python `3.11` o superior**).
- **Dependencias de Python:** definidas en un único archivo [`requirements.txt`](requirements.txt):
  - `fastapi==0.115.12`
  - `uvicorn==0.34.2`
  - `httpx==0.28.1`
  - `tzdata>=2025.2`
  - `pydantic>=2.10.0`
  - `gradio>=6,<7`
  - `python-telegram-bot>=22,<23`
  - `pysqlite3-binary>=0.5.4; sys_platform == 'linux'` *(respaldo binario automático en entornos Linux cuyo Python haya sido compilado sin cabeceras C de `_sqlite3`)*.
- **Servicios externos necesarios:**
  1. **Groq Cloud API (`https://api.groq.com`):** requiere una clave `GROQ_API_KEY` con acceso al modelo `openai/gpt-oss-120b`.
  2. **Telegram Bot API:** requiere un token `TELEGRAM_BOT_TOKEN` generado en `@BotFather` y conexión a internet para *long polling*.

---

## 5. Preparación

Todos los comandos se ejecutan desde la **carpeta raíz del repositorio** (`Chatbot_VLFisioterapia_Arias_Castaneda_Rojas`).

### En Linux / macOS (bash)

```bash
# 1. Clonar el repositorio y entrar a la carpeta raíz
git clone https://github.com/Victor-Sheen-U/Chatbot_VLFisioterapia_Arias_Castaneda_Rojas.git
cd Chatbot_VLFisioterapia_Arias_Castaneda_Rojas

# 2. Crear y activar el entorno virtual
python3 -m venv .venv
source .venv/bin/activate

# 3. Instalar las dependencias
pip install --upgrade pip
pip install -r requirements.txt
```

### En Windows (PowerShell)

```powershell
# 1. Clonar el repositorio y entrar a la carpeta raíz
git clone https://github.com/Victor-Sheen-U/Chatbot_VLFisioterapia_Arias_Castaneda_Rojas.git
cd Chatbot_VLFisioterapia_Arias_Castaneda_Rojas

# 2. Crear y activar el entorno virtual
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 3. Instalar las dependencias
python -m pip install --upgrade pip
pip install -r requirements.txt
```

---

## 6. Configuración

El proyecto lee las variables de entorno desde el sistema o desde un archivo `.env` ubicado en la raíz del proyecto (cargado automáticamente por `cargar_env()` en [`consulta_llm.py`](consulta_llm.py), [`main.py`](main.py) y [`bot_telegram.py`](bot_telegram.py)).

> **Importante:** Las claves reales nunca se suben al repositorio. El archivo [`.gitignore`](.gitignore) excluye `.env`, entornos virtuales, cachés, respaldos y archivos `.log`.

### Tabla de variables de entorno

| Variable | Para qué sirve | Proceso donde se configura y utiliza |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | Token secreto entregado por `@BotFather` para conectar el bot con Telegram mediante *long polling*. | **Bot de Telegram** (`bot_telegram.py` y `simular_respuesta_perdida.py`) |
| `CHATBOT_API_KEY` | Clave privada compartida (mínimo 16 caracteres ASCII) que autentica las peticiones HTTP entre el bot (`X-API-Key`) y el router `/telegram`. Si no se define en FastAPI, `/telegram` responde `HTTP 503` por seguridad. | **FastAPI** (`main.py` / `api_telegram.py`) y **Bot de Telegram** (`bot_telegram.py`) |
| `CHATBOT_API_URL` | URL base donde escucha FastAPI (por defecto `http://127.0.0.1:8012`). | **Bot de Telegram** (`bot_telegram.py`) y **Gradio / Cliente LLM** (`consulta_llm.py`, `app_gradio.py`) |
| `GROQ_API_KEY` | Clave de acceso a la API de Groq para interpretar los mensajes del usuario con el LLM. | **Bot de Telegram** (`bot_telegram.py`) y **Gradio / Cliente LLM** (`app_gradio.py`, `consulta_llm.py`) |
| `GROQ_MODEL` | Identificador del modelo de lenguaje en Groq (por defecto `openai/gpt-oss-120b`). | **Bot de Telegram** (`bot_telegram.py`) y **Gradio / Cliente LLM** (`app_gradio.py`, `consulta_llm.py`) |
| `HOST` | Dirección IP local de escucha del servidor Uvicorn (por defecto `127.0.0.1`). | **FastAPI** (`main.py`) |
| `PORT` | Puerto TCP donde escucha el servidor FastAPI (por defecto `8012`). | **FastAPI** (`main.py`) |

### Crear el archivo `.env`

1. Copia la plantilla incluida en el repositorio ([`.env.example`](.env.example)):

   **Linux / macOS (bash):**
   ```bash
   cp .env.example .env
   chmod 600 .env
   ```

   **Windows (PowerShell):**
   ```powershell
   Copy-Item .env.example .env
   ```

2. Genera una clave privada de al menos 16 caracteres ASCII para `CHATBOT_API_KEY` (por ejemplo con `python -c "import secrets; print(secrets.token_urlsafe(32))"`) y edita `.env` completando `TELEGRAM_BOT_TOKEN`, `CHATBOT_API_KEY` y `GROQ_API_KEY` con el mismo valor de `CHATBOT_API_KEY` para ambos procesos.

---

## 7. Arranque

Todos los procesos deben ejecutarse ubicados en la **carpeta raíz** (`Chatbot_VLFisioterapia_Arias_Castaneda_Rojas`) y respetando el orden: **primero FastAPI (Terminal 1)** y **luego el Bot de Telegram (Terminal 2)**.

> **Regla de instancia única:** Mantén **un solo proceso** de `bot_telegram.py` (o `simular_respuesta_perdida.py`) en ejecución para el mismo `TELEGRAM_BOT_TOKEN`. Dos procesos simultáneos de *polling* provocarán `Conflict: terminated by other getUpdates request`.

### Terminal 1 — Iniciar FastAPI (Puerto 8012)

**Linux / macOS (bash):**
```bash
source .venv/bin/activate
python -m uvicorn main:app --host 127.0.0.1 --port 8012
```

**Windows (PowerShell):**
```powershell
.\.venv\Scripts\Activate.ps1
python -m uvicorn main:app --host 127.0.0.1 --port 8012
```

**Direcciones que deben comprobarse con FastAPI encendida:**
- Catálogo JSON de servicios: `http://127.0.0.1:8012/servicios`
- Documentación interactiva Swagger UI (incluye rutas `/telegram`): `http://127.0.0.1:8012/docs`

### Terminal 2 — Iniciar el Bot de Telegram (Long Polling)

Con la Terminal 1 activa, abre una segunda terminal en la carpeta raíz:

**Linux / macOS (bash):**
```bash
source .venv/bin/activate
python bot_telegram.py
```

**Windows (PowerShell):**
```powershell
.\.venv\Scripts\Activate.ps1
python bot_telegram.py
```

Verás en consola:
```text
Bot de VLFisioterapia en ejecución por polling. Conectado a http://127.0.0.1:8012. Para detenerlo: Ctrl+C.
```

### Terminal 3 (Opcional) — Interfaz Web Gradio 6 del Segundo Corte (Puerto 7860)

Si deseas abrir también la interfaz web del segundo corte:

```bash
source .venv/bin/activate
python app_gradio.py
```
- Dirección web local: `http://127.0.0.1:7860`

### Verificación con las suites de pruebas automatizadas

Las suites [`probar_fisioterapia.py`](probar_fisioterapia.py) y [`test_telegram.py`](test_telegram.py) utilizan `TestClient` y una base SQLite temporal (no alteran `reservas.db` ni requieren servidor externo encendido):

```bash
source .venv/bin/activate
python probar_fisioterapia.py
python test_telegram.py
```

---

## 8. Uso del chatbot

### Enlace público del bot en Telegram

- **Usuario del bot:** `@HuesitosXBot` (*Huesitos_bot*)
- **Enlace directo:** [https://t.me/HuesitosXBot](https://t.me/HuesitosXBot)

> **Aviso de operación local:** El enlace público permite abrir el chat del bot en Telegram, pero **el bot solo responderá mientras los procesos locales de FastAPI (Terminal 1) y `bot_telegram.py` (Terminal 2) estén en ejecución**.

### Comandos disponibles en Telegram

| Comando | Función |
|---|---|
| `/start` | Muestra el mensaje de bienvenida e introducción de VLFisioterapia. |
| `/help` | Lista los comandos disponibles, ejemplos de solicitudes y nota de confirmación. |
| `/reserva CODIGO` | Consulta en tiempo real el estado y detalles de una reserva creada desde la misma cuenta de Telegram (ej.: `/reserva VF-1A2B3C4D5E`). |
| `/estado [REFERENCIA]` | Recupera el recibo persistido de la última operación (o de un `solicitud_id` de 32 caracteres hexadecimales) cuando hubo pérdida de respuesta de red. |
| `/reintentar` | Reenvía de forma idempotente una operación en estado `incierta` conservando su mismo `solicitud_id`. |
| `/descartar` | Retira la propuesta pendiente actual sin ejecutar ningún cambio en SQLite. |
| `/reset` | Limpia el historial conversacional del chat actual sin borrar las reservas guardadas en `reservas.db`. |

### Ejemplos de mensajes para probar las funciones principales

1. **Consultar servicios y tarifas:**
   - `"¿Qué servicios ofrecen?"`
   - `"¿Cuánto cuesta la fisioterapia deportiva y cuánto dura?"`
2. **Consultar horarios de atención o disponibilidad:**
   - `"¿Cuál es el horario de atención?"`
   - `"¿Qué horarios hay con Mariana el 2026-11-23?"`
3. **Agendar una cita (en uno o varios turnos):**
   - Turno 1: `"Hola, me llamo Laura Gómez y quiero una cita de fisioterapia deportiva"`
   - Turno 2: `"Para el 2026-11-24 a las 11:00"`
   - Acción: Revisar el resumen y pulsar el botón inline **Confirmar** (recibirás tu código `VF-...`).
4. **Consultar una reserva propia:**
   - `"/reserva VF-XXXXXXXXXX"` *(usando el código devuelto al confirmar)*.
5. **Modificar / reprogramar una reserva propia:**
   - `"Quiero reprogramar mi cita VF-XXXXXXXXXX para el 2026-11-24 a las 15:00"` → pulsar **Confirmar**.
6. **Cancelar una reserva propia:**
   - `"Quiero cancelar mi cita VF-XXXXXXXXXX"` → pulsar **Confirmar**.

---

## 9. Datos y navegación del repositorio

### Catálogo del negocio y base de datos SQLite

- **Catálogo del negocio ([`clinica.json`](clinica.json)):** archivo JSON en la raíz del proyecto que define los 6 tratamientos fisioterapéuticos (`evaluacion`, `deportiva`, `traumatologica`, `puncion_seca`, `postural`, `neurologica`), los 3 fisioterapeutas (`mariana`, `carlos`, `valeria`) con sus especialidades, horarios de lunes a viernes (`07:00–19:00`) y sábados (`08:00–14:00`), paso de agenda de 30 minutos, zona horaria `America/Bogota`, preguntas frecuentes y datos de derivación humana.
- **Base de datos SQLite ([`reservas.db`](reservas.db)):**
  - **Ubicación y creación:** se ubica en la raíz (`./reservas.db`). Si el archivo no existe al iniciar `main.py` o `preparar_datos_prueba.py`, `AgendaTelegram` (`api_telegram.py`) y `AgendaSQLite` (`almacen_sqlite.py`) la crean automáticamente junto con las tres tablas: `reservas` (16 columnas), `propietarios_telegram` y `operaciones_telegram`.
  - **Datos de práctica incluidos:** el repositorio incluye `reservas.db` pre-cargada con **5 reservas de práctica** (`Paciente Demo 1` a `Paciente Demo 5`) en días hábiles futuros (4 activas y 1 cancelada, incluyendo citas simultáneas con distintas fisioterapeutas a las `09:00`) y con las tablas `propietarios_telegram` y `operaciones_telegram` limpias.
  - **Cómo restaurar los datos de práctica iniciales en cualquier momento:**
    ```bash
    python preparar_datos_prueba.py --reiniciar
    ```

### Navegación a los documentos y entregables

- **Documentación principal (Bitácora Consolidada):** [`documentacion/Bitacora_Consolidada.md`](documentacion/Bitacora_Consolidada.md)
- **Matriz de 25 pruebas y resultados:** [`documentacion/Matriz_25_Pruebas.md`](documentacion/Matriz_25_Pruebas.md)
- **Evidencias de pruebas (T01–T25):** [`evidencias/`](evidencias/)
- **Presentación final del proyecto (PDF):** [`presentacion/Presentacion_Final.pdf`](presentacion/Presentacion_Final.pdf)
- **Histórico de cortes anteriores:**
  - Bitácora del Segundo Corte: [`documentacion/historico/Bitacora_Segundo_Corte.md`](documentacion/historico/Bitacora_Segundo_Corte.md)
  - Presentación del Segundo Corte: [`documentacion/historico/Presentacion_Segundo_Corte.pdf`](documentacion/historico/Presentacion_Segundo_Corte.pdf)
  - Bitácoras previas por sesión (C01 a C05): [`documentacion/historico/bitacoras_previas/`](documentacion/historico/bitacoras_previas/)
  - Evidencias del Segundo Corte: [`evidencias/segundo_corte/`](evidencias/segundo_corte/)
