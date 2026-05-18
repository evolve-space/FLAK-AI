# FlakAI v2

> Herramienta automática de análisis de partidos de fútbol con IA. Sube el vídeo, recibe los clips clasificados.(Proyecto actualmente desarrollandose situado en un 60% de su desarrollo final).

---

## ¿Qué hace FlakAI?

FlakAI v2 analiza grabaciones de partidos de fútbol de forma completamente automática. El usuario sube el vídeo y la herramienta se encarga del resto: detecta todos los eventos relevantes del partido, extrae un clip por cada uno y los entrega organizados en carpetas por tipo de evento. Sin intervención manual. Sin configuración adicional.

**Resultado final:** una carpeta estructurada con todos los clips del partido, listos para usar.

```
partido_final/
├── goles/
│   ├── gol_00_23_14.mp4
│   └── gol_00_67_42.mp4
├── corners/
│   ├── corner_00_08_33.mp4
│   └── corner_00_51_07.mp4
├── faltas/
│   └── falta_00_34_19.mp4
├── disparos/
│   └── disparo_00_44_55.mp4
└── saques/
│   └── saque_banda_00_12_01.mp4
└── porteria/
    └── saque_porteria_00_12_01.mp4
```

---

## Flujo de uso

```
1. Sube el vídeo del partido
        ↓
2. La IA analiza el vídeo automáticamente
        ↓
3. FFmpeg extrae un clip por cada evento detectado
        ↓
4. Los clips se clasifican por tipo de evento
        ↓
5. Descarga la carpeta organizada con todos los clips
```

Sin pasos intermedios. Sin revisión manual. Sin configuración.

---

## Eventos detectados

| Evento | Carpeta de salida |
|--------|------------------|
| Gol | `goles/` |
| Córner | `corners/` |
| Saque de banda | `saques_banda/` |
| Falta | `faltas/` |
| Saque de puerta | `saques_puerta/` |
| Disparo a puerta | `disparos/` |

Cada clip tiene una duración de ±30 segundos alrededor del momento exacto del evento.

---

## Características técnicas

- **Detección automática** — Motor de IA analiza el vídeo completo y localiza cada evento con su timestamp y nivel de confianza.
- **Extracción paralela con FFmpeg** — Hasta 4 clips extraídos simultáneamente por vídeo. Stream-copy para máxima velocidad sin recodificación.
- **Clasificación automática** — Cada clip se clasifica y organiza en su carpeta correspondiente sin intervención humana.
- **Procesado en segundo plano** — El usuario sube el vídeo y puede cerrar el navegador. El sistema procesa en background y notifica cuando termina.
- **Multi-equipo** — Varios equipos o usuarios pueden usar la plataforma de forma independiente con sus propios vídeos y resultados.
- **Facturación integrada** — Planes de acceso con Stripe. Portal de cliente para gestionar suscripciones.
- **Auto-ingesta por carpeta** — Modo servidor: detecta nuevos vídeos en una carpeta local y los procesa automáticamente sin interfaz web.

---

## Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Backend | Python 3.11+, FastAPI, SQLAlchemy, SQLite |
| Frontend | Next.js 14+, TypeScript, Tailwind CSS |
| Procesado de vídeo | FFmpeg (stream-copy + extracción paralela) |
| Motor de IA | PyTorch / ONNX (detector de eventos) |
| Autenticación | JWT (Bearer token) |
| Facturación | Stripe (Checkout + Webhooks + Portal) |

---

## Arquitectura

```
┌─────────────────────────────────────────────┐
│              Frontend (Next.js)              │
│   Subida de vídeo → Progreso → Descarga     │
└──────────────────┬──────────────────────────┘
                   │ REST API
┌──────────────────▼──────────────────────────┐
│              Backend (FastAPI)               │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │          AI Worker (asyncio)        │   │
│  │                                     │   │
│  │  Vídeo → Detector IA → Timestamps   │   │
│  │       → FFmpeg → Clips              │   │
│  │       → Clasificación por evento    │   │
│  │       → Carpeta de salida lista     │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  SQLite / PostgreSQL                        │
└─────────────────────────────────────────────┘
```

---

## Instalación

### Requisitos

- Python 3.11+
- Node.js 18+
- [FFmpeg](https://ffmpeg.org/download.html) instalado en el sistema

### 1. Clonar el repositorio

```bash
git clone https://github.com/TU_USUARIO/flak-ai-v2.git
cd flak-ai-v2
```

### 2. Backend

```bash
cd backend
python -m venv venv

# Windows
venv\Scripts\activate
# Linux/Mac
source venv/bin/activate

pip install -r requirements.txt
```

Crea `backend/.env`:

```env
DATABASE_URL=sqlite:///./flak_ai.db
JWT_SECRET=cambia-esto-en-produccion

# FFmpeg (dejar vacío si está en PATH del sistema)
FFMPEG_PATH=
FFPROBE_PATH=

# Motor de detección
DETECTOR_BACKEND=torch

# Duración del clip centrado en el evento (segundos)
CLIP_WINDOW_SECONDS=10.0

# Stripe (opcional)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_ID_PRO=price_...
STRIPE_PRICE_ID_CLUB=price_...
APP_URL=http://localhost:3000
```

```bash
python seed_admin.py   # Crea usuario administrador (admin / admin)
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

### Inicio rápido (Windows)

```powershell
.\start.ps1
```

---

## Variables de entorno

| Variable | Descripción | Por defecto |
|----------|-------------|-------------|
| `DATABASE_URL` | URL de base de datos | `sqlite:///./flak_ai.db` |
| `JWT_SECRET` | Clave secreta JWT | *(cambiar en producción)* |
| `FFMPEG_PATH` | Ruta al binario ffmpeg | busca en PATH |
| `DETECTOR_BACKEND` | Motor de detección: `mock` o `torch` | `mock` |
| `CLIP_WINDOW_SECONDS` | Duración total del clip (seg) | `10.0` |
| `TORCH_DETECTION_STRIDE` | Stride de detección en segundos | `5.0` |
| `TORCH_DETECTION_THRESHOLD` | Confianza mínima para emitir evento (%) | `40.0` |
| `TORCH_NMS_WINDOW` | Ventana NMS: suprime eventos duplicados (seg) | `20.0` |
| `STRIPE_SECRET_KEY` | Clave privada de Stripe | — |
| `CORS_ORIGINS` | Orígenes adicionales separados por comas | — |

---

## API REST

Documentación interactiva en `http://localhost:8000/docs`.

```
POST   /api/auth/register       Registro de usuario
POST   /api/auth/token          Login → JWT

GET    /api/videos/             Listar vídeos
POST   /api/videos/upload       Subir vídeo (procesado automático)
GET    /api/videos/{id}/status  Estado del procesado y progreso
DELETE /api/videos/{id}         Eliminar vídeo

GET    /api/clips/              Listar clips generados
GET    /api/clips/{id}/download Descargar clip individual

GET    /api/billing/plans       Planes disponibles
POST   /api/billing/checkout    Crear sesión de pago Stripe
POST   /api/billing/portal      Portal de gestión de suscripción

GET    /api/admin/teams         Panel de administración
```

---

## Planes de suscripción

| Plan | Límite | Descripción |
|------|--------|-------------|
| `free_trial` | 1 vídeo | Prueba gratuita |
| `pro` | Ilimitado | Plan profesional |
| `club` | Ilimitado | Plan club |

---

## Estructura del proyecto

```
flak-ai-v2/
├── backend/
│   ├── main.py                  # App FastAPI + middlewares
│   ├── ai_worker.py             # Worker automático de procesado
│   ├── clip_generation.py       # Extracción de clips con FFmpeg
│   ├── models.py                # Modelos SQLAlchemy
│   ├── schemas.py               # Esquemas Pydantic
│   ├── config.py                # Configuración 12-factor (.env)
│   ├── auth.py                  # JWT + autenticación
│   ├── auto_ingest_watcher.py   # Auto-ingesta por carpeta local
│   ├── detection/               # Motor de detección IA (torch/onnx/mock)
│   ├── billing/                 # Suscripciones y tiers
│   └── routers/                 # Endpoints REST
├── frontend/
│   ├── app/
│   │   ├── dashboard/           # Dashboard: subida, progreso, descarga
│   │   └── login/               # Autenticación
│   ├── components/              # Componentes UI
│   └── lib/                     # Cliente API y utilidades
├── datasets/                    # Manifiestos exportados
└── start.ps1                    # Inicio rápido Windows
```

---

## Licencia

Copyright © 2026 NotSaam. Todos los derechos reservados.

Queda prohibida la reproducción, distribución o modificación de este software, total o parcial, sin autorización expresa y por escrito del autor.
