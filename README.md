# incident-investigator

Un agente de IA que investiga incidentes de producción leyendo los logs de un sistema observado.
Responde preguntas como *"¿qué pasó con el pedido 123?"* con un informe donde cada afirmación cita
la línea de log que la respalda. Si no encuentra nada, dice "no se encontró evidencia" en lugar de
adivinar.

El primer sistema observado es **Revestimientos**, una tienda online en Laravel 12. Los dos sistemas
son independientes: comparten un contrato de logs versionado (JSON Schema + catálogo de eventos), y
nada más.

> **Estado** (2026-10-12): arquitectura definida, contrato de logs v1 vendoreado y
> [requisitos](docs/requisitos.md) en borrador. Implementación no empezada. El stack está **en
> revisión** hasta cerrar los requisitos. Revestimientos ya implementó los cambios de logs que piden
> los requisitos (spec `observabilidad-02`), y falta mergear su PR. Lo próximo está en
> [`docs/roadmap.md`](docs/roadmap.md).

## El diseño en pocas líneas

- **Externo por diseño**:
  - Lee un directorio de archivos `.jsonl`, montado en solo lectura.
  - Sin base de datos y sin red hacia el sistema observado.
  - Los datos personales se eliminan antes de escribir los logs.
- **Grafo explícito, no un agente libre**: un pipeline de LangGraph donde el modelo decide *qué
  buscar* y *qué significa*, y el código hace la búsqueda, la correlación y la verificación de
  citas.
- **Modelo intercambiable**: plan gratuito de Gemini por defecto, y Ollama o Anthropic como
  alternativas, detrás de la interfaz de chat models de LangChain, siempre con salida estructurada.
- **Medido**: una suite de evaluación con incidentes reproducidos a propósito y respuestas conocidas,
  puntuada por modelo: precisión de causa raíz, validez de citas e identificadores inventados.

## Documentos

- [Requisitos](docs/requisitos.md): usuario, casos de uso, reglas del agente y criterios de éxito.
- [Arquitectura](docs/arquitectura.md).
- Decisiones (ADRs):
  - [ADR-001 — Sistema externo y contrato de logs](docs/adr/ADR-001-sistema-externo-contrato-logs.md)
  - [ADR-002 — LangGraph con recolección determinística de evidencia](docs/adr/ADR-002-langgraph-recoleccion-deterministica.md)
  - [ADR-003 — Modelo intercambiable](docs/adr/ADR-003-modelo-intercambiable.md)
  - [ADR-004 — Sin RAG en v1](docs/adr/ADR-004-sin-rag-en-v1.md)
- [Roadmap](docs/roadmap.md).
- [Reglas del proyecto](AGENTS.md): idioma, commits, proceso y frontera de datos.
- [Contrato de Revestimientos vendoreado](contracts/revestimientos/README.md).
