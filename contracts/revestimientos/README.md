# Contrato vendoreado: contrato de logs v1 de Revestimientos

Copia del contrato de logs del que es dueño Revestimientos (el sistema observado). **No se edita
acá.** Se actualiza copiando una versión nueva desde el origen y anotando el commit.

- **Origen**: Revestimientos, `docs/observabilidad/log-schema.v1.json`. La guía legible es
  `docs/observabilidad/README.md` en el mismo repositorio.
- **Commit**: `b8c2ba0` (merge del PR #39, 2026-09-24). El archivo del schema se modificó por última
  vez en `aaabfae`.
- **Versión**: `schema_version: "1"`.

**Compatibilidad** (la fija el contrato):
- Los eventos nuevos y los atributos opcionales nuevos son compatibles y se tienen que tolerar.
- Quitar, renombrar o cambiar el tipo de algo exige `schema_version: "2"`.

**Pendiente**: cuando se mergee la spec `observabilidad-02` en Revestimientos, se copia de nuevo el
schema y se actualiza el commit de origen.
