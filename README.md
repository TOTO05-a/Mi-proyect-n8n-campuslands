# Sistema Inteligente de Gestión de Justificación de Inasistencias

> **Plataforma enterprise de automatización** para la recepción, análisis con IA y clasificación de solicitudes de justificación estudiantil. Desarrollado con n8n, Docker, OpenAI GPT-4o, OCR y Telegram.

---

## 📋 Descripción General

Este sistema automatiza completamente el flujo de gestión de inasistencias académicas:

1. El estudiante envía un formulario web con su documento de soporte
2. El webhook n8n recibe la solicitud y la procesa automáticamente
3. OCR.Space extrae el texto del documento (PDF o imagen)
4. OpenAI Gemini analiza el documento y lo clasifica
5. Un motor de decisión híbrido determina el resultado
6. El resultado se guarda en PostgreSQL y Google Sheets
7. El estudiante recibe notificación automática por Telegram


### 4. Obtener URL pública de ngrok

```bash
# Ver la URL pública generada
curl http://localhost:4040/api/tunnels | python3 -m json.tool
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

##  Comandos del Bot de Telegram

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