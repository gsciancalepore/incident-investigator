# Roadmap

Última actualización: 2026-10-12 (requisitos en borrador, docs traducidos al español, spec
`observabilidad-02` en borrador en Revestimientos).

| Versión | Suma | Depende de |
|---|---|---|
| **v1** | CLI, investigación sobre logs `.jsonl` locales, modelo intercambiable, suite de evaluación | Contrato de logs v1 de Revestimientos (spec `observabilidad-01-contrato-logs`), ampliado por `observabilidad-02` |
| v2 | RAG sobre la documentación de Revestimientos (Chroma); búsqueda híbrida opcional sobre logs | Resultados de la evaluación de v1 |
| v3 | Logs de staging con un adaptador nuevo de `LogSource` (API de Render o un backend de logs gratuito) | Etapa 2 de observabilidad de Revestimientos (ADR-013 de Revestimientos) |
| v4 | Herramientas expuestas por MCP; una interfaz Gradio para demos | — |
| v5 | Corridas disparadas por alertas (SDK de agentes o un runner programado) | Etapa 5 de observabilidad de Revestimientos; una decisión de presupuesto, porque las llamadas a modelos hosteados se pagan |

Nada después de la v1 está comprometido. Cada versión empieza con su propia ADR.

## v1 — antes de escribir código

1. ~~Revestimientos mergea el contrato de logs v1~~: hecho (PR #39). El schema está vendoreado en
   `contracts/revestimientos/`.
2. ~~Decidir el proveedor de modelo por defecto~~: plan gratuito de Gemini (ADR-003). Crear una
   `GOOGLE_API_KEY`.
3. **Requisitos** ([`requisitos.md`](requisitos.md)): borrador escrito el 2026-10-12. Falta la
   revisión del dueño.
4. **Cambios de logs en Revestimientos**: spec `observabilidad-02-ampliacion-contrato-v1` en
   borrador v3. Pasó dos rondas de `revisor-spec` ("aprobable con correcciones menores",
   ya aplicadas) y tiene las decisiones D1–D3 del dueño. Faltan la aprobación y la implementación. Al mergear, copiar de nuevo el schema en `contracts/revestimientos/`.
5. **Revisar el stack** con los requisitos cerrados: ADR-002 y ADR-003 están en revisión, y
   `arquitectura.md` §Pendiente lista lo que hay que alinear.
6. Reproducir los escenarios de evaluación en Revestimientos y commitear sus fixtures de logs.
   Depende del paso 4, porque los fixtures tienen que incluir los eventos nuevos.

## Punto de retome — 2026-10-12

**Leer esto primero.** Hay dos ramas sin push:
- **Acá**: `docs/requisitos-y-espanol`, con los requisitos y la traducción de todos los docs al
  español. Los nombres de archivo siguen la convención de Revestimientos (`arquitectura.md`,
  `ADR-00N-slug.md`).
- **En Revestimientos**: `docs/observabilidad-02-spec`, con el borrador de la spec.

Lo próximo es el paso 4: que el dueño apruebe la spec (v3) y se implemente en Revestimientos.
