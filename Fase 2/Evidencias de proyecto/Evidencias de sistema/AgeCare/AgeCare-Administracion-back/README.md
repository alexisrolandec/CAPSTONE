# AgeCare — Admin API

## Instrucciones de arranque, seguir al pie de la letra...

Para utilizar la consola usaremos docker, crear siguiente archivo .env a la misma altura de docker-compose.yml.
# Entorno
ADMIN_ENVIRONMENT=dev

# Conexión con la consola: sin barra / al final
ADMIN_ALLOWS_ORIGINS=http://localhost:5173,http://127.0.0.1:5173,http://localhost:3000
ADMIN_FRONTEND_URL=http://localhost:5173

# PostgreSQL: Docker Compose configura la conexión interna del contenedor
ADMIN_DATABASE_URL=postgresql+asyncpg://agecare:agecare@localhost:5432/agecare_admin

# El Compose actual utiliza su propio JWT de desarrollo
ADMIN_JWT_SECRET=solo-para-desarrollo

# Cuentas iniciales: solo se crean si no existe ninguna cuenta
ADMIN_BOOTSTRAP_DEV_ACCOUNTS=true
ADMIN_ALLOWED_EMAIL_DOMAINS=wellq.co.uk,wellq.co,gmail.com

# Seguridad
ADMIN_ENCRYPTION_KEY=PEGAR_CLAVE_FERNET
ADMIN_MFA_REQUIRED_ROLES=admin,analyst,support,editor,moderator
ADMIN_INVITATION_HOURS=24

# Correo Gmail: necesario para enviar invitaciones
ADMIN_SMTP_HOST=smtp.gmail.com
ADMIN_SMTP_PORT=587
ADMIN_SMTP_SECURITY=starttls
ADMIN_SMTP_USERNAME=CORREO_REMITENTE@gmail.com
ADMIN_SMTP_PASSWORD=CONTRASENA_DE_APLICACION
ADMIN_MAIL_FROM=CORREO_REMITENTE@gmail.com
ADMIN_MAIL_WORKER_ENABLED=true

ADMIN_CONTENT_WORKER_ENABLED=false
ADMIN_GEMINI_FREE_TIER_CONFIRMED=false


## IMPORTANTE


para iniciar la consola y utilizarla es necesario inicial con alguna cuenta activa, usar : USer=admin@wellq.co.uk , Password=Admin123!
como utilizaremos smpt de gmail, es necesario conseguir una clave, se deja los siguientes pasos para conseguirla.

1.Iniciar sesión con el Gmail que enviará las invitaciones.
2.Activar la verificación en dos pasos de esa cuenta; Google la exige para crear contraseñas de aplicación. Ayuda oficial.
Entrar en Contraseñas de aplicación.
3.Escribir un nombre, por ejemplo AgeCare desarrollo, y pulsar Crear.
4.Copiar la contraseña generada al .env, sin espacios:


Ejecutar docker compose build api para construir la imagen de la api, luego
Ejecutar esto 
## docker compose run --rm --no-deps api python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())" 
dara una clave al final, esa clave usar en ADMIN_ENCRYPTION_KEY, esto permite tener mfa, mas abajo estan las credenciales de admin.
LUego ejecutar docker compose up -d --build api y la api estara lista.

Encaso de que se hayan ejecutado todos esos comandos, y desea actualizar .env, necesitara ejecutar el siguiente comando de docker 
docker compose up -d --force-recreate api

Después del primer login, el admin deberá configurar MFA y guardar sus códigos de recuperación.

arrancar fronted, instrucciones estan ahi.
///////////////////////////////////////////////////////////


## Arranque rápido (Docker)

```bash
docker compose up --build
```

Levanta PostgreSQL, aplica las migraciones sin sembrar datos y sirve la API en
`http://localhost:8000`. Documentación interactiva (Swagger):
`http://localhost:8000/api/v1/admin/docs`.

## Arranque manual

```bash
python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
# Crear .env desde ejemplo solo si no existe; conservar secretos de instalaciones existentes.
alembic upgrade head                  # crea las tablas
# No ejecutar scripts.seed sobre datos existentes: es destructivo.
uvicorn app.main:app --reload
```

Configura ADMIN_DATABASE_URL y las demás variables necesarias antes de ejecutar Alembic. Las migraciones no crean cuentas. En desarrollo puede habilitarse la carga inicial no destructiva descrita arriba; en producción se requiere alta inicial controlada con credenciales propias. No resolverlo ejecutando el seed en una base existente.

## Credenciales demo (seed)

| Correo | Contraseña | Rol |
|---|---|---|
| admin@wellq.co.uk | Admin123! | admin |
| soporte@wellq.co.uk | Soporte123! | soporte |
| analista@wellq.co.uk | Analista123! | analista |
| editora@wellq.co.uk | Editora123! | editor |

(El administrador demo debe configurar MFA al primer acceso. Primero configura `ADMIN_ENCRYPTION_KEY` según la guía enlazada más abajo.)

## Probar con curl

```bash
BASE=http://localhost:8000/api/v1/admin

# Login de un rol sin MFA. Para el administrador utiliza la pantalla de configuración de MFA.
TOKEN=$(curl -s $BASE/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"analista@wellq.co.uk","password":"Analista123!"}' | python3 -c \
  'import sys,json;print(json.load(sys.stdin)["access_token"])')
AUTH="Authorization: Bearer $TOKEN"

# Sección 1 · Uso comercial
curl -s "$BASE/metrics/commercial/summary?period=current_month" -H "$AUTH"
curl -s "$BASE/metrics/commercial/registrations?period=last_7_days" -H "$AUTH"
curl -s "$BASE/metrics/commercial/plans" -H "$AUTH"
curl -s "$BASE/metrics/commercial/funnel" -H "$AUTH"

# Sección 2 · Estado operativo
curl -s "$BASE/ops/status" -H "$AUTH"
curl -s "$BASE/ops/latency?window=1h" -H "$AUTH"
curl -s "$BASE/ops/incidents?days=30" -H "$AUTH"

# Sección 3 · Usuarios y soporte
curl -s "$BASE/metrics/roles/summary" -H "$AUTH"
curl -s "$BASE/metrics/roles/weekly-active?weeks=8" -H "$AUTH"
curl -s "$BASE/support/summary" -H "$AUTH"
curl -s "$BASE/support/tickets?status=open" -H "$AUTH"

# Sección 4 · Uso por funcionalidad
curl -s "$BASE/metrics/features/adoption?days=30" -H "$AUTH"
curl -s "$BASE/metrics/features/alerts?threshold=0.15" -H "$AUTH"
```

## Tests

```bash
pytest            # SQLite en memoria: autenticación, cuentas, invitaciones/MFA, métricas y auditoría
```

La suite corre sobre SQLite en memoria (sin PostgreSQL) gracias a tipos
portables; el despliegue real usa PostgreSQL vía `ADMIN_DATABASE_URL`.

## Auditoría de cambios

### Cuentas de la consola

El rol admin gestiona cuentas con GET/POST `/api/v1/admin/users` y PATCH/DELETE `/api/v1/admin/users/{id}`. El listado admite `q`, `role`, `is_active`, `page` y `page_size`. La edición incluye nombre, correo, rol y estado. Se validan los dominios autorizados y correos duplicados.

La pantalla crea cuentas por invitación: POST sin contraseña crea una cuenta pendiente y encola un correo transaccional. SMTP y la clave de cifrado deben estar configurados. El titular elige la contraseña y confirma su autenticador antes de activar la cuenta. Por compatibilidad, POST conserva el campo opcional `password` para integraciones anteriores, pero la política de MFA se exige igualmente antes del acceso. Una cuenta pendiente no puede activarse mediante PATCH.

No se permite eliminar/desactivar la cuenta autenticada ni cambiar su propio rol, y se conserva al menos un administrador activo. DELETE elimina sesiones, invitaciones y configuraciones pendientes y conserva la auditoría. Una edición sin cambios no genera eventos. Esta ampliación requiere la migración `0002`: `alembic upgrade head`. Consulta [la guía de configuración y endpoints](docs/invitaciones-mfa.md). Pruebas: `pytest tests/test_staff.py tests/test_invitations_mfa.py tests/test_security_migration.py`.

Sistema → Auditoría está disponible para el rol admin. Registra cambios de negocio confirmados: crear, editar, eliminar, publicar, aprobar y activar o desactivar registros. No registra consultas, clics, navegación, controles de juegos ni operaciones técnicas de sesión.

Los cambios de juegos incluyen usuario, fecha y valores antes/después. Guardar un juego sin modificar datos no genera eventos. La auditoría se guarda en la misma transacción que el cambio: si se revierte la operación, también se revierte el evento.

GET /api/v1/admin/audit-log permite buscar y filtrar por usuario, acción, entidad y fechas, con paginación. GET /api/v1/admin/audit-log/export exporta hasta 5000 cambios. Ambas operaciones exigen rol admin. La vista y el CSV usan descripciones en español y nombres de módulos y registros, sin rutas HTTP. Los antiguos eventos de clics, solicitudes y autenticación quedan ocultos, sin borrarlos de la base de datos. El endpoint audit-events fue retirado.

No requiere migración. Se enmascaran credenciales y se protegen las celdas CSV contra fórmulas. Pruebas: pytest tests/test_audit.py. En Windows se incluye tzdata para America/Santiago.
## Estructura

```
app/
  config.py        Configuración (pydantic-settings, prefijo ADMIN_)
  database.py      Engine async + sesión por petición
  errors.py        Formato estándar de error {error:{code,message,details,request_id}}
  enums.py         Enumeraciones y matriz de permisos por rol
  security.py      Argon2id, JWT (access 15 min / refresh 12 h rotatorio), TOTP
  deps.py          get_current_admin, require(módulo, write=…)
  audit.py         Escritura de audit_log con enmascarado de campos sensibles
  periods.py       Resolución de PeriodKey en America/Santiago + buckets
  models.py        Tablas: staff, agregadas de métricas, ops, tickets, contenido,
                   marketplace, moderación, settings, legales, audit_log
  schemas/         Esquemas Pydantic por módulo
  routers/         Un router por módulo (prefijo común /api/v1/admin)
alembic/           Migración inicial
scripts/seed.py    Datos demo coherentes con el wireframe de la consola
tests/             pytest + httpx (asyncio)
```

## Decisiones de implementación

- **Estrategia mixta de datos**: los endpoints analíticos leen tablas agregadas
  (`metrics_daily_users`, `metrics_plan_snapshot`, `role_activity_window`,
  `feature_usage_window`, `ops_latency_window`…) que en producción alimentan
  jobs; el seed las puebla directamente. El estado operativo se lee de la foto
  que mantiene el monitor (`ops_component_state`).
- **En la matriz de adopción, `null` ≠ `0.0`**: null = la función no aplica al
  rol (celda gris); 0.0 = disponible pero sin uso.
- **Refresh tokens rotatorios** con detección de reutilización: reusar un token
  rotado revoca todas las sesiones del administrador.
- **Bloqueo optimista** en configuración (`version`) y validación del valor
  contra el JSON Schema de cada parámetro.
- **Auditoría de cambios de negocio confirmados**, con
  campos sensibles enmascarados.
- Push/webhooks requieren verificar sus integraciones. TTS y pipeline editorial de fase 10 tienen código nuevo, con worker deshabilitado por defecto y validación real pendiente: [contenido-editorial.md](docs/contenido-editorial.md).
