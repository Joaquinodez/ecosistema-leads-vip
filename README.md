# Ecosistema de Automatización IA Autónomo — NubeFlow Consulting

Entrega final: automatización de extremo a extremo para la gestión de leads de una consultora ficticia (NubeFlow Consulting).

Cada consulta que llega por correo (Gmail) o por webhook es analizada con IA. Si es una consulta comercial, se registra como lead en Airtable, se puntúa (VIP / Calificado / Descarte) y se redacta una respuesta. Los leads VIP no reciben respuesta hasta que un gerente la aprueba por correo (Human-in-the-loop). Todo error queda registrado y se alerta al administrador.

**Stack (las 4 categorías exigidas):**

| Categoría | Herramienta |
|---|---|
| Orquestador | n8n Cloud |
| Base de datos (memoria y registro) | Airtable (3 tablas vinculadas) |
| Procesamiento IA | Anthropic API: Claude Haiku 4.5 (clasificación) y Claude Sonnet 5 (redacción) |
| Canal de salida | Gmail (con Thread ID) |

## Enlaces obligatorios

- **Dashboard de control (Shared View de Airtable, solo lectura):** https://airtable.com/app3PoaFFyXzghbx2/shrD1YBckiE15GRWT/tblCqhr8ehXv8PyNG/viwmOKIVRHHUIq8Zf
  Vista de solo lectura de la base de datos ("Dashboard Publico"): oculta email, teléfono y mensaje de los leads. Desde el mismo enlace se pueden consultar las tablas Errores e Interacciones. Los datos de los leads son ficticios, de demostración.
- **JSON del flujo:** [`workflow/ecosistema_leads_vip_v2.json`](workflow/ecosistema_leads_vip_v2.json)
- **Evidencias (screenshots):** carpeta [`screenshots/`](screenshots/)

## Entregables y criterios de la rúbrica

| # | Criterio (20 % c/u) | Entregable |
|---|---|---|
| 1 | Mapa de arquitectura | [`docs/01_diagrama_arquitectura.pdf`](docs/01_diagrama_arquitectura.pdf) y [`docs/01b_componentes_arquitectura.pdf`](docs/01b_componentes_arquitectura.pdf) |
| 2 | Estructuras de datos documentadas | [`docs/02_manual_de_datos.pdf`](docs/02_manual_de_datos.pdf), [`schemas/airtable_schema.json`](schemas/airtable_schema.json) y [`schemas/integracion_json_schemas.json`](schemas/integracion_json_schemas.json) |
| 3 | Optimización de costos | [`docs/03_matriz_de_costos.pdf`](docs/03_matriz_de_costos.pdf) (+ [`matriz_costos.csv`](docs/matriz_costos.csv), [`matriz_costos_escenarios.csv`](docs/matriz_costos_escenarios.csv)) |
| 4 | Seguridad y resiliencia | [`docs/04_seguridad_y_resiliencia.pdf`](docs/04_seguridad_y_resiliencia.pdf) |
| 5 | Dashboard de control | Shared View de Airtable (enlace arriba) |

## Cómo funciona

1. **Disparo:** un correo nuevo en Gmail (solo mensajes no enviados por la propia cuenta) o un POST al webhook `/webhook/nuevo-lead-vip`.
2. **Validación y anti-duplicados:** se normalizan los datos, se exigen nombre y email válido y se busca el email en Airtable para no crear leads repetidos.
3. **Clasificación (Claude Haiku):** puntaje 0–100 con reglas y puntajes acotados. VIP ≥ 80, Calificado ≥ 50, Descarte < 50. Los umbrales viven en el nodo *Config*.
4. **Rama automática:** los leads no VIP reciben una respuesta estándar.
5. **Rama VIP con HITL:** Claude Sonnet redacta un borrador y el gerente recibe un correo con los botones *Aprobar y enviar* / *Rechazar*. El flujo espera hasta 48 h; si no hay respuesta, el lead queda como *Expirado*.
6. **Registro:** cada paso actualiza el campo *Estado* del lead (Pendiente → Procesado_por_IA → Esperando_Aprobacion → Aprobado_por_Humano / Rechazado → Completado) y guarda interacciones con el Thread ID de Gmail.

**Parámetros dinámicos (sin datos hardcodeados):** el nodo *Config: Parametros del Sistema* concentra correos del aprobador y del administrador, modelos, `max_tokens`, umbrales de score, tamaños de empresa y horas de espera. Los prompts usan variables del lead y de esa configuración.

**Filtro anti-bucle:** se excluyen los correos propios (`-from:me`) y los asuntos generados por el sistema (por ejemplo "Aprobacion requerida"), tanto en la consulta del trigger como en un nodo IF.

**Resiliencia:** cada fallo (API de IA, formato de respuesta, Airtable, Gmail, datos faltantes) pasa por un punto único de manejo que guarda una fila en la tabla *Errores*, marca el lead con `Error_API` cuando corresponde y alerta al administrador por correo.

## Estructura del repositorio

```
README.md
workflow/ecosistema_leads_vip_v2.json     Workflow n8n (49 nodos, credenciales sanitizadas)
schemas/airtable_schema.json              Esquema de la base (tablas, campos, estados)
schemas/integracion_json_schemas.json     JSON Schemas de las integraciones
docs/                                     PDFs de los entregables 1 a 4 y matrices de costos (CSV)
tests/plan_pruebas.md, casos_prueba.json  Prueba de estrés (9 casos)
screenshots/                              Evidencias del flujo, la base y el dashboard
```

## Prueba de estrés

Se ejecutaron 9 casos, incluidos caminos infelices. Detalle en [`tests/plan_pruebas.md`](tests/plan_pruebas.md).

| Caso | Resultado |
|---|---|
| T1 Correo real, lead Calificado | OK |
| T2 VIP aprobado por el gerente (HITL) | OK |
| T3 VIP rechazado por el gerente | OK |
| T4 Aprobación expirada | OK (verificado con la espera acortada temporalmente; el valor de producción es 48 h) |
| T5 Datos faltantes (webhook sin email) | OK (ejecución con datos fijados) |
| T6 Correo no comercial | OK (ejecución con datos fijados; descartado) |
| T7 Falla de la API de IA (401) | OK (error registrado y alerta) |
| T8 Lead duplicado | OK (una sola fila) |
| T9 Prompt injection | OK (score 5, Descarte) |

## Puesta en marcha

1. Importa `workflow/ecosistema_leads_vip_v2.json` en n8n (Workflows → Import from file).
2. Crea o asigna las credenciales: Gmail OAuth2, Airtable y Anthropic como *Header Auth* (**Name** = `x-api-key`, **Value** = la API key).
3. En el nodo **Config** cambia `aprobador_email` y `admin_email`.
4. Crea la base de Airtable según `schemas/airtable_schema.json` y actualiza los IDs de base y tablas en los nodos Airtable.
5. Publica el workflow y envía un correo de prueba.

## Notas técnicas

- Claude Sonnet 5 no acepta el parámetro `temperature` y su respuesta incluye un bloque `thinking` antes del bloque de texto; el flujo ya lo contempla (se lee el primer bloque de tipo `text`).
- No hay un workflow global de errores con Error Trigger; los errores se manejan por nodo y convergen en un punto único.
- Los tokens y el costo real se registran en la clasificación; los costos de redacción se estiman en la matriz de costos.
- Las tarifas de Anthropic de la matriz corresponden al 21-sep-2026.
