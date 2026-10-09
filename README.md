# SYNOC · Agente de Triaje NOC

Proyecto integrador del curso de agentes de IA con n8n. Cada checkpoint parte del `.json` anterior y lo extiende.

| Hito | Archivo | Estado |
|---|---|---|
| M1 · Agente base | [`checkpoint1_edgar_arcia.json`](checkpoint1_edgar_arcia.json) | Entregado |
| M2 · Multi-agente | — | Pendiente |
| M3 · Memoria | — | Pendiente |

## Checkpoint 1: agente base y motor de razonamiento

El agente recibe reportes de fallas de red escritos en lenguaje libre (técnicos de terreno, operadores, supervisores), los clasifica por severidad P1–P4, registra el incidente en Google Sheets y avisa al supervisor por Gmail con el recorrido completo de la ejecución.

```
Chat de entrada ──► Agente de Triaje NOC ──┬──► Armar reporte de observabilidad ──► Enviar log por Gmail
 (Chat Trigger)     (AI Agent · Tools Agent) │
                     ▲            ▲          └──► Respuesta al chat
       Anthropic Chat Model   Registrar incidente
        (Claude Sonnet 5.5)      (Google Sheets Tool)
```

### Cómo cumple cada requisito

| Requisito | Dónde está |
|---|---|
| Trigger | `Chat de entrada` (Chat Trigger). El mensaje llega sin estructura. |
| AI Agent en modo Tools Agent | `Agente de Triaje NOC`, campo *Agent = Tools Agent*. |
| Modelo con nodo nativo | `Anthropic Chat Model` conectado al puerto *Chat Model*. |
| Guardrail de iteraciones | *Options → Max Iterations = 8* (rango pedido: 5 a 10). |
| System Prompt modular | *Options → System Message*: Rol → Ámbito → Objetivo → Reglas y restricciones → Escalamiento → Formato de salida. |
| Sin lenguaje inclusivo | Regla explícita en el System Message (masculino genérico, sin "x", "@", "e" ni desdoblamientos). |
| Herramienta lateral | `Registrar incidente` (Google Sheets Tool, *append*) conectada al puerto *Tool* del agente, no a la cadena principal. |
| Descripción semántica extensa | *Tool Description* del nodo: cuándo usarla (4 condiciones + 6 casos de ejemplo) y cuándo no. Cada columna se llena con `$fromAI()` y su propia descripción. |
| Sin bloques lineales antes del agente | No hay IF ni Switch: el trigger va directo al agente, que decide solo si usa la herramienta. |
| Observabilidad | `Return Intermediate Steps = ON` → `Armar reporte de observabilidad` arma el Execution Log (herramientas usadas, entradas, resultados, iteraciones sobre el máximo) → `Enviar log por Gmail`. |

### Planilla de incidentes

Hoja de Google Sheets llamada `Incidentes` con estos encabezados en la fila 1 (también en [`plantilla-incidentes.csv`](plantilla-incidentes.csv)):

`ticket_id, fecha_hora, sitio_nodo, especialidad, sintoma, severidad, clientes_afectados, hipotesis_causa, accion_sugerida, escalamiento, reportado_por, estado`

`ticket_id`, `fecha_hora` y `estado` los pone el flujo; el resto lo completa el modelo.

### Importar y probar

1. En n8n: *Workflows → Import from File* → `checkpoint1_edgar_arcia.json`.
2. Credenciales: *Anthropic API* en el modelo, *Google Sheets OAuth2* en `Registrar incidente` (elegir planilla y hoja) y *Gmail OAuth2* en `Enviar log por Gmail`.
3. Abrir el chat del flujo y probar:
   - Con herramienta: `Se cayó el sitio ANT-034 en Antofagasta, sin 4G ni 5G desde las 14:10. Reporta Juan Pérez, técnico de terreno.`
   - Sin herramienta: `Hola, ¿qué tipo de incidentes puedes registrar?`

Con el primer mensaje el agente registra una fila (P1) y llega el correo con el log. Con el segundo responde sin usar la herramienta, y el correo lo deja indicado.

> El JSON del repo lleva solo datos de ejemplo: el correo del supervisor (`supervisor.noc@ejemplo.com`) y la URL de la planilla son de referencia y no incluye credenciales. Al importarlo hay que reemplazarlos por los propios.
