# FlakAI v2

> Plataforma de anotación de vídeo con inteligencia artificial para análisis de partidos de fútbol.

---

## ¿Qué es FlakAI?

FlakAI v2 es una plataforma web completa que automatiza la extracción y etiquetado de eventos en grabaciones de partidos de fútbol. Sube un vídeo, la IA detecta automáticamente eventos relevantes (goles, córners, faltas, tiros a puerta y saques de banda.), extrae clips centrados en cada evento y los distribuye a equipos humanos para su revisión. Los datos validados sirven como dataset de entrenamiento para modelos de visión por computador.

---

## Características principales

- **Detección automática de eventos** — Worker de IA en segundo plano analiza cada vídeo y detecta: goles, córners, saques de banda, faltas, saques de puerta y disparos a puerta.
- **Extracción de clips con FFmpeg** — Genera clips de ±5 segundos alrededor de cada evento, con extracción paralela (hasta 4 clips simultáneos por vídeo).
- **Lotes de etiquetado** — Los clips se distribuyen automáticamente en lotes (batches) entre revisores mediante round-robin.
- **Revisión en tiempo real** — Interfaz web para aprobar o rechazar clips con teclado (`A` / `R` / `←` / `→`).
- **Multi-tenant con roles** — Cada equipo gestiona sus propios vídeos y revisores. Aprobación de nuevos equipos por administrador.
- **Facturación con Stripe** — Planes `free_trial`, `pro` y `club` con límites de subida. Portal de cliente integrado.
- **Exportación de datasets** — Exporta clips revisados en formato JSONL/CSV listos para entrenar modelos.
- **Auto-ingesta por carpeta** — Watcher en segundo plano que detecta nuevos vídeos en una carpeta local y los ingesta automáticamente.

---

## Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Backend | Python 3.11+, FastAPI, SQLAlchemy, SQLite |
| Frontend | Next.js 14+, TypeScript, Tailwind CSS, Shadcn/UI |
| Procesado de vídeo | FFmpeg (stream-copy + extracción paralela) |
| Autenticación | JWT (Bearer token) |
| Rate limiting | SlowAPI |
| Facturación | Stripe (Checkout + Webhooks + Portal) |

---

## Arquitectura

```
┌─────────────────────────────────────────────────────┐
│                   Frontend (Next.js)                 │
│   Login → Dashboard → Subida → Revisión de clips    │
└────────────────────────┬────────────────────────────┘
                         │ REST API
┌────────────────────────▼────────────────────────────┐
│                  Backend (FastAPI)                   │
│                                                      │
│  /api/videos   /api/clips    /api/batches            │
│  /api/admin    /api/billing  /api/auth               │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │             AI Worker (asyncio)              │   │
│  │  Vídeo → Detector → Eventos → FFmpeg → Clips │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  SQLite (MVP) / PostgreSQL (producción)              │
└─────────────────────────────────────────────────────┘
```

**Flujo de datos:**
1. Usuario sube vídeo → se guarda en `backend/uploads/`
2. Worker de IA procesa el vídeo en segundo plano (semáforo: máx. 3 vídeos simultáneos)
3. El detector genera eventos con timestamp y nivel de confianza
4. FFmpeg extrae clips centrados en cada evento → `backend/clips/`
5. Los clips se asignan automáticamente a lotes de revisión
6. Revisores aprueban o rechazan cada clip en el dashboard
7. Clips aprobados exportables como dataset de entrenamiento

---

## Requisitos previos

- Python 3.11+
- Node.js 18+
- [FFmpeg](https://ffmpeg.org/download.html) instalado en el sistema (o ruta configurada en `.env`)
- (Opcional) Stripe CLI para pruebas de webhooks locales

---

## Instalación

### 1. Clonar el repositorio

```bash
git clone https://github.com/TU_USUARIO/flak-ai-v2.git
cd flak-ai-v2
```

### 2. Backend

```bash
cd backend

# Crear entorno virtual
python -m venv venv

# Activar (Windows)
venv\Scripts\activate
# Activar (Linux/Mac)
source venv/bin/activate

pip install -r requirements.txt
```

Crea el archivo `backend/.env`:

```env
DATABASE_URL=sqlite:///./flak_ai.db
JWT_SECRET=cambia-esto-en-produccion

# FFmpeg (dejar vacío si está en PATH del sistema)
FFMPEG_PATH=
FFPROBE_PATH=

# Detector: "mock" para desarrollo, "torch" para modelo real
DETECTOR_BACKEND=mock

# Clip: ventana en segundos centrada en el evento
CLIP_WINDOW_SECONDS=10.0

# Stripe (opcional — solo si usas facturación)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_ID_PRO=price_...
STRIPE_PRICE_ID_CLUB=price_...
APP_URL=http://localhost:3000
```

Iniciar el backend:

```bash
# Crear usuario administrador (admin / admin)
python seed_admin.py

# Lanzar servidor
uvicorn main:app --reload
# → http://localhost:8000
# → Docs: http://localhost:8000/docs
```

### 3. Frontend

```bash
cd frontend
npm install
```

Crea `frontend/.env.local`:

```env
NEXT_PUBLIC_API_URL=http://localhost:8000
```

```bash
npm run dev
# → http://localhost:3000
```

### Inicio rápido (Windows, todo de una vez)

```powershell
.\start.ps1
```

---

## Variables de entorno (referencia completa)

| Variable | Descripción | Por defecto |
|----------|-------------|-------------|
| `DATABASE_URL` | URL de base de datos | `sqlite:///./flak_ai.db` |
| `JWT_SECRET` | Clave secreta para tokens JWT | *(cambiar en producción)* |
| `FFMPEG_PATH` | Ruta al binario ffmpeg | busca en PATH |
| `DETECTOR_BACKEND` | Motor de detección: `mock` o `torch` | `mock` |
| `CLIP_WINDOW_SECONDS` | Duración total del clip (seg) | `10.0` |
| `AUTO_APPROVE_CONFIDENCE` | Confianza mínima para aprobación automática (%) | `101.0` (todo a revisión) |
| `TORCH_DETECTION_STRIDE` | Stride de detección en segundos | `5.0` |
| `TORCH_DETECTION_THRESHOLD` | Confianza mínima para emitir evento (%) | `40.0` |
| `STRIPE_SECRET_KEY` | Clave privada de Stripe | — |
| `CORS_ORIGINS` | Orígenes adicionales separados por comas | — |

---

## Tipos de eventos detectados

| Evento | Código |
|--------|--------|
| Gol | `goal` |
| Córner | `corner` |
| Saque de banda | `throw_in` |
| Falta | `foul` |
| Saque de puerta | `goal_kick` |
| Disparo a puerta | `shot_on_target` |

---

## API REST

Documentación interactiva disponible en `http://localhost:8000/docs` (Swagger UI).

```
POST   /api/auth/register            Registro de usuario
POST   /api/auth/token               Login → JWT

GET    /api/videos/                  Listar vídeos del equipo
POST   /api/videos/upload            Subir vídeo (chunked)
DELETE /api/videos/{id}              Eliminar vídeo

GET    /api/clips/                   Listar clips pendientes de revisión
PATCH  /api/clips/{id}/review        Aprobar o rechazar clip

GET    /api/batches/                 Listar lotes del equipo
POST   /api/batches/                 Crear nuevo lote
PATCH  /api/batches/{id}/complete    Marcar lote como completado

GET    /api/billing/plans            Planes y suscripción actual
POST   /api/billing/checkout         Crear sesión de pago Stripe
POST   /api/billing/portal           Portal de gestión de suscripción

GET    /api/admin/teams              Listar todos los equipos (admin)
PATCH  /api/admin/teams/{id}/approve Aprobar equipo (admin)
GET    /api/admin/export/reviewed    Exportar dataset global (admin)

GET    /api/export/reviewed          Exportar dataset del equipo (JSONL/CSV)
```

---

## Planes de suscripción

| Plan | Límite | Descripción |
|------|--------|-------------|
| `free_trial` | 1 vídeo | Prueba gratuita para nuevos equipos |
| `pro` | Ilimitado | Plan profesional |
| `club` | Ilimitado | Plan club con funciones avanzadas |

---

## Modelo de acceso y equipos

- Cada equipo nuevo queda en estado **pendiente de aprobación** hasta que un admin lo aprueba en el panel de administración.
- El admin puede rechazar equipos; el nombre se libera para futuros registros.
- Credenciales del admin inicial (generadas con `seed_admin.py`): **admin** / **admin** — cambiar en producción.

---

## Estructura del proyecto

```
flak-ai-v2/
├── backend/
│   ├── main.py                  # App FastAPI + middlewares
│   ├── ai_worker.py             # Worker de procesado en background
│   ├── clip_generation.py       # Extracción de clips con FFmpeg
│   ├── models.py                # Modelos SQLAlchemy (ORM)
│   ├── schemas.py               # Esquemas Pydantic (validación)
│   ├── config.py                # Configuración 12-factor (.env)
│   ├── auth.py                  # JWT + dependencias de autenticación
│   ├── auto_ingest_watcher.py   # Watcher de carpeta local
│   ├── detection/               # Capa de detección (mock / torch / onnx)
│   ├── billing/                 # Lógica de suscripción y tiers
│   └── routers/
│       ├── auth.py              # Registro y login
│       ├── videos.py            # CRUD vídeos + subida chunked
│       ├── clips.py             # Revisión de clips
│       ├── batches.py           # Gestión de lotes
│       ├── billing.py           # Stripe checkout + webhooks
│       ├── admin.py             # Panel administrador
│       ├── export_reviewed.py   # Exportación de datasets
│       └── ml_admin.py          # Métricas ML para admin
├── frontend/
│   ├── app/
│   │   ├── dashboard/           # Dashboard principal (vídeos, clips, lotes)
│   │   └── login/               # Página de autenticación
│   ├── components/              # Componentes reutilizables
│   └── lib/                     # Utilidades y cliente API
├── datasets/                    # Manifiestos de dataset exportados
└── start.ps1                    # Script de inicio rápido (Windows)
```

---

## Licencia

Copyright © 2026 NotSaam. Todos los derechos reservados.

Queda prohibida la reproducción, distribución o modificación de este software, total o parcial, sin autorización expresa y por escrito del autor.
