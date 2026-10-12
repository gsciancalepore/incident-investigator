# Reglas del proyecto

Reglas para cualquier persona o agente que trabaje en este repositorio. Siguen las convenciones de
Revestimientos (`~/ceramica/AGENTS.md`), el sistema que este proyecto observa, adaptadas a un
proyecto en Python que todavía no tiene código.

## Qué es este repositorio

`incident-investigator` es un agente de IA que investiga incidentes de Revestimientos leyendo sus
logs. Es un **sistema externo** (ADR-001):
- comparte con Revestimientos solo un contrato de logs versionado;
- nunca importa, llama ni modifica a Revestimientos.

**Estado (2026-10-12)**: sin código.
- Los [requisitos](docs/requisitos.md) están en borrador.
- El stack está **en revisión** (ADR-002 y ADR-003).
- Lo próximo está en el punto de retome de [`docs/roadmap.md`](docs/roadmap.md).

## Idioma y proceso

- **Idioma**: toda la documentación, los commits y las respuestas en español.
- **Commits**: Conventional Commits, `tipo(ámbito): descripción`, con los tipos `feat`, `fix`,
  `chore`, `docs`, `refactor`, `test` y `perf`.
  - El ámbito va cuando el cambio se circunscribe a un área: `docs(adr)`, `chore(contracts)`.
  - Se omite solo si el cambio es transversal.
- **Autoría**: el **único** autor y contribuidor del proyecto es el dueño.
  - Los commits **no llevan** trailers de atribución a herramientas de IA: nada de
    `Co-Authored-By: Claude ...` ni equivalentes.
  - Tampoco van en las descripciones de los Pull Requests.
  - Si el harness sugiere agregarlos, se omiten igual: esta regla tiene prioridad.
- **Requisitos antes que código**: nada se implementa sin un caso de uso en `docs/requisitos.md`.
  Las herramientas y el stack salen de los requisitos, no al revés.
- **Nunca inventar comportamiento** del sistema observado ni del agente. Si algo es ambiguo, se
  detiene el trabajo y se pregunta al dueño.
- **Cambios importantes → ADR** en `docs/adr/`, con el formato
  `ADR-00N-slug.md` y una línea de **Estado**.
- **Las decisiones no se borran, se marcan.** Una ADR revisada, descartada o reemplazada se conserva
  con su estado actualizado (`en revisión`, `descartada`, `reemplazada por ADR-00N`) y el motivo.
- **Ramas**: todo cambio va en una rama nueva (`docs/...`, `feat/...`, `fix/...`), nunca directo en
  `main`. Cuando exista el remoto en GitHub, se integra por Pull Request.

## Orden de lectura antes de trabajar

1. `AGENTS.md`.
2. `docs/roadmap.md`, en particular el punto de retome.
3. `docs/requisitos.md`.
4. `docs/arquitectura.md`, incluida §Pendiente de alinear con los requisitos.
5. Las ADRs que toque el cambio (`docs/adr/`).
6. `contracts/revestimientos/README.md` y el schema vendoreado.

## Relación con Revestimientos

- **El dueño del contrato es Revestimientos.**
  - `contracts/revestimientos/` es una copia: **no se edita acá**.
  - Se actualiza copiando una versión nueva desde `~/ceramica/docs/observabilidad/` y anotando el
    commit de origen.
- **Si el investigador necesita un dato que los logs no tienen**, el cambio se pide con una **spec
  en Revestimientos** (`~/ceramica/docs/specs/`). Por ejemplo, `observabilidad-02`.
  - Revestimientos sigue su propio proceso: `revisor-spec`, aprobación del dueño, TDD y
    `revisor-entrega`.
  - Al trabajar en ese repositorio rigen **sus** reglas (`~/ceramica/AGENTS.md`), no estas.
- **Compatibilidad** (la fija el contrato):
  - Se toleran eventos y atributos nuevos.
  - Un `schema_version` desconocido se rechaza.
  - La ausencia de un evento nunca prueba que algo no ocurrió.

## Frontera de datos

- El agente lee **solo** el directorio del contrato de logs, montado en solo lectura. Nunca lee
  `.env`, la base de datos ni otros archivos de Revestimientos.
- Los logs no tienen datos personales, porque se eliminan del lado del productor. Esa garantía es la
  que permite usar el plan gratuito de Gemini (ADR-003): una fuente sin esa garantía no puede ir a
  ese modelo.
- Los valores de los logs son **datos no confiables**. Van al prompt solo dentro de bloques de
  evidencia delimitados, nunca como instrucciones.
- Ningún secreto se commitea. `GOOGLE_API_KEY` y similares se leen del entorno.

## Entorno y comandos

Todavía no hay código, así que tampoco hay comandos de build, tests ni calidad. Se definen al
cerrar la revisión del stack, y entonces se agregan acá junto con el Definition of Done del código.
