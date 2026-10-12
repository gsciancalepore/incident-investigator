# ADR-004 — Sin generación aumentada por recuperación (RAG) en v1

- **Estado**: **aceptada (2026-09-24)**.

## Contexto

La única fuente de la v1 son logs estructurados. Se buscan mejor con filtros exactos que por
similitud de embeddings. Los filtros son: `request_id`, `order_id`, `payment_id`, evento, nivel y
ventana de tiempo.

Una búsqueda semántica sobre líneas de log agregaría un vector store y un modelo de embeddings, y
recuperaría con menos precisión que un filtro.

## Decisión

- Sin vector store en v1. La recuperación sobre logs es filtrado determinístico más saltos de
  correlación (ADR-002).
- El conocimiento del dominio que necesita el modelo, por ejemplo qué estados son previstos, sale
  de las descripciones de eventos del contrato de logs, inyectadas en el prompt de análisis.

## Condición de revisión

Se revisa cuando la v2 sume la **documentación** del sistema observado como segunda fuente: specs,
ADRs y runbooks.

- Son texto no estructurado, donde RAG sí encaja.
- El diseño candidato es Chroma con un retriever sobre `docs/` de Revestimientos.
  - Lo consulta el nodo de análisis para explicar *por qué* el sistema se comportó así, citando el
    documento.
- También es candidata para v2 la recuperación híbrida sobre logs (filtros más búsqueda semántica
  sobre las descripciones de eventos), si la evaluación muestra preguntas que los filtros no
  resuelven.
