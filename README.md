# Proyecto Integrador - Agentes IA en n8n

**Alumno:** Federico Jorda  
**Checkpoint actual:** Módulo 4 - Integraciones Avanzadas e Interconexión de Sistemas

Este repositorio contiene la evolución del proyecto integrador desarrollado en n8n. El Checkpoint 4 parte del workflow del Módulo 3 y agrega integraciones reales con Gmail, HubSpot y Slack, manteniendo la memoria persistente en Airtable y la arquitectura multi-agente.

## Archivos principales

- `checkpoint1_federico_jorda.json`: checkpoint inicial del proyecto.
- `checkpoint4_federico_jorda.json`: workflow del Checkpoint 4.
- `docs/credentials.png`: evidencia de las credenciales conectadas en n8n.
- `docs/workflow-ejecucion.png`: evidencia del workflow completo ejecutado correctamente.

## Arquitectura del Checkpoint 4

```text
Gmail Trigger
  |
  v
IF - Anti Auto Reply
  | false
  v
Set - Payload Limpio
  |
  v
Airtable - Search Records
  |
  v
IF - Usuario existente
  |-----------------------|
  | true                  | false
  v                       v
Contexto existente     Crear memoria
  |                       |
  |-----------------------|
          |
          v
      AI Agent
          |
          v
        Switch
  |---------|---------|
  v         v         v
Analista  Redactor  Escalar humano
  |         |         |
  |---------|---------|
          |
          v
    Salida unificada
      |           |
      |           +--> HubSpot Lookup --> IF contacto existe
      |                    |                    |
      |                    +--------------------+
      |                             |
      |                             v
      |                  Crear/Actualizar contacto
      |                             |
      |                             v
      |                  Gmail - Crear Borrador HITL
      |                             |
      |                             v
      |                  Set - Payload Slack
      |                             |
      |                             v
      |                  Slack - #operaciones
      |
      v
Chat Memory Manager
  |
  v
IF - Más de 5 intercambios
  | true
  v
Basic LLM Chain + Structured Output Parser
  |
  v
Airtable - Create or Update
```

## Controles preventivos evaluados

### 1. IF anti auto-reply

El nodo inmediatamente posterior al Gmail Trigger corta el flujo si detecta correos automáticos. La condición contempla asuntos como:

- `Auto-reply`
- `Auto reply`
- `Out of office`
- `Undeliverable`

y remitentes `no-reply@` / `noreply@`.

La condición utilizada es:

```javascript
{{
  /auto[- ]?reply|out of office|undeliverable/i.test($json.Subject || '') ||
  /no-?reply@/i.test($json.From || '')
}}
```

La rama TRUE finaliza el procesamiento y la rama FALSE continúa.

### 2. Lookup antes de escribir en el CRM

Antes de crear o actualizar un contacto se ejecuta `HubSpot - Lookup Contacto`, buscando por email con límite 1.

Luego, `IF - Contacto Existe` comprueba:

```text
{{ $json.id }} is not empty
```

Esto hace visible el control anti-duplicados antes de la operación de escritura y reduce el riesgo de errores 409.

### 3. Human-in-the-loop con Gmail Draft

La salida de correo está configurada exclusivamente como:

```text
Resource: Draft
Operation: Create
```

El sistema crea un borrador en Gmail y no envía el mensaje automáticamente. La revisión y el envío final quedan bajo control humano.

### 4. Limpieza de payload

`Set - Payload Limpio` reduce el correo entrante a los datos necesarios:

```json
{
  "from": "remitente",
  "subject": "asunto",
  "body_text": "texto",
  "email": "correo",
  "session_id": "correo",
  "user_message": "contenido de la solicitud"
}
```

No se propagan adjuntos, binarios, HTML completo ni metadata innecesaria.

Antes de Slack se utiliza además `Set - Payload Slack`, que envía únicamente el mensaje y los campos mínimos necesarios.

## Integraciones externas

### Gmail

- Entrada: `Gmail Trigger - Message Received`.
- Salida: `Draft - Create`.
- Autenticación: OAuth2.
- Guardrail: el agente nunca envía el correo por sí mismo.

### HubSpot

- Recurso: Contact.
- Lectura: Search por email.
- Escritura: Create or Update.
- Autenticación: OAuth2.
- Control: lookup previo antes de la escritura.

### Slack

- Recurso: Message.
- Operación: Send.
- Canal: `#operaciones`.
- Autenticación: OAuth2.
- Recibe un payload previamente reducido.

## Memoria y contexto

Airtable mantiene la memoria persistente del proyecto con los campos:

- `Session_ID`
- `Fecha_actualizacion`
- `Resumen_consolidado`
- `Estado_del_caso`
- `Datos_clave`
- `Nombre_usuario`

En la entrada por Gmail se utiliza el email del remitente como identificador estable de sesión.

Cuando el historial supera los 5 intercambios, el workflow genera un resumen estructurado con:

```json
{
  "asunto_principal": "string",
  "puntos_clave": ["string"],
  "accion_requerida": "string"
}
```

y actualiza la memoria por `Session_ID`.

## Sistema multi-agente

El Manager clasifica cada solicitud dentro de una taxonomía cerrada:

- `VALIDAR_COTIZACION`
- `CREAR_BORRADOR`
- `ESCALAR_HUMANO`

Luego deriva la tarea al Worker correspondiente.

El workflow del Checkpoint 4 referencia los sub-workflows construidos en el Módulo 2:

- `M2 - Worker Analista Cotización`
- `M2 - Worker Redactor Correo`

Para ejecutar el flujo completo en otra instancia de n8n, esos sub-workflows deben existir y estar seleccionados nuevamente en los nodos Execute Workflow.

## Credenciales necesarias

El workflow requiere credenciales configuradas en n8n para:

- Gmail OAuth2
- HubSpot OAuth2
- Slack OAuth2
- Airtable Personal Access Token
- Groq

Las credenciales y secretos no están incluidos en el repositorio.

### Evidencia de credenciales

![Credenciales conectadas en n8n](docs/credentials.png)

## Evidencia de ejecución

![Workflow M4 ejecutado](docs/workflow-ejecucion.png)

## Prueba de regresión realizada

Se verificó manualmente:

- recepción de un correo real por Gmail;
- paso por el filtro anti auto-reply;
- reducción del payload;
- recuperación/creación de memoria en Airtable;
- clasificación del Manager;
- ejecución del Worker correspondiente;
- búsqueda previa del contacto en HubSpot;
- creación/actualización sin duplicados;
- creación del borrador en Gmail sin envío automático;
- publicación de la notificación en `#operaciones`;
- ejecución completa en verde en n8n.

También se corroboró en las herramientas externas que:

- el contacto existe una sola vez en HubSpot;
- la respuesta queda en Borradores de Gmail;
- la notificación llega al canal de Slack.

## Importación

1. Descargar `checkpoint4_federico_jorda.json`.
2. Abrir n8n.
3. Importar el archivo desde el menú de workflows.
4. Volver a seleccionar las credenciales propias.
5. Verificar/revincular los dos sub-workflows M2 en los nodos Execute Workflow.
6. Ejecutar pruebas manuales antes de activar el workflow en producción.

## Seguridad

Este repositorio no publica:

- contraseñas;
- access tokens;
- client secrets;
- API keys;
- secretos OAuth2.

El JSON conserva únicamente referencias/nombres de credenciales de n8n, no sus secretos.
