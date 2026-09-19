# Webhook Service - Especificación de Requisitos

## 📋 Información General

| Aspecto | Descripción |
|---------|-------------|
| **Nombre del Proyecto** | Webhook Service |
| **Objetivo** | Servicio web para recepción, almacenamiento y visualización de webhooks en tiempo real |
| **Stack Tecnológico** | NestJS (Backend), Angular (Frontend), Prisma ORM, Bull Queue, SQLite (dev), SQL Server (prod) |
| **Fecha de Creación** | 2026-09-19 |

---

## 🎯 Visión General

El sistema permite a usuarios crear identificadores únicos (UUIDs) con alias descriptivos para recibir solicitudes POST (webhooks) y visualizar los datos recibidos en tiempo real a través de una interfaz web moderna. Los datos se almacenan durante 30 días y se pueden filtrar, paginar y ordenar. El sistema soporta grandes volúmenes de transacciones mediante colas de mensajes.

---

## 📖 Requisitos Funcionales

### RF-1: Generación de UUIDs

**GIVEN** el usuario está en la página principal  
**WHEN** hace clic en el botón "Generar nuevo UUID"  
**THEN** se abre un formulario para ingresar un alias  
**AND** el sistema genera un UUID v4 único  
**AND** se guarda en la base de datos con:
- UUID (único)
- Alias (obligatorio, máximo 50 caracteres alfanuméricos + especiales)
- Fecha/Hora de creación
- Estado: activo

**AND** el usuario es redirigido a la página del UUID generado

---

### RF-2: Listado de UUIDs en Página Principal

**GIVEN** el usuario accede a la página principal  
**THEN** se muestra una tabla con todos los UUIDs generados  
**AND** están ordenados de más reciente a más antiguo  
**AND** cada fila contiene:
- UUID (copiable)
- Alias
- Fecha de creación
- Cantidad de mensajes recibidos
- Botones: Ver Detalles, Editar Alias, Eliminar

---

### RF-3: Editar Alias de UUID

**GIVEN** el usuario hace clic en "Editar Alias"  
**WHEN** ingresa un nuevo alias (máximo 50 caracteres, único, alfanumérico + especiales)  
**THEN** el sistema valida que sea único  
**AND** se actualiza en la base de datos  
**AND** se muestra notificación de éxito

---

### RF-4: Eliminar UUID y sus Datos

**GIVEN** el usuario hace clic en "Eliminar"  
**WHEN** se abre modal de confirmación pidiendo escribir el alias exacto  
**AND** el usuario escribe correctamente el alias  
**THEN** se eliminan:
- El registro de `webhook_uuids`
- Todos los mensajes asociados en `webhook_messages`

**AND** se muestra notificación de eliminación exitosa

---

### RF-5: Recibir Webhooks POST

**GIVEN** un cliente externo hace una solicitud POST a `/webhook/:uuid`  
**WHEN** el UUID existe  
**AND** el payload es válido (máximo 3MB)  
**THEN** el sistema:
- Acepta cualquier estructura JSON (sin validación)
- Obtiene la IP del header `X-Forwarded-For`
- Coloca el mensaje en la cola RabbitMQ/Bull
- Responde con HTTP 201 Created

**AND** si el UUID no existe, responde HTTP 404

---

### RF-6: Procesamiento de Mensajes desde Cola

**GIVEN** un mensaje está en la cola  
**WHEN** el consumer procesa el mensaje  
**THEN** intenta guardar en `webhook_messages`:
- webhook_uuid (FK)
- json_data (el payload recibido)
- received_at (timestamp)
- source_ip (IP completa)

**AND** si falla, reintentar hasta 3 veces  
**AND** después de 3 fallos, descartar el mensaje (sin Dead Letter Queue)  
**AND** si se guarda exitosamente, emitir evento WebSocket a los clientes conectados

---

### RF-7: Visualización de Mensajes Recibidos

**GIVEN** el usuario accede a la página de detalle de un UUID  
**WHEN** carga la página  
**THEN** se muestra:
- Información del UUID y Alias
- Dashboard con estadísticas
- Tabla de mensajes recibidos

**AND** la tabla contiene:
- Timestamp (más reciente primero)
- Source IP
- JSON recibido (con arreglos expandidos como columnas)
- Botón: Ver JSON completo en modal

**AND** está paginada (cantidad de registros por página: configurable)

---

### RF-8: Ordenamiento y Paginación

**GIVEN** el usuario está viendo la tabla de mensajes  
**WHEN** la página carga  
**THEN** está ordenada por fecha/hora descendente (más reciente primero)  
**AND** muestra paginación  
**AND** se puede navegar entre páginas  
**AND** el tamaño de página es configurable en frontend

---

### RF-9: Filtro por Rango de Fechas

**GIVEN** el usuario está en la página de detalle del UUID  
**WHEN** utiliza los filtros de fecha "Desde" y "Hasta"  
**THEN** la tabla se actualiza mostrando solo mensajes en ese rango  
**AND** el dashboard también filtra sus estadísticas  
**AND** se aplica a la tabla y resumen simultáneamente

---

### RF-10: Dashboard de Estadísticas

**GIVEN** el usuario está en la página de detalle del UUID  
**THEN** se muestra un dashboard con:
- Total de solicitudes recibidas
- Solicitudes en las últimas 24 horas
- Solicitudes en las últimas 7 días
- Solicitudes en el período filtrado

**AND** todas estas estadísticas respetan los filtros de rango de fechas aplicados

---

### RF-11: Visualización en Tiempo Real (WebSocket)

**GIVEN** el usuario tiene la página de detalle del UUID abierta  
**WHEN** se recibe un nuevo mensaje en ese UUID  
**THEN** el sistema:
- Emite evento WebSocket a los clientes conectados
- Se añade la nueva fila a la tabla automáticamente
- Se actualiza el dashboard sin necesidad de recargar
- El usuario ve la actualización en tiempo real

**AND** si el usuario cierra la página, se desconecta del WebSocket

---

### RF-12: Visualización de Arreglos Expandidos

**GIVEN** un JSON recibido contiene arreglos  
**WHEN** se visualiza en la tabla  
**THEN** cada arreglo se muestra como una columna separada  
**AND** el contenido se expande/colapsa según sea necesario

---

### RF-13: Limpieza Automática de Datos Antiguos

**GIVEN** se ejecuta el job de limpieza (diariamente)  
**WHEN** revisa los registros en `webhook_messages`  
**THEN** elimina todos los mensajes con `received_at` > 30 días  
**AND** verifica si hay UUIDs sin mensajes en los últimos 30 días  
**AND** si es así, elimina también el registro en `webhook_uuids`

---

---

## 📊 Requisitos No Funcionales

### RNF-1: Rendimiento

**GIVEN** un UUID recibe múltiples solicitudes simultáneamente  
**WHEN** se procesan  
**THEN** el sistema debe manejar sin saturación gracias a la cola  
**AND** máximo 3 reintentos antes de descartar  
**AND** tiempo de respuesta HTTP 201 < 100ms

---

### RNF-2: Tamaño de Payload

**GIVEN** un cliente envía un webhook  
**WHEN** el payload supera 3MB  
**THEN** el sistema rechaza con HTTP 413 Payload Too Large

---

### RNF-3: Disponibilidad de Datos

**GIVEN** datos almacenados  
**WHEN** se consultan  
**THEN** están disponibles por 30 días desde su recepción  
**AND** después se eliminan automáticamente

---

### RNF-4: Acceso

**GIVEN** un usuario tiene un UUID  
**WHEN** accede a la página del UUID  
**THEN** no requiere autenticación adicional  
**AND** cualquiera con el UUID puede verlo (seguridad por obscuridad)

---

### RNF-5: Actualización en Tiempo Real

**GIVEN** el usuario está en la página de detalles  
**WHEN** se recibe un nuevo mensaje  
**THEN** la actualización en UI ocurre en < 1 segundo  
**AND** sin necesidad de recargar la página

---

### RNF-6: Interfaz de Usuario

**GIVEN** el usuario accede a la aplicación  
**THEN** la interfaz tiene:
- Diseño profesional
- No responsive (desktop only)
- Dark mode como plus (opcional)
- Colores coherentes y tipografía clara

---

### RNF-7: Persistencia de Datos

**GIVEN** el servidor se reinicia  
**WHEN** se vuelve a iniciar  
**THEN** los datos persisten correctamente  
**AND** no hay pérdida de información

---

---

## 🗄️ Modelo de Datos

### Tabla: `webhook_uuids`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | INT | PRIMARY KEY, AUTO_INCREMENT | Identificador único en BD |
| uuid | VARCHAR(36) | UNIQUE, NOT NULL | UUID v4 generado |
| alias | VARCHAR(50) | UNIQUE, NOT NULL | Nombre descriptivo |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Fecha/hora de creación |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Última actualización |
| status | VARCHAR(20) | DEFAULT 'active' | Estado (active, archived, etc.) |

---

### Tabla: `webhook_messages`

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| id | INT | PRIMARY KEY, AUTO_INCREMENT | Identificador único en BD |
| webhook_uuid | VARCHAR(36) | NOT NULL, FK | Referencia al UUID |
| json_data | LONGTEXT | NOT NULL | Payload JSON recibido |
| received_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Timestamp de recepción |
| source_ip | VARCHAR(45) | NOT NULL | IP completa del cliente |

**Índices:**
- `FK webhook_uuid → webhook_uuids(uuid) ON DELETE CASCADE`
- `INDEX (webhook_uuid, received_at DESC)`

---

---

## 🔌 API REST Endpoints

### POST /webhook/:uuid

**Descripción:** Recibir un webhook

**Parámetros:**
- `uuid` (path): UUID válido en el sistema

**Headers:**
- `X-Forwarded-For`: IP del cliente
- `Content-Type`: application/json

**Body:** JSON válido, máximo 3MB

**Respuestas:**
- `201 Created`
- `404 Not Found`
- `413 Payload Too Large`
- `400 Bad Request`

---

### GET /uuids

**Descripción:** Obtener listado de todos los UUIDs

**Query Parameters:**
- `page` (opcional): Número de página (default: 1)
- `limit` (opcional): Registros por página (default: 10)

**Respuesta:** `200 OK`

---

### POST /uuids

**Descripción:** Crear un nuevo UUID

**Body:**
```json
{
  "alias": "My-Webhook-Service"
}
```

**Validaciones:**
- `alias`: Obligatorio, máximo 50 caracteres, único

**Respuesta:** `201 Created`

---

### PATCH /uuids/:uuid

**Descripción:** Actualizar alias de un UUID

**Body:**
```json
{
  "alias": "Updated-Alias"
}
```

**Respuesta:** `200 OK`

---

### DELETE /uuids/:uuid

**Descripción:** Eliminar un UUID y todos sus mensajes

**Respuesta:** `204 No Content`

---

### GET /uuids/:uuid/messages

**Descripción:** Obtener mensajes de un UUID

**Query Parameters:**
- `page` (opcional)
- `limit` (opcional)
- `fromDate` (opcional): ISO 8601
- `toDate` (opcional): ISO 8601

**Respuesta:** `200 OK`

---

### GET /uuids/:uuid/stats

**Descripción:** Obtener estadísticas de un UUID

**Query Parameters:**
- `fromDate` (opcional)
- `toDate` (opcional)

**Respuesta:** `200 OK`

---

---

## 🔌 WebSocket Events

### Cliente → Servidor

#### `connect-uuid`
Conectarse a los eventos de un UUID específico

#### `disconnect-uuid`
Desconectarse de un UUID

---

### Servidor → Cliente

#### `message-received`
Nuevo mensaje recibido en el UUID

#### `uuid-updated`
UUID fue actualizado (alias, estado, etc.)

#### `uuid-deleted`
UUID fue eliminado

---

---

## 🔧 Configuración por Entorno

### Desarrollo Local

```env
DATABASE_URL="file:./dev.db"
DATABASE_PROVIDER="sqlite"
QUEUE_PROVIDER="bull"
REDIS_URL="redis://localhost:6379"
WEBSOCKET_URL="ws://localhost:3000"
API_PORT=3000
```

### Producción

```env
DATABASE_URL="Server=sql-server:1433;Database=webhook_service;..."
DATABASE_PROVIDER="sqlserver"
QUEUE_PROVIDER="rabbitmq"
RABBITMQ_URL="amqp://rabbitmq:5672"
WEBSOCKET_URL="wss://webhook.example.com"
API_PORT=3000
```

---

**Versión:** 1.0  
**Última Actualización:** 2026-09-19
