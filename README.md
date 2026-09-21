# Ecosistema de Automatización IA Autónomo — NubeFlow Consulting

Caso de uso: cada correo nuevo que llega al Gmail de la empresa es evaluado por Claude; si es una consulta comercial relevante, se guarda como lead en Airtable, se puntúa (VIP / Calificado / Descarte), se redacta una respuesta y, solo para leads VIP, un humano aprueba el envío (único punto HITL).

**Stack:** n8n Cloud · Airtable · Anthropic (Claude Haiku 4.5 y Sonnet 5) · Gmail

## Estructura del repositorio

```
README.md
workflow/ecosistema_leads_vip_v2.json     Workflow n8n (49 nodos, credenciales sanitizadas)
schemas/airtable_schema.json              Esquema de la base (tablas, campos, estados)
schemas/integracion_json_schemas.json     JSON Schemas de las integraciones
docs/01_diagrama_arquitectura.pdf         Entregable 1
docs/01b_componentes_arquitectura.pdf     Componentes, flujo de estados, Thread ID
docs/02_manual_de_datos.pdf               Entregable 2
docs/03_matriz_de_costos.pdf              Entregable 3 (+ CSV)
docs/04_seguridad_y_resiliencia.pdf       Entregable 4
tests/plan_pruebas.md, casos_prueba.json  Prueba de estrés (9 casos)
screenshots/                              (a completar por el autor)
```

## Puesta en marcha

1. Importa `workflow/ecosistema_leads_vip_v2.json` en n8n (Workflows → Import from file).
2. Crea/asigna credenciales: Gmail OAuth2, Airtable, y Anthropic como *Header Auth* (**Name** = `x-api-key`, **Value** = tu API key).
3. En el nodo **Config** cambia `aprobador_email` y `admin_email` (están como `CAMBIAR@tu-empresa.com`).
4. Crea la base de Airtable según `schemas/airtable_schema.json` y actualiza los IDs de base/tablas en los nodos Airtable.
5. Publica el workflow y envía un correo de prueba.

## Checklist de entregables (20 % cada uno)

| # | Entregable | Estado |
|---|---|---|
| 1 | Diagrama de arquitectura (PDF) | Listo: `docs/01_*.pdf` |
| 2 | Manual de datos (esquema Airtable + JSON Schemas) | Listo: `docs/02_*.pdf` + `schemas/` |
| 3 | Matriz de costos por modelo y tarea | Listo: `docs/03_*.pdf` |
| 4 | Seguridad y resiliencia | Listo: `docs/04_*.pdf` |
| 5 | Dashboard público con KPIs y tasa de error | Listo: vista pública de solo lectura de Airtable (sin email, teléfono ni mensaje): https://airtable.com/app3PoaFFyXzghbx2/shr3QtQIQHdjN4MEe . La Interface con KPIs agregados existe en la base, pero publicarla requiere plan Team |

## Pendiente por el autor

- Hecho: enlace público del dashboard (vista de solo lectura): https://airtable.com/app3PoaFFyXzghbx2/shr3QtQIQHdjN4MEe
- Tomar capturas (workflow, ejecuciones, Airtable, dashboard) en `screenshots/`.
- Verificar T4 (aprobación expirada), que requiere esperar 48 h. T2, T3, T8 y T9 ya fueron verificados con datos reales.
- Grabar el video de 3 minutos.

## Guion del video (3 min, sin mostrar credenciales)

1. 0:00–0:30 Problema y arquitectura (diagrama).
2. 0:30–1:15 Enviar un correo real y ver la ejecución en n8n.
3. 1:15–2:00 Fila creada en Airtable; punto HITL con correo de aprobación.
4. 2:00–2:30 Camino infeliz: error registrado y alerta al admin.
5. 2:30–3:00 Dashboard público, costos y cierre.
Oculta las credenciales de n8n y las API keys antes de grabar.

## Limitaciones conocidas

- No hay workflow global de errores (Error Trigger); los errores se manejan por nodo.
- Los tokens/costo se registran solo para el scoring (T2); T1 y T3 son estimaciones en la matriz de costos.
- Fuente del lead por correo queda como "Otro" (el select no tiene opción "Correo").
- Solo T4 (expiración a 48 h) sigue sin verificar. En v2 se corrigió que Sonnet 5 rechaza temperature y devuelve un bloque thinking antes del texto.
- Tarifas de Anthropic consultadas el 21-sep-2026; verifícalas antes de entregar.
