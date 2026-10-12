# Arquitectura — v1

> Estado: **definida, no implementada** (2026-09-24). **En revisión desde el 2026-10-12**: los
> [requisitos](requisitos.md) se escribieron después de esta arquitectura, y el dueño reabrió el
> stack hasta cerrarlos. §Pendiente de alinear con los requisitos lista las diferencias conocidas.
> La v1 no tiene, a propósito, ni MCP, ni SDK de agentes, ni infraestructura más allá de un proceso
> Python y un modelo de lenguaje.

## Qué es

`incident-investigator` es un agente de IA que responde preguntas sobre incidentes de producción
leyendo los logs de un sistema observado. Por ejemplo: "¿qué pasó con el pedido 123?" o "¿por qué
falló anoche el webhook de pago?". Devuelve un informe donde cada afirmación cita la entrada de log
que la respalda.

Es un **sistema externo**. El sistema que observa, Revestimientos (una tienda en Laravel), nunca lo
importa, lo llama ni sabe que existe. Los dos sistemas comparten una sola cosa: un **contrato de
logs versionado**.

## Contexto del sistema

```
┌──────────────────────────────┐       contrato de logs v1      ┌───────────────────────────────┐
│ Revestimientos (observado)   │ JSON Schema + catálogo eventos │ incident-investigator         │
│ Laravel 12                   │ ─────────────────────────────▶ │ Python 3.12                   │
│                              │  archivos app-AAAA-MM-DD.jsonl │                               │
│ • request_id en cada línea   │  en un directorio              │ • lee solo ese directorio     │
│ • sin datos personales (se   │                                │ • sin base de datos, sin red  │
│   eliminan antes de escribir)│                                │   hacia el sistema observado  │
└──────────────────────────────┘                                └──────────────┬────────────────┘
                                                                               │ interfaz de chat model
                                                                  ┌────────────▼──────────────┐
                                                                  │ Modelo (intercambiable)   │
                                                                  │ por defecto: Gemini gratis│
                                                                  └───────────────────────────┘
```

**El dueño del contrato es el lado observado.**
- Vive en Revestimientos, en `docs/observabilidad/log-schema.v1.json`.
- Hay una copia vendoreada acá en `contracts/revestimientos/`, con el commit del que se tomó.
- Revestimientos valida cada línea que emite contra el schema en su propia suite de tests.
- Este proyecto valida sus fixtures contra la copia vendoreada.
- Si cualquiera de los dos lados se desvía de forma incompatible, falla un test.

**Regla de compatibilidad (la fija el contrato).**
- Los eventos nuevos y los atributos opcionales nuevos son compatibles.
- El investigador conserva los eventos desconocidos como evidencia genérica e ignora los atributos
  desconocidos.
- Quitar, renombrar o cambiar el tipo de algo exige un `schema_version` nuevo, y el investigador lo
  rechaza hasta soportarlo.

**Advertencia del contrato.** Que falte un evento no prueba que el hecho no ocurrió: el productor
puede perder líneas antes que romper un request. Los informes no deben tratar la ausencia como
prueba.

## Por dentro del investigador

Estructura hexagonal: la lógica de investigación depende de puertos, nunca de archivos ni de
proveedores de modelos.

```
src/incident_investigator/
├── contract/     LogEntry, EventCatalog: modelos Pydantic generados desde el schema o validados contra él
├── ports/        LogSource (consultar logs), ChatModel (el puerto es BaseChatModel de LangChain)
├── adapters/     FileLogSource (directorio jsonl), factory de modelos (google-genai por defecto | ollama | anthropic)
├── graph/        StateGraph de LangGraph: nodos y estado
├── report/       modelo Report → renderers de Markdown y JSON
└── cli.py        `investigate --logs DIR "pregunta"`
eval/
├── scenarios/    una carpeta por incidente: logs de fixture, pregunta y hallazgos esperados
└── run.py        corre cada escenario contra un modelo, lo puntúa y escribe una tabla de resultados
```

### Puerto `LogSource`

```python
class LogQuery(BaseModel):
    request_ids: list[str] = []
    order_ids: list[int] = []
    payment_ids: list[str] = []
    events: list[str] = []
    levels: list[Level] = []
    since: datetime | None = None
    until: datetime | None = None
    limit: int = 500

class LogSource(Protocol):
    def query(self, q: LogQuery) -> list[LogEntry]: ...
    def schema_version(self) -> str: ...
```

La v1 tiene un solo adaptador, `FileLogSource`, que:
- lee `app-*.jsonl` de un directorio, línea por línea;
- valida cada línea contra el contrato, tolerando los cambios aditivos;
- se niega a correr si encuentra un `schema_version` que no soporta.

Las fuentes posteriores (la API de logs de Render, Grafana Loki) son adaptadores nuevos detrás del
mismo puerto. El grafo no cambia.

### El grafo de investigación (LangGraph)

```
            ┌─────────────────────┐
pregunta ──▶│ 1. entender         │  LLM → IncidentQuery (salida estructurada)
            └─────────┬───────────┘   identificadores, ventana de tiempo, síntoma
                      ▼
            ┌─────────────────────┐
            │ 2. reunir evidencia │  determinístico: consultas a LogSource + saltos de correlación
            └─────────┬───────────┘   order_id → request_ids → todas las líneas de esos requests
                      ▼               payment_id ↔ order_id
            ┌─────────────────────┐
            │ 3. línea de tiempo  │  determinístico: ordenar, deduplicar, compactar
            └─────────┬───────────┘
                      ▼
            ┌─────────────────────┐   LLM → hipótesis, cada una citando ids de evidencia,
            │ 4. analizar         │◀─┐ o un pedido de más evidencia
            └─────────┬───────────┘  │
        ¿hace falta más evidencia? ──┘ (máximo 2 rondas extra, volviendo por el nodo 2)
                      ▼
            ┌─────────────────────┐   determinístico: cada cita tiene que existir en el conjunto
            │ 5. verificar        │   de evidencia; lo que no tiene respaldo se descarta o se
            └─────────┬───────────┘   degrada; si falla → vuelve a 4 una vez (reflexión)
                      ▼
            ┌─────────────────────┐
            │ 6. informar         │  Report (Pydantic) → report.md + report.json
            └─────────────────────┘
```

**Regla de diseño: el modelo decide qué buscar y qué significa; el código busca y verifica.** Los
nodos 2, 3 y 5 son Python plano. Eso mantiene al agente:
- reproducible;
- testeable sin modelo;
- usable con modelos locales chicos, que no son confiables llamando herramientas libremente
  (ADR-002).

### Informe

Secciones fijas, en este orden:
1. **Síntoma**.
2. **Línea de tiempo**.
3. **Evidencia**: cada entrada citada por archivo, `timestamp` y `request_id`.
4. **Hipótesis**: cada una con confianza alta, media o baja, y la evidencia que la respalda.
5. **Causa raíz**: solo si la evidencia es concluyente; si no, lo dice.
6. **Próximos pasos**.

Cuando los logs no tienen nada relevante, el informe dice "no se encontró evidencia" y lista qué se
buscó. No completa los huecos con suposiciones.

### Conocimiento del dominio sin RAG

La v1 no recupera documentos (ADR-004). Lo que el modelo sabe del sistema observado sale del
**propio contrato**:
- Cada evento del catálogo tiene una descripción escrita por el lado observado, que dice cuál es el
  comportamiento esperado.
- Por ejemplo, `order.stock_negative` es un **estado conocido y aceptado** (la tienda vende por
  encima del stock y repone desde el fabricante), no un bug.
- Esas descripciones van en el prompt de análisis.

## Límites de confianza

| Límite | Regla |
|---|---|
| Datos personales | El sistema observado los elimina **antes** de escribir la línea. El investigador nunca los ve, así que lo que ve el modelo ya viene limpio. Es también lo que hace aceptable el plan gratuito de Gemini, que puede usar los prompts para mejorar los productos de Google (ADR-003) |
| Contenido de los logs → prompt | Los valores de los logs son **datos no confiables**. Algunos vienen de fuera de la tienda, como cabeceras (el `x-request-id` de MercadoPago) o referencias de pago. Entran al prompt solo dentro de bloques de evidencia delimitados y tipados, nunca como instrucciones. El contrato limita el largo de los textos de origen externo |
| Salida del modelo | Se parsea a modelos Pydantic. Lo que no valida se reintenta una vez y después se informa como falla de análisis; nunca pasa como texto libre |
| Efectos secundarios | Ninguno. El investigador lee un directorio y escribe un informe |

## Evaluación (lo que lo hace pieza de portfolio)

Cada escenario de `eval/scenarios/` es un incidente reproducido a propósito en Revestimientos. Las
líneas de log que resultan se commitean como fixtures, y la respuesta conocida se escribe antes:

| Escenario | Pregunta | Esperado |
|---|---|---|
| Firma de webhook inválida | "¿MercadoPago avisó del pago X?" | `webhook.signature_invalid`, causa identificada |
| Falla de la preferencia de pago | "El cliente dice que el botón de pagar no hizo nada — pedido N" | `checkout.mp_preference_failed` |
| Pago rechazado | "¿Por qué el pedido N sigue pendiente?" | `webhook.payment_not_approved` con su estado |
| Trabado después del redirect de pago | "El pedido N nunca se pagó" | `checkout.payment_started` sin nada después, informado como **"abandonó en MercadoPago o se perdió el webhook: los logs no pueden distinguirlo"** (un límite declarado del contrato v1) |
| Checkout rechazado por stock | "No pude comprar" (ventana de tiempo) | `checkout.rejected`, motivo `dominio` |
| Stock negativo | "El stock del producto P quedó negativo" | reconocido como comportamiento **previsto**, no como bug |
| Pedido inexistente | "¿Qué pasó con el pedido 99999?" | "no se encontró evidencia" |

Métricas por modelo:
- precisión de causa raíz;
- **validez de citas** (tiene que ser 100%);
- recall de evidencia;
- **identificadores inventados** (tiene que ser 0);
- respuestas "sin evidencia" correctas;
- latencia y costo.

La misma suite corre contra cada modelo configurado, y la tabla comparativa va en el README.

## Stack

| Tema | Elección |
|---|---|
| Lenguaje | Python 3.12 |
| Empaquetado | `uv` |
| Orquestación | LangGraph |
| Interfaz de modelo | Chat models de LangChain: `langchain-google-genai` (**por defecto, plan gratuito de Gemini**), `langchain-ollama`, `langchain-anthropic` (extra opcional) |
| Modelos de datos | Pydantic v2 |
| Validación del schema | `jsonschema` |
| CLI | Typer |
| Calidad | pytest, ruff, mypy (strict) |

## Lo que la v1 deliberadamente no hace

No tiene nada de esto:
- MCP;
- un SDK de agentes;
- vector store ni RAG;
- una interfaz más allá de la CLI;
- corridas programadas o disparadas por alertas;
- acceso a bases de datos;
- infraestructura hosteada.

Cada una es una versión posterior con su propia ADR (ver `roadmap.md`).

## Pendiente de alinear con los requisitos (2026-10-12)

Diferencias conocidas entre esta arquitectura y [`requisitos.md`](requisitos.md). Se resuelven al
revisar el stack, no antes.

1. **`LogQuery` no puede filtrar por producto, por sesión ni por `external_reference`.**
   - CU8 y CU9 necesitan `product_id`, que en el contrato aparece como `product_id` en
     `order.stock_negative` y en `lines[]`, y como `subject_id` en los eventos `product.*`.
   - CU6 necesita `session_ref` (spec observabilidad-02 de Revestimientos, OBS-13).
   - CU1 necesita `external_reference`: un `checkout.returned` con `order_id: null` se ata al pedido
     solo por ese valor, que viene sin firma (OBS-12).
   - **Saltos de correlación nuevos en el nodo 2**:
     - para CU6: `order.created` → su `request_id` → su `http.request.session_ref` → los otros
       `http.request` con el mismo `session_ref` → sus `request_id` → los `checkout.rejected`;
     - para CU1: `checkout.returned` sin `order_id` → `external_reference`, tratado como vínculo
       débil.
2. **Consumo de logs.**
   - Los requisitos fijan un montaje de solo lectura en un contenedor, de un directorio que contiene
     solo el contrato (`storage/logs/contrato/`, OBS-10).
   - Hoy la arquitectura describe una CLI que recibe una ruta.
   - El lector tiene que tolerar una última línea incompleta.
3. **Evento nuevo `checkout.returned`** (OBS-12): el escenario "trabado después del redirect de
   pago" se parte en dos: el cliente volvió (se distingue) o no volvió (sigue sin distinguirse).
4. **Veredicto accionable** (regla R8): el informe suma quién tiene que actuar (cliente, tienda o
   desarrollador).
5. **Escenarios de evaluación**: los 7 escenarios de la tabla cubren 6 de los 11 casos de uso. Faltan CU4
   (monto distinto), CU5 (pagó sobre un pedido cancelado), CU7 (transferencia sin confirmar), CU8
   (precio) y CU10 (sitio lento).
6. **ADR-003**: la sección *Decisión* lista Ollama y Anthropic como proveedores de v1, y Gemini
   aparece después como proveedor por defecto. Se unifica al revisar el stack.
