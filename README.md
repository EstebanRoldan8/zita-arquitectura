# zita-arquitectura
Caso de estudio de arquitectura — SaaS multi-tenant para negocios de servicios en LATAM (WhatsApp + IA + gestión). Documentación técnica, sin código fuente.
# ZITA

Plataforma SaaS multi-tenant para negocios de servicios en LATAM (barberías, spas, clínicas estéticas, veterinarias). Automatiza la agenda por WhatsApp: un asistente conversacional con IA atiende a los clientes finales —agenda, reagenda, cancela, consulta disponibilidad, responde por texto o nota de voz— mientras el dueño gestiona su negocio desde un dashboard web (citas, clientes, empleados, inventario, flujo de caja, fidelización, reportes).

El problema que resuelve es concreto: en este segmento la agenda vive en el chat de WhatsApp del dueño. ZITA convierte ese canal en un sistema transaccional real, sin pedirle al cliente final que instale nada ni cambie de app.

## Stack

| Capa | Tecnología | Por qué |
|---|---|---|
| Backend | Node.js + Express | El dominio es I/O-bound (webhooks, llamadas a APIs externas, DB). El modelo async de Node encaja bien y mantiene el stack en un solo lenguaje. Arquitectura en capas: `routes → middlewares → controllers → services`. |
| Base de datos | PostgreSQL (Supabase) | Las garantías del negocio (no doble-booking, no ingresos duplicados, stock no negativo) viven en constraints de la base, no en código de aplicación. Supabase aporta además Auth con JWT y Row Level Security nativa para el multi-tenancy. |
| Frontend | React 19 + Vite + Tailwind CSS 4 | SPA con React Query para cache/estado de servidor (invalidación por mutación en lugar de estado global manual), React Router para navegación y Recharts para las métricas. PWA para instalación en móvil. |
| IA conversacional | Claude (Anthropic) con tool calling | El bot no genera texto libre sobre datos del negocio: ejecuta 14 tools tipadas (`agendar_cita`, `verificar_disponibilidad`, `cancelar_cita`, `escalar_a_humano`, etc.) que validan contra la DB. El LLM decide *qué* hacer; la lógica de negocio decide *si se puede*. |
| Transcripción de voz | Whisper (Groq) | Los clientes en este mercado mandan notas de voz constantemente. El audio se descarga de la API de Meta, se transcribe y entra al mismo pipeline que un mensaje de texto. |
| Mensajería | WhatsApp Cloud API (Meta) | Canal nativo del usuario final. Webhooks firmados con HMAC para mensajes entrantes. |
| Pagos | Wompi | Pasarela dominante en Colombia. Firma de integridad en checkout, verificación de firma en webhooks y reconciliación periódica contra su API. |
| Observabilidad | Sentry + Winston + tabla `system_alerts` | Errores de aplicación a Sentry; fallos de negocio críticos (activación de plan fallida, envío de WhatsApp fallido) quedan además en una tabla consultable desde el panel de administración. |
| Infraestructura | Railway (API) + Vercel (frontend) | Deploy continuo desde `main`. Los cron jobs corren dentro del proceso del backend con locks distribuidos en DB (ver abajo), así que escalar réplicas no duplica trabajo. |

## Decisiones de arquitectura

### 1. Las invariantes de concurrencia viven en PostgreSQL, no en JavaScript

Dos clientes pueden intentar reservar el mismo slot con el mismo empleado en la misma milésima —por el bot, por el dashboard, o uno por cada canal. En lugar de confiar en un check-then-insert en la aplicación (que siempre tiene ventana de carrera), la regla es un índice único parcial:

```sql
CREATE UNIQUE INDEX idx_no_double_booking ON appointments
  (business_id, employee_id, scheduled_at)
  WHERE status IN ('pending', 'confirmed');
```

El `WHERE` es la parte importante: una cita cancelada libera el slot sin borrar la fila. El mismo patrón protege contra ingresos duplicados por cita (`UNIQUE` parcial sobre `income.appointment_id`) y stock negativo (`CHECK (stock >= 0)`). El código de aplicación trata la violación de constraint (`23505`) como un resultado esperado y responde en consecuencia, no como una excepción.

### 2. Multi-tenancy con Row Level Security

Cada tabla con datos de tenant lleva `business_id` y políticas RLS que resuelven el negocio del usuario autenticado vía una función `SECURITY DEFINER`:

```sql
CREATE POLICY "owner_services_own" ON services
  FOR ALL USING (business_id = get_business_id_for_user(auth.uid()));
```

El frontend habla con Supabase usando la anon key, así que el aislamiento entre tenants lo garantiza la base de datos incluso si una query del cliente olvida un filtro. El backend usa el service role únicamente para flujos que legítimamente cruzan tenants (webhooks entrantes, cron jobs, chatbot), siempre con filtros explícitos por `business_id`. Las migraciones posteriores auditan y cierran tablas que quedaron sin políticas —RLS solo sirve si cubre el 100% de las tablas.

### 3. Webhooks idempotentes en ambos extremos

Meta reintenta webhooks agresivamente cuando no recibe 200 a tiempo, y un reintento no puede convertirse en una cita duplicada o una respuesta doble del bot. Cada conversación mantiene una ventana deslizante de los últimos 50 `message_id` procesados (columna `text[]` con índice GIN); un ID repetido se descarta antes de tocar el LLM.

En pagos el problema es el inverso: el webhook puede *no llegar*. Un cron reconcilia cada 2 horas las transacciones aprobadas en Wompi contra la tabla `payments` local. Eso crea una carrera webhook-vs-cron, resuelta con un check de idempotencia sobre el estado del pago y —más importante— una única función compartida de activación de plan que usan ambos caminos. Si el webhook y la reconciliación divergen en lógica, tarde o temprano divergen en datos.

### 4. Verificación criptográfica de webhooks, con fail-closed

Los webhooks de Meta se validan con HMAC-SHA256 sobre el body **crudo**. Express parsea JSON antes de que cualquier middleware vea el buffer original, así que la ruta del webhook monta su propio parser con `verify()` para capturar `rawBody` antes del parseo —re-serializar el objeto parseado no reproduce los bytes firmados. La comparación usa `crypto.timingSafeEqual` y, si el secret no está configurado, el endpoint rechaza todo en lugar de degradarse a "aceptar sin validar". Los webhooks de Wompi validan su propio esquema de firma sobre los campos de la transacción, y el checkout genera la firma de integridad que Wompi exige para que el monto no sea manipulable en el cliente.

### 5. Cron jobs con locks distribuidos vía `INSERT` + `UNIQUE`

Recordatorios de citas, auto-completado, resumen diario y reconciliación de pagos corren con node-cron dentro del proceso del API. Con más de una réplica, cada job correría N veces. La solución no requiere Redis: una tabla `scheduler_locks` con constraint `UNIQUE` sobre la llave del lock. Adquirir el lock es un `INSERT` directo —si otro proceso lo tiene, la violación `23505` es la señal de "no adquirido". Los locks llevan TTL y los expirados se limpian con un `DELETE` idempotente, de modo que un proceso que muere con el lock tomado no bloquea el job para siempre.

### 6. Resiliencia frente a APIs externas (y control de costos del LLM)

Todo lo externo puede fallar o tardar. Las llamadas a Claude pasan por un wrapper de retry con backoff exponencial que clasifica errores: reintenta timeouts y 5xx/429, no reintenta 400/401/403 (reintentar un request inválido solo quema cuota). Axios lleva timeouts explícitos en cada integración y los códigos de error de red están enumerados como transitorios para la reconciliación de pagos.

En la otra dirección, el costo por conversación se controla en capas: fast-paths que responden saludos y despedidas sin invocar al LLM, historial de conversación acotado, cache con TTL de datos del negocio que se inyectan al prompt, y límites diarios de mensajes de bot por cliente y por negocio según plan —persistidos en DB, no en memoria, para sobrevivir reinicios y réplicas.

### Otras piezas con criterio propio

- **Borrado GDPR real**: endpoint administrativo que ejecuta un `DELETE` permanente (no soft-delete) en orden de dependencias FK, anonimiza órdenes en lugar de borrarlas (registro contable) y devuelve un log de auditoría de filas afectadas.
- **Credenciales de WhatsApp por tenant cifradas** en reposo (AES) y descifradas solo al momento de usarlas.
- **Cancelación de citas con guard de estado**: el `UPDATE` a `cancelled` exige `status IN ('pending','confirmed')` en la misma sentencia, así una cita completada no puede cancelarse por una carrera entre el bot y el dashboard.
- **49 migraciones SQL versionadas e idempotentes** (bloques `DO $$` donde PostgreSQL no soporta `IF NOT EXISTS`), que cuentan la evolución real del esquema.

## Flujo de desarrollo

El proyecto se desarrolló con asistencia de IA (Claude / Claude Code) como herramienta dentro de un proceso de ingeniería normal: las decisiones de arquitectura, la revisión de cada cambio y la verificación en entorno real son humanas. La asistencia se usó sobre todo para acelerar auditorías sistemáticas del código existente (seguridad, escalabilidad, servicios críticos) cuyos hallazgos se triaron manualmente y se convirtieron en migraciones y fixes versionados en commits atómicos. El historial de git refleja ese ciclo: auditar, priorizar, corregir, verificar.

## Estructura del proyecto

```
zita/
├── backend/
│   └── src/
│       ├── config/          # Clientes de Supabase, Anthropic, WhatsApp, pagos, planes
│       ├── controllers/     # Handlers HTTP: validación de input y orquestación (23 recursos)
│       ├── middlewares/     # JWT, contexto de tenant, firma HMAC de webhooks,
│       │                    # rate limiting, sanitización, gates por plan/feature
│       ├── routes/          # Definición de endpoints REST
│       ├── services/        # Lógica de negocio: chatbot, scheduler/cron, pagos,
│       │                    # notificaciones, reportes (PDF/Excel), GDPR, alertas
│       └── utils/           # Disponibilidad de agenda, retry, cifrado, timezone, logger
│
├── frontend/
│   └── src/
│       ├── components/      # UI por dominio (citas, clientes, métricas) + primitivos
│       ├── pages/           # Rutas públicas, dashboard del negocio y panel admin
│       ├── context/         # Sesión y autenticación
│       ├── hooks/           # Data-fetching con React Query
│       └── lib/             # Cliente HTTP y cliente Supabase
│
└── supabase/
    └── migrations/          # Esquema, políticas RLS, constraints de concurrencia,
                             # funciones SQL e índices de performance (49 migraciones)
```

## Correr en local

```bash
# Backend
cd backend
npm install
cp .env.example .env   # Supabase, Anthropic, WhatsApp Cloud API, pasarela de pagos
npm run dev            # http://localhost:3000

# Frontend
cd frontend
npm install
npm run dev            # http://localhost:5173
```

El chatbot y los pagos requieren credenciales propias de Meta for Developers y de la pasarela; el resto del sistema funciona sin ellas.

---

## Author

**Juan Esteban Roldán Mayorga**

- 💼 Full-stack developer — SaaS & AI integration
- 📧 [esteban.zita.ia@gmail.com](mailto:esteban.zita.ia@gmail.com)
- 🔗 [GitHub](https://github.com/EstebanRoldan8)
