# ADR-003 — Modelo de lenguaje intercambiable

- **Estado**: **aceptada (2026-09-24)**. El proveedor por defecto se decidió el mismo día: **API de
  Gemini, plan gratuito**. **En revisión desde el 2026-10-12**, junto con el resto del stack (ver
  [requisitos](../requisitos.md)).

## Contexto

El presupuesto del dueño es solo de planes gratuitos. El modelo tiene que ser reemplazable, tanto
para seguir siendo gratis como para comparar calidad en la suite de evaluación.

La máquina de desarrollo (WSL2) expone hoy **2 GB de RAM y ninguna GPU**:
- No alcanza para correr un modelo de 7–8B en local con Ollama.
- Alcanza para modelos de 1–3B, cuyo razonamiento es débil para esta tarea.

## Decisión

- El grafo depende de `BaseChatModel` de LangChain. Una factory arma el modelo concreto a partir de
  la configuración (`provider`, `model`, `temperature=0`).
- Toda llamada al modelo usa **salida estructurada** (Pydantic), así que los proveedores son
  intercambiables a nivel de salida.
- Proveedores soportados en v1: **Ollama** (local, gratis) y **Anthropic** (hosteado, pago, extra
  opcional, usado para comparar calidad con un tope de gasto explícito).

## Proveedor por defecto: plan gratuito de la API de Gemini (decisión del dueño, 2026-09-24)

La máquina no puede correr un modelo local útil (ver Contexto), así que el proveedor por defecto es
un plan gratuito hosteado:

- **Integración**: `langchain-google-genai`, con la clave leída de `GOOGLE_API_KEY` y nunca
  commiteada. `temperature=0` y salida estructurada, como todos los proveedores.
- **El id del modelo es configuración, no código.** Por defecto, un modelo *Flash* vigente con plan
  gratuito. Los ids de modelo cambian cada pocos meses, así que el que se usó se registra con cada
  corrida de evaluación.
- **Condición: nunca datos personales.**
  - En el plan gratuito, Google **puede usar los prompts y las respuestas para mejorar sus
    productos**.
  - Es aceptable solo porque el contrato de logs garantiza que los datos personales se eliminan
    antes de escribir la línea (ADR-001).
  - Si el investigador alguna vez suma una fuente sin esa garantía, el plan gratuito deja de ser
    una opción para ella.
- **Límites de uso**: el plan gratuito tiene límites, y cambian.
  - El adaptador del modelo reintenta con backoff ante errores de límite.
  - El runner de evaluación espacia los escenarios.
  - Los límites se verifican el día que se corre la suite; no se anotan acá.
- **Los otros proveedores siguen soportados**:
  - Ollama: offline, modelos chicos, para la comparación de la evaluación.
  - Anthropic: extra opcional con tope de gasto.

## Consecuencias

- Los resultados de evaluación se informan por modelo. El README muestra la comparación, no un
  número único.
