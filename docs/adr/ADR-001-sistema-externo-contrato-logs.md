# ADR-001 — Sistema externo que consume un contrato de logs versionado

- **Estado**: **aceptada (2026-09-24)**.
- **Contraparte**: ADR-013 de Revestimientos (`docs/adr/ADR-013-capa-observabilidad.md` en ese
  repositorio), que fija el mismo borde desde el lado del sistema observado.

## Contexto

El dueño quiere un investigador de incidentes con IA para Revestimientos, una tienda en Laravel,
como proyecto de portfolio independiente. Se evaluó un agente dentro del repositorio de la tienda y
se descartó por dos motivos:

- **Acoplamiento**: ataba el ciclo de vida de los dos sistemas.
- **Frontera de datos**: como mostró la revisión de ese borrador, dependía de un prompt, porque un
  agente con acceso a la shell podía consultar la base directamente.

## Decisión

- El investigador es un **proyecto separado**.
- Lee **solo** un directorio de archivos `.jsonl`.
- La forma y el significado de esos archivos los define un **contrato de logs** del que es dueño el
  sistema observado: un JSON Schema más un catálogo de eventos, versionado como `v1`, `v2`, etc.
- Hay una copia del contrato en `contracts/revestimientos/`, con el commit de origen.
- El investigador valida cada línea que lee, y se niega a correr con un `schema_version` que no
  soporta.

## Consecuencias

- El investigador se puede reescribir o reemplazar sin tocar la tienda, y al revés.
- Los datos personales se eliminan del lado del productor antes de escribir la línea, así que el
  modelo nunca los recibe, haga lo que haga el agente.
- Los cambios de contrato requieren coordinación: un cambio incompatible exige una versión nueva de
  los dos lados.
