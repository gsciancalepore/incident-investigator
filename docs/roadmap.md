# Roadmap

Última actualización: 2026-10-12, noche. Las fechas de este documento son en UTC: en Argentina fue
la noche del 2026-10-11. El PR #1 mergeó los requisitos, todavía como borrador, y la documentación en
español. Del lado de Revestimientos, la spec `observabilidad-02` está aprobada (#42) e implementada, y
su PR #44 espera el merge del dueño.

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
3. **Requisitos** ([`requisitos.md`](requisitos.md)): escritos el 2026-10-12 y mergeados **como
   borrador** en el PR #1. Falta que el dueño los apruebe de forma explícita; mergear el PR no
   equivale a aprobarlos.
4. **Cambios de logs en Revestimientos**, spec `observabilidad-02-ampliacion-contrato-v1`:
   - ~~aprobación~~: el dueño la aprobó el 2026-10-12, tras dos rondas de `revisor-spec`;
   - ~~merge de la spec~~: PR #42 de Revestimientos;
   - **implementación**: hecha el 2026-10-12 en la rama `feat/observabilidad-02-ampliacion-contrato`
     de Revestimientos (`github.com/gsciancalepore/revestimientos`). `revisor-entrega` la dio
     apta y el **PR #44** espera el merge del dueño. El estado exacto está en el punto de retome de
     `~/ceramica/docs/roadmap.md`.
   - **Después del merge**: volver a copiar el schema en `contracts/revestimientos/` y actualizar el
     commit de origen en su README.
5. **Revisar el stack** con los requisitos: ADR-002 y ADR-003 están en revisión, y
   `arquitectura.md` §Pendiente lista lo que hay que alinear. Ver la agenda en el punto de retome.
6. Reproducir los escenarios de evaluación en Revestimientos y commitear sus fixtures de logs.
   Depende del paso 4, porque los fixtures tienen que incluir los eventos nuevos. Además, la
   evaluación tiene que cubrir los 11 casos de uso (`arquitectura.md` §Pendiente, punto 5).

## Punto de retome — 2026-10-12 (noche)

**Leer esto primero.**

**Repositorio**: `github.com/gsciancalepore/incident-investigator`, público.
- `main` tiene el **PR #1**, mergeado por el dueño el 2026-10-12:
  - requisitos, todavía como borrador;
  - documentación traducida al español, con nombres de archivo según la convención de
    Revestimientos;
  - `AGENTS.md` y `CLAUDE.md`.
- Este punto de retome entró después, por el **PR #2** (rama `docs/retome-2026-10-12`). Si al
  retomar ese PR no está mergeado, se lee desde la rama.
- Todavía no hay código.

**Revestimientos** (`~/ceramica`) implementó los cambios de logs que piden los requisitos (paso 4).
Su punto de retome en `docs/roadmap.md` dice qué falta de ese lado.

**Lo próximo acá, en orden**:
1. `git fetch` y partir de `origin/main`. Si el PR #2 sigue abierto, el dueño lo mergea primero.
2. Que el dueño apruebe los requisitos (paso 3), o pida cambios.
3. Cuando se mergee la implementación en Revestimientos, volver a copiar el schema (paso 4).
4. **Revisar el stack** (paso 5), con [`requisitos.md`](requisitos.md) y `arquitectura.md`
   §Pendiente como entrada. Agenda propuesta, una decisión por punto, cada una con su ADR o con el
   estado actualizado de la ADR que revisa:
   1. **Orquestación**: el flujo es un workflow fijo con dos loops acotados (ADR-002). Hay que
      decidir si LangGraph se justifica o alcanza con Python plano.
   2. **Modelo**: confirmar Gemini en el plan gratuito y su modelo *Flash* vigente, y unificar la
      sección *Decisión* de ADR-003, que no lo nombra.
   3. **`LogQuery`**: filtros por producto (`product_id`, normalizado desde `subject_id`), por
      `session_ref` y por `external_reference`.
   4. **Consumo y empaquetado**: CLI en un contenedor con el montaje
      `../ceramica/storage/logs/contrato:/logs:ro`, sin red salvo hacia la API del modelo.
   5. **Informe**: el veredicto accionable de la regla R8.
   6. **Calidad**: herramientas y comandos. Al decidirlas, completar §Entorno y comandos de
      `AGENTS.md` y el Definition of Done del código.

**Reglas que no se negocian** (`AGENTS.md`, §Idioma y proceso):
- documentación y commits en español;
- sin trailers de atribución a IA;
- requisitos antes que código;
- nunca un push sin los gates en verde, cuando existan.
