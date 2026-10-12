# ADR-002 — LangGraph con recolección determinística de evidencia

- **Estado**: **aceptada (2026-09-24)**. **En revisión desde el 2026-10-12**: el dueño reabrió el
  stack hasta cerrar los [requisitos](../requisitos.md). Lo que se revisa es la **herramienta**
  (LangGraph); el **patrón** (workflow explícito, con el modelo decidiendo y el código buscando y
  verificando) no está en discusión.

## Contexto

El patrón por defecto es un agente ReAct libre, que decide qué herramienta llamar en cada paso. En
este problema tiene dos costos:

- **Reproducibilidad**: el recorrido por los logs cambia de una corrida a otra.
- **Modelos locales chicos**: llaman herramientas de forma poco confiable, y la v1 tiene que
  funcionar con ellos.

## Decisión

La investigación es un `StateGraph` explícito de LangGraph:

1. entender;
2. reunir evidencia;
3. armar la línea de tiempo;
4. analizar;
5. verificar;
6. informar.

**El modelo decide qué buscar y qué significa**:
- El nodo 1 convierte la pregunta en una consulta tipada.
- El nodo 4 formula hipótesis que citan ids de evidencia, o pide más evidencia.

**El código busca y verifica**:
- Los nodos 2 y 3 consultan la fuente de logs y correlacionan `order_id`, `payment_id` y
  `request_id`.
- El nodo 5 verifica que cada cita exista.

Dos loops acotados dan el comportamiento agéntico:
- **Analizar → reunir**: hasta 2 rondas extra de evidencia.
- **Verificar → analizar**: 1 ronda de reflexión cuando fallan las citas.

## Consecuencias

- Todos los nodos salvo el 1 y el 4 se pueden testear sin modelo.
- Las citas inventadas se detectan por estructura, no por prompt.
- Es menos abierto que un agente ReAct. Las preguntas que necesitan una fuente distinta de los logs
  quedan fuera de alcance hasta que esa fuente exista detrás del puerto `LogSource`.
