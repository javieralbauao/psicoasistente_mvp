# PsicoAsistente MVP

Aplicación web full-stack para orientación psicoeducativa general con:
- **Autenticación JWT** (registro/login)
- **Chat con historial por usuario**
- **Detección básica de crisis/intención** (reglas)
- **Síntesis de voz opcional** con ElevenLabs (MP3)

> ⚠️ **Disclaimer clínico obligatorio**: Esta aplicación **no** sustituye terapia profesional, diagnóstico ni atención de emergencia.

---

## Tabla de contenidos
1. [Propósito y alcance](#propósito-y-alcance)
2. [Características principales](#características-principales)
3. [Arquitectura del proyecto](#arquitectura-del-proyecto)
4. [Stack tecnológico](#stack-tecnológico)
5. [Requisitos](#requisitos)
6. [Instalación y configuración](#instalación-y-configuración)
7. [Ejecución local](#ejecución-local)
8. [Uso de la interfaz web](#uso-de-la-interfaz-web)
9. [API y ejemplos de uso](#api-y-ejemplos-de-uso)
10. [Variables de entorno](#variables-de-entorno)
11. [Seguridad implementada](#seguridad-implementada)
12. [Limitaciones del MVP](#limitaciones-del-mvp)
13. [Guía para producción (pendiente/recomendado)](#guía-para-producción-pendienterecomendado)
14. [Troubleshooting](#troubleshooting)
15. [Estructura de carpetas](#estructura-de-carpetas)
16. [Roadmap sugerido](#roadmap-sugerido)

---

## Propósito y alcance
**PsicoAsistente MVP** busca ofrecer una experiencia de apoyo conversacional general (psicoeducación), con foco en:
- interacción segura básica,
- continuidad de conversación por historial,
- y opción de respuesta en audio.

No es un sistema clínico, no reemplaza profesionales de salud mental y no realiza diagnóstico.

---

## Características principales
- Registro de usuario (`/auth/register`)
- Inicio de sesión (`/auth/login`)
- Validación de sesión (`/auth/me`)
- Chat protegido por JWT (`/chat/message`)
- Historial por usuario (`/chat/history`)
- Detección de crisis por palabras clave y `safety_flag="CRISIS"`
- Clasificación simple de intención (`ansiedad`, `tristeza`, `soledad`, etc.)
- TTS opcional con ElevenLabs (`/tts` y audio en `/media/...`)
- Frontend estático servido por FastAPI (`/` y `/chat`)

---

## Arquitectura del proyecto
Patrón **monolito en capas**:

- **Routers (HTTP)**: reciben solicitudes, validan payloads, aplican auth y orquestan flujo.
- **Services (dominio)**:
  - `assistant_engine.py`: lógica de respuesta segura (reglas)
  - `tts_service.py`: integración con ElevenLabs
- **Persistencia**: SQLAlchemy (`User`, `ChatMessage`)
- **Seguridad**: hash de contraseñas + JWT Bearer
- **Frontend**: HTML/CSS/JS consumiendo API REST

Flujo resumido:
1. Usuario se registra/inicia sesión.
2. Frontend guarda JWT en `localStorage`.
3. Usuario envía mensaje.
4. Backend guarda mensaje, genera respuesta, intenta TTS.
5. Devuelve texto + metadata (`intent`, `safety_flag`, `audio_url`/`audio_error`).

---

## Stack tecnológico
- **Backend**: FastAPI, Uvicorn, SQLAlchemy, Pydantic v2
- **Auth/Security**: python-jose (JWT), passlib+bcrypt
- **HTTP externo**: requests (ElevenLabs)
- **Frontend**: HTML, CSS, JavaScript vanilla
- **DB por defecto**: SQLite (`sqlite:///./app.db`)

Dependencias exactas en [`requirements.txt`](./requirements.txt).

---

## Requisitos
- Python **3.11+** (recomendado 3.11 o 3.12)
- pip
- (Opcional) API key de ElevenLabs para audio real

---

## Instalación y configuración
### 1) Clonar/abrir proyecto
```bash
cd /ruta/a/psicoasistente_mvp
```

### 2) Crear entorno virtual
```bash
python -m venv .venv
```

### 3) Activar entorno virtual
- Linux/macOS:
```bash
source .venv/bin/activate
```
- Windows PowerShell:
```powershell
.\.venv\Scripts\Activate.ps1
```

### 4) Instalar dependencias
```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### 5) Crear `.env`
```bash
cp .env.example .env
```
> En Windows PowerShell: `Copy-Item .\.env.example .\.env`

Edita `.env` y completa al menos:
- `SECRET_KEY` (obligatoria, fuerte y aleatoria)
- `DATABASE_URL` (SQLite por defecto)
- `ELEVENLABS_API_KEY` (opcional, necesaria solo para audio real)

---

## Ejecución local
### Opción A (directa con uvicorn)
```bash
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

### Opción B (scripts Windows)
```powershell
.\scripts\run_backend.ps1
```

Una vez iniciado:
- UI login/registro: http://127.0.0.1:8000/
- UI chat: http://127.0.0.1:8000/chat
- Health: http://127.0.0.1:8000/health

---

## Uso de la interfaz web
1. Entra a `/`.
2. Crea cuenta o inicia sesión.
3. Serás redirigido a `/chat`.
4. Envía un mensaje.
5. Verás respuesta de texto, intent/safety y posible audio.

Notas:
- Si no hay token válido, `/chat` redirige a `/`.
- Si ElevenLabs no está configurado, el chat sigue funcionando en texto.

---

## API y ejemplos de uso
Base URL local: `http://127.0.0.1:8000`

### `GET /health`
Verifica estado general y si TTS está configurado.

```bash
curl http://127.0.0.1:8000/health
```

### `POST /auth/register`
Crea usuario y devuelve JWT.

```bash
curl -X POST http://127.0.0.1:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"demo@example.com","password":"ClaveSegura123!","full_name":"Demo"}'
```

### `POST /auth/login`
Inicia sesión y devuelve JWT.

```bash
curl -X POST http://127.0.0.1:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"demo@example.com","password":"ClaveSegura123!"}'
```

### `GET /auth/me` (protegido)
```bash
curl http://127.0.0.1:8000/auth/me \
  -H "Authorization: Bearer <TOKEN>"
```

### `GET /chat/history` (protegido)
```bash
curl http://127.0.0.1:8000/chat/history \
  -H "Authorization: Bearer <TOKEN>"
```

### `POST /chat/message` (protegido)
```bash
curl -X POST http://127.0.0.1:8000/chat/message \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"message":"Me siento muy ansioso hoy"}'
```

### `POST /tts` (protegido)
```bash
curl -X POST http://127.0.0.1:8000/tts \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"text":"Hola, esta es una prueba"}'
```

---

## Variables de entorno
Ver archivo [`.env.example`](./.env.example).

- `SECRET_KEY` (**obligatoria**): clave JWT
- `DATABASE_URL`: por defecto `sqlite:///./app.db`
- `ELEVENLABS_API_KEY`: habilita TTS real
- `ELEVENLABS_VOICE_ID`: voz de ElevenLabs
- `ELEVENLABS_MODEL_ID`: modelo de ElevenLabs
- `ELEVENLABS_BASE_URL`: endpoint base del proveedor

---

## Seguridad implementada
### Ya incluido
- Hash de contraseña con `bcrypt_sha256`
- JWT con expiración
- Endpoints críticos protegidos por Bearer
- Anti-enumeración básica en login (mensaje genérico)
- Validación de payloads con Pydantic
- Escape HTML en render de mensajes en frontend
- Disclaimer visible en UI y respuestas

### Importante (aún pendiente para producción)
- Restringir CORS por entorno (no `*`)
- Rate limiting / protección anti brute-force
- Migraciones (Alembic) en lugar de `create_all`
- Logging estructurado + observabilidad
- Rotación de secretos y políticas de retención de datos

---

## Limitaciones del MVP
- Motor conversacional **rule-based** (sin LLM activo)
- Sin suite formal de tests automatizados
- Sin pipeline CI/CD
- Sin panel de administración
- Sin controles avanzados de seguridad operacional

---

## Guía para producción (pendiente/recomendado)
Checklist mínimo antes de exponer a internet:
- [ ] CORS restringido por dominio
- [ ] TLS/HTTPS y proxy inverso
- [ ] Rate limiting por IP/usuario
- [ ] Alembic + estrategia de migraciones
- [ ] Monitoreo/alertas y logs centralizados
- [ ] Backups de DB y plan de recuperación
- [ ] Auditoría de manejo de datos sensibles
- [ ] Términos de uso, privacidad y compliance aplicable

---

## Troubleshooting
### 1) `401 No autenticado`
- Verifica header `Authorization: Bearer <TOKEN>`.
- Inicia sesión nuevamente si expiró token.

### 2) `ELEVENLABS_API_KEY no configurada`
- Define `ELEVENLABS_API_KEY` en `.env`.
- Reinicia el backend.

### 3) No se reproduce audio automático
- Algunos navegadores bloquean autoplay.
- Reintenta reproducción manual.

### 4) Error de CORS en frontend externo
- Actualmente el backend permite `*`, pero si lo restringes deberás incluir explícitamente el origen de tu frontend.

### 5) `Frontend no encontrado (index.html/chat.html)`
- Verifica que exista carpeta `frontend/` y ambos archivos HTML.

---

## Estructura de carpetas
```text
psicoasistente_mvp/
├─ app/
│  ├─ main.py
│  ├─ config.py
│  ├─ database.py
│  ├─ models.py
│  ├─ schemas.py
│  ├─ security.py
│  ├─ routers/
│  │  ├─ auth.py
│  │  ├─ chat.py
│  │  └─ tts.py
│  └─ services/
│     ├─ assistant_engine.py
│     └─ tts_service.py
├─ frontend/
│  ├─ index.html
│  ├─ chat.html
│  ├─ styles.css
│  └─ app.js
├─ scripts/
│  ├─ run_backend.ps1
│  ├─ run_backend.cmd
│  └─ make_zip.ps1
├─ .env.example
├─ requirements.txt
└─ README.md
```

---

## Roadmap sugerido
- Añadir pruebas unitarias/integración para auth/chat/tts
- Incorporar Alembic para versionado de schema
- Externalizar motor de intención (provider-agnostic)
- Agregar rate limiting y hardening de autenticación
- Preparar despliegue Docker + CI/CD

---

## Licencia
Actualmente este repositorio no declara licencia explícita.
Si vas a reutilizarlo públicamente, agrega una licencia (`MIT`, `Apache-2.0`, etc.).
