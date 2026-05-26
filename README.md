# Sistema Inteligente de Gestión de Justificación de Inasistencias

> **Plataforma enterprise de automatización** para la recepción, análisis con IA y clasificación de solicitudes de justificación estudiantil. Desarrollado con n8n, Docker, OpenAI GPT-4o, OCR y Telegram.

---

## 📋 Descripción General

Este sistema automatiza completamente el flujo de gestión de inasistencias académicas:

1. El estudiante envía un formulario web con su documento de soporte
2. El webhook n8n recibe la solicitud y la procesa automáticamente
3. OCR.Space extrae el texto del documento (PDF o imagen)
4. OpenAI GPT-4o analiza el documento y lo clasifica
5. Un motor de decisión híbrido determina el resultado
6. El resultado se guarda en PostgreSQL y Google Sheets
7. El estudiante recibe notificación automática por Telegram

---

## 🏗 Arquitectura del Sistema

```
Formulario Web
      │
      ▼
[n8n Webhook] ──► Validación & Sanitización
      │
      ▼
[PostgreSQL] ◄── Upsert Estudiante + Crear Solicitud
      │
      ▼
[Redis] ──► Verificación de Duplicados (Hash SHA-256)
      │
      ├── Duplicado ──► Rechazado automático
      │
      ▼
[OCR.Space] ──► Extracción de texto
      │
      ▼
[OpenAI GPT-4o] ──► Clasificación + Score de confianza
      │
      ▼
[Motor de Decisión]
  ├── Score ≥ 80% ──► Aprobado automático
  ├── Score 40-79% ──► Revisión manual
  └── Score < 40%  ──► Rechazado automático
      │
      ▼
[PostgreSQL + Google Sheets] ──► Almacenamiento
      │
      ▼
[Telegram Bot] ──► Notificación al estudiante
```

---

## 🛠 Stack Tecnológico

| Componente | Tecnología | Versión |
|------------|-----------|---------|
| Motor de automatización | n8n | Latest |
| Base de datos | PostgreSQL | 16 Alpine |
| Cache & deduplicación | Redis | 7 Alpine |
| IA / NLP | OpenAI GPT-4o | API |
| OCR | OCR.Space API | Engine 2 |
| Notificaciones | Telegram Bot API | — |
| Almacenamiento | Google Sheets + Drive | API v4 |
| Túnel público | ngrok | Latest |
| Contenedores | Docker + Compose | v3.9 |

---

## 🚀 Instalación y Despliegue

### Requisitos Previos

- Docker >= 24.x
- Docker Compose >= 2.x
- Git
- Cuenta en: OpenAI, Telegram BotFather, Google Cloud, OCR.Space, ngrok

### 1. Clonar el Repositorio

```bash
git clone https://github.com/TU_USUARIO/justificaciones-ia.git
cd justificaciones-ia
```

### 2. Configurar Variables de Entorno

```bash
cp .env.example .env
nano .env   # Editar con tus credenciales reales
```

Consulta `.env.example` para la documentación completa de cada variable.

### 3. Iniciar los Servicios Docker

```bash
docker-compose up -d
```

Verificar que todos los servicios estén corriendo:

```bash
docker-compose ps
docker-compose logs -f n8n
```

### 4. Obtener URL pública de ngrok

```bash
# Ver la URL pública generada
curl http://localhost:4040/api/tunnels | python3 -m json.tool
```

Actualiza `NGROK_PUBLIC_URL` en tu `.env` con la URL obtenida y reinicia n8n:

```bash
docker-compose restart n8n
```

### 5. Importar el Workflow en n8n

1. Accede a n8n: `http://localhost:5678`
2. Usuario y contraseña: los que configuraste en `.env`
3. Ve a **Settings → Import from file**
4. Importa `workflows/justificaciones_workflow.json`
5. Configura las credenciales (ver sección siguiente)

### 6. Configurar Credenciales en n8n

En **Settings → Credentials**, crea:

| Nombre | Tipo | Datos necesarios |
|--------|------|-----------------|
| `PostgreSQL Justificaciones` | PostgreSQL | Host, DB, usuario, contraseña |
| `OpenAI Justificaciones` | OpenAI API | API Key |
| `Telegram Bot Justificaciones` | Telegram API | Bot Token |
| `Google Sheets Justificaciones` | Google OAuth2 | Service Account JSON |

### 7. Configurar el Webhook de Telegram

```bash
curl -X POST "https://api.telegram.org/bot{TU_TOKEN}/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://TU_NGROK.ngrok-free.app/webhook/telegram"}'
```

### 8. Activar el Workflow

En n8n, activa el workflow con el toggle en la parte superior derecha.

### 9. Probar el Sistema

Abre el formulario web en el navegador:
```
file:///ruta/al/proyecto/frontend/index.html
```

O sirve con Python:
```bash
cd frontend && python3 -m http.server 8080
```

---

## 📁 Estructura del Proyecto

```
justificaciones-ia/
├── 📄 docker-compose.yml         # Orquestación Docker
├── 📄 .env.example               # Variables de entorno (template)
├── 📄 .gitignore                 # Exclusiones Git
│
├── 📂 workflows/
│   └── justificaciones_workflow.json   # Workflow n8n exportable
│
├── 📂 frontend/
│   └── index.html                # Formulario web del estudiante
│
├── 📂 scripts/
│   └── init_db.sql               # Schema PostgreSQL completo
│
├── 📂 prompts/
│   └── ai_prompts.md             # Prompts de IA documentados
│
├── 📂 docs/
│   └── technical_docs.md         # Documentación técnica
│
├── 📂 logs/                      # Logs del sistema (gitignored)
├── 📂 screenshots/               # Screenshots del flujo n8n
└── 📂 .github/
    └── workflows/                # CI/CD pipelines
```

---

## 🔒 Seguridad

- Todas las credenciales via variables de entorno
- Nunca se hardcodean secretos en el código
- Validación y sanitización de todos los inputs
- Verificación de tipo y tamaño de archivos
- Detección de documentos duplicados via SHA-256
- Redis para rate limiting
- n8n con autenticación básica habilitada
- PostgreSQL en red Docker interna (no expuesto externamente en producción)

---

## 🤖 Comandos del Bot de Telegram

| Comando | Descripción |
|---------|-------------|
| `/start` | Bienvenida e instrucciones |
| `/status [CÓDIGO]` | Consultar estado de una solicitud (Ej: `/status JUS-2024-00001`) |
| `/history` | Ver historial de tus solicitudes |
| `/help` | Mostrar ayuda completa |

---

## 📊 Dashboard y Métricas

Las métricas están disponibles en Google Sheets y mediante la vista `dashboard_metrics` en PostgreSQL:

```sql
SELECT * FROM dashboard_metrics;
```

Campos disponibles: `total_approved`, `total_rejected`, `total_manual_review`, `avg_confidence_score`, `avg_processing_minutes`, `requests_last_24h`.

---

## 🔍 Lógica de Decisión

| Score de Confianza | Acción |
|-------------------|--------|
| ≥ 80% | ✅ Aprobación automática |
| 40% – 79% | 🔍 Derivado a revisión manual |
| < 40% | ❌ Rechazo automático |
| Fraude detectado | ❌ Rechazo por fraude |
| Duplicado | ❌ Rechazo por duplicado |

---

## 🐛 Troubleshooting

**n8n no inicia:**
```bash
docker-compose logs n8n | tail -50
```

**PostgreSQL connection refused:**
```bash
docker-compose exec postgres pg_isready -U justificaciones_admin
```

**Webhook no recibe datos:**
- Verificar que ngrok esté corriendo: `http://localhost:4040`
- Actualizar `NGROK_PUBLIC_URL` en `.env`
- Reiniciar n8n: `docker-compose restart n8n`

**OCR sin resultados:**
- Verificar `OCRSPACE_API_KEY` en `.env`
- Comprobar que el archivo sea JPG, PNG o PDF menor a 5MB

---

## 📜 Licencia

Proyecto académico desarrollado para Campuslands. Uso educativo.

---

*Desarrollado con n8n + Docker + OpenAI + OCR.Space + Telegram*
