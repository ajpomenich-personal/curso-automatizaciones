[README.md](https://github.com/user-attachments/files/33386535/README.md)
# Checkpoint 1: Agente de observabilidad de correos de Airbnb

Workflow de n8n con un AI Agent en modo Tools Agent que lee los correos de Airbnb, los clasifica, los registra en un Google Sheets y envía un reporte de observabilidad con una acción sugerida.

- **Autor:** Alejandro Pomenich
- **Archivo:** `checkpoint1_alejandro_pomenich.json`
- **Actividad:** Configuración del "Tools Agent" y el Motor de Razonamiento Base

## Qué hace

Por cada correo de `automated@airbnb.com`, el agente:

1. Clasifica el tipo: `reserva_confirmada`, `cancelacion`, `pago` u `otro`.
2. Extrae los datos clave: código de reserva, huésped, propiedad, fechas y monto.
3. Registra una fila en la hoja Log, usando la herramienta `sheets_log`.
4. Redacta un "Reporte de observabilidad" con una acción sugerida.

Un nodo final de Gmail envía ese reporte por mail para supervisión humana.

| Tipo de correo | Acción sugerida |
|---|---|
| reserva_confirmada | Notificar equipo de limpieza |
| cancelacion | Notificar equipo de limpieza |
| pago | Chequear el banco |
| otro | Sin acción (o "Revisar manualmente" si faltan datos) |

## Arquitectura

```
Gmail Trigger → AI Agent (Tools Agent) → Send a message (Gmail)
                  ├─ Chat Model: OpenAI Chat Model (gpt-4.1)
                  └─ Tool: sheets_log (Google Sheets, Append Row)
```

| Componente | Configuración |
|---|---|
| Disparador | Gmail Trigger, filtro de remitente `automated@airbnb.com`, Simplify apagado para recibir el cuerpo completo del mail |
| AI Agent | Modo Tools Agent, prompt definido manualmente con el contenido del correo |
| Modelo | OpenAI Chat Model, `gpt-4.1` |
| Guardrail | `maxIterations = 7` (rango exigido: 5 a 10) |
| Herramienta | `sheets_log`, Google Sheets en operación Append Row, conectada lateralmente al agente |
| Observabilidad | Nodo Gmail "Send a message" con el output del agente |

### Decisión de diseño sobre el disparador

Se usa Gmail Trigger en lugar de Chat Trigger o Webhook. El correo de Airbnb cumple el rol del mensaje desestructurado de entrada: su cuerpo se mapea como prompt del agente.

## System Prompt

Está estructurado de forma modular dentro del nodo AI Agent:

1. **Rol:** agente de observabilidad de correos de Airbnb.
2. **Ámbito y restricciones:** solo procesa correos de `automated@airbnb.com`. No responde a huéspedes, no escribe al equipo de limpieza, no modifica nada en Airbnb, no inventa datos y no usa lenguaje inclusivo.
3. **Objetivo:** clasificar, registrar y redactar el reporte.
4. **Proceso, clasificación y reglas de herramientas:** orden obligatorio y una sola llamada a `sheets_log` por correo.
5. **Escalamiento:** si no puede clasificar o faltan datos clave, usa tipo `otro` y acción "Revisar manualmente".
6. **Salida y estilo:** formato fijo del reporte, en español neutro y sin emojis.

## Descripción de la herramienta

La descripción de `sheets_log` indica cuándo debe activarse de forma autónoma (todo correo de Airbnb, una vez por mail, después de clasificar y antes de redactar el reporte), qué columnas completa y qué hacer cuando falta un dato. Así se evita la "instrucción huérfana".

## Estructura de la hoja Log

La fila 1 debe tener exactamente estos encabezados:

`fecha_proceso | id_mail | fecha_mail | asunto | tipo | codigo_reserva | huesped | propiedad | monto_clp | accion_sugerida | razonamiento`

## Cómo importar y probar

1. En n8n, ir a **Workflows > Import from file** y elegir `checkpoint1_alejandro_pomenich.json`.
2. Configurar tus credenciales: **Gmail OAuth2**, **Google Sheets OAuth2** y **OpenAI**.
3. En el nodo `sheets_log`, elegir tu propio documento de Google Sheets con la hoja `Log` y los encabezados de arriba.
4. En el nodo `Send a message`, cambiar el destinatario por tu correo.
5. En Gmail Trigger, hacer **Fetch Test Event** con un correo de Airbnb en la bandeja.
6. Hacer clic en **Execute Workflow** y abrir el panel **Logs**.

### Qué verificar en la ejecución

- Recorrido completo en verde.
- `sheets_log` llamado una sola vez, con una fila nueva en Log.
- El modelo aparece con 2 llamadas: es el ciclo normal del agente (decidir, usar la herramienta, redactar la respuesta).
- Llega el correo de reporte, con la acción sugerida.

## Seguridad

El JSON incluye el ID del documento de Google Sheets y los IDs de las credenciales de n8n, pero no los secretos. Si se comparte el repositorio, conviene reemplazar esos valores y el correo personal por datos de ejemplo.
