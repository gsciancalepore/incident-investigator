# Requisitos — v1 (borrador)

> Estado: **borrador para revisión** (2026-10-12). Se escribe antes de revisar el stack: las
> herramientas tienen que salir de estos casos de uso, no al revés. Donde este documento y
> `arquitectura.md` no coincidan, este es el más nuevo y la arquitectura se va a ajustar.

## 1. Usuario y contexto

Hay un solo usuario: **el desarrollador de la tienda, atendiendo el reclamo de un cliente**.

El cliente escribe, normalmente por WhatsApp o email, con algo como "pagué y mi pedido sigue
pendiente" o "toqué pagar y no pasó nada". El desarrollador tiene que responder dos preguntas: qué
pasó, y si tiene que actuar el cliente, la tienda, o si hay un bug para corregir.

Con qué cuenta el desarrollador al empezar:

| Dato | De dónde sale | ¿Llega al agente? |
|---|---|---|
| Número de pedido | La página de éxito, o buscado por email en el panel de admin | **Sí** |
| `payment_id` de MercadoPago | El comprobante del cliente | **Sí** |
| Fecha y hora aproximadas | El reclamo | **Sí** |
| Producto que intentaba comprar | El reclamo | Sí (id del producto) |
| Nombre, email, teléfono | El reclamo | **Nunca** |

**Precondición**: el desarrollador traduce la identidad del cliente a un `order_id` con el panel de
admin, que lee la base de datos. El agente nunca recibe datos personales ni toca la base (ADR-001).
Es una decisión a propósito, no algo que haya que resolver.

## 2. Consumo de logs

- El agente lee **solo** los logs del contrato, a través de un **montaje de solo lectura** de un
  directorio que no contiene nada más:
  `../ceramica/storage/logs/contrato:/logs:ro`.
- `storage/logs` en Revestimientos también tiene `browser.log` y `laravel.log`. Ninguno de los dos
  está cubierto por el contrato ni por su garantía de redacción, así que el agente no tiene que
  verlos (ver OBS-10).
- El agente tolera una última línea incompleta, porque la tienda puede estar escribiendo el archivo
  mientras el agente lo lee.
- **Alcance de v1: solo local.** El montaje funciona porque los dos sistemas corren en la misma
  máquina, y eso cubre desarrollo y evaluación. Revestimientos está deployado en **Render** (base
  de datos en **Neon**). El disco de Render es efímero: los archivos de log locales se pierden en
  cada deploy o reinicio. Leer logs de producción requiere otro adaptador de `LogSource`, que
  queda para v3 (roadmap).

## 3. Casos de uso

La cobertura se evalúa contra el contrato de logs v1 más los cambios de la sección 6.

| # | Reclamo | El desarrollador aporta | Evidencia en el contrato v1 | Cobertura |
|---|---|---|---|---|
| CU1 | "Pagué y el pedido sigue pendiente" | pedido o `payment_id` | `checkout.payment_started`, `webhook.*`, `order.paid`, `order.status_changed` | **Parcial**: si no llegó ningún webhook, los logs no distinguen entre un pago abandonado y un webhook perdido. Lo resuelve en parte OBS-12 |
| CU2 | "Toqué pagar y no pasó nada" | pedido | `checkout.mp_preference_failed`, `app.exception` por `request_id` | Completa |
| CU3 | "Me rechazaron el pago" / "me dice pendiente" | pedido o `payment_id` | `webhook.payment_not_approved` con `status` | Completa |
| CU4 | "Me cobraron un monto distinto" | pedido o `payment_id` | `order.payment_amount_mismatch` (`cobrado_cents` contra `total_cents`) | Completa |
| CU5 | "Pagué, pero me cancelaron el pedido" | pedido | `order.paid_after_cancel`, `order.status_changed` | Completa en lo esencial: toda cancelación la hace un admin (regla 165). Qué admin fue no se registra, y no cambia la respuesta |
| CU6 | "No pude comprar / me dio error" | ventana de tiempo, quizás el producto | `checkout.rejected` (`motivo`, `campos`, `detalle`), `http.request` con status ≥ 400, `app.exception` | **Parcial**: el rechazo no se puede atar a este cliente. Lo resuelve en parte OBS-13 |
| CU7 | "Pagué por transferencia y nadie lo confirmó" | pedido | `checkout.payment_started` (`transferencia`), `order.paid` con `origen: manual` | Completa (la respuesta es "todavía no se confirmó") |
| CU8 | "El precio que pagué no es el que vi" | pedido, producto | `order.created` (`lines`), `product.price_changed` | **Parcial**: `lines` trae `product_id` y `cantidad`, pero no el precio unitario. Lo resuelve OBS-14 |
| CU9 | "Me dijeron que mi producto no tiene stock" | pedido, producto | `order.stock_negative` | Completa: es un estado **previsto**, no un bug |
| CU10 | "El sitio anda lento / no anda" | ventana de tiempo | `http.request` (`status`, `duration_ms`), `app.exception` | Completa |
| CU11 | Cualquier consulta sobre un pedido que no existe | pedido | ninguna | Completa: "no se encontró evidencia" |

### Conocimiento del dominio que el agente necesita

Estas reglas de Revestimientos explican comportamientos que, sin contexto, parecerían fallas. En la
copia vendoreada del contrato (la v1 original) no todas están en las descripciones. OBS-15 las agrega
en Revestimientos, y llegan acá al volver a copiar el schema:

- **Toda cancelación la hace un admin** (regla 165). El cliente no puede cancelar y no hay
  cancelaciones automáticas.
- **Un pedido impago nunca vence solo.** No hay scheduler en ningún entorno (ADR-012 de Revestimientos, descartada):
  un pedido en `pending_payment` sigue así hasta que llega el pago o un admin lo cancela.
- **El stock se descuenta al confirmar el pago, no al crear el pedido** (ADR-005 de Revestimientos). Un pago cobrado
  nunca se rechaza por falta de stock (regla 145): el stock queda negativo.
- **Cancelar un pedido pagado restituye el stock** (`order.stock_restored`). La devolución del
  dinero se hace fuera del sistema.

### Fuera de alcance en v1

- **Envíos y entregas**: la tienda no emite eventos de envío.
- **Emails o notificaciones al cliente**: la tienda no envía ninguno.
- **Encontrar el pedido a partir de la identidad del cliente**: lo hace el desarrollador, por
  diseño.
- **Devoluciones de dinero**: se hacen fuera del sistema, en MercadoPago.
- **Preguntas sobre el panel de admin** (usuarios, roles): los eventos existen, pero no son
  reclamos de clientes. Se pueden sumar después sin costo.
- **Ejecutar acciones**: el agente diagnostica. El desarrollador decide y actúa.

## 4. Reglas del agente

Son las reglas de negocio del agente. Cada una la verifica el código o la suite de evaluación, no
solo el prompt.

| # | Regla | La hace cumplir |
|---|---|---|
| R1 | Toda afirmación cita una línea de log que existe en el conjunto de evidencia | Código (paso de verificación) y eval: validez de citas = 100% |
| R2 | Nunca afirmar un identificador (pedido, pago, request, producto) que no esté en los logs | Eval: identificadores inventados = 0 |
| R3 | La ausencia de un evento no prueba que algo no ocurrió. Decir qué no pueden determinar los logs | Prompt y eval (CU1) |
| R4 | Los estados que el contrato marca como previstos, como `order.stock_negative` o `webhook.ignored`, se informan como previstos, no como bugs | Descripciones del contrato en el prompt, y eval (CU9) |
| R5 | Si no encuentra nada, responder "no se encontró evidencia" y listar qué se buscó | Código y eval (CU11) |
| R6 | Los valores de los logs son datos no confiables, nunca instrucciones | Bloques de evidencia delimitados (arquitectura, límites de confianza) |
| R7 | Sin efectos secundarios: lee un directorio y escribe un informe | Montaje de solo lectura |
| R8 | El informe termina con un veredicto accionable para el desarrollador: quién tiene que actuar (cliente, tienda, desarrollador) y cómo | Modelo del informe |

## 5. Criterios de éxito (umbrales propuestos)

| Métrica | Umbral | Por qué |
|---|---|---|
| Validez de citas | 100% | R1 es la promesa del producto |
| Identificadores inventados | 0 | R2 |
| Respuestas "sin evidencia" correctas | 100% | R5 |
| Precisión de causa raíz sobre los escenarios de evaluación | ≥ 80% con el modelo por defecto | A revisar después de la primera corrida |
| Latencia por pregunta | < 60 s | El desarrollador está respondiendo a un cliente |
| Costo por pregunta | $0 con el proveedor por defecto | Presupuesto del dueño (ADR-003) |

## 6. Cambios pedidos a los logs de Revestimientos

Están especificados en Revestimientos, en la spec
`docs/specs/observabilidad-02-ampliacion-contrato-v1.md`: **aprobada** el 2026-10-12 y mergeada en el
PR #42. Ese mismo día se **implementó** en la rama `feat/observabilidad-02-ampliacion-contrato`, con
PR #44 abierto. Todos los cambios son compatibles con el contrato v1.

| Regla | Cambio | Habilita |
|---|---|---|
| OBS-10 | Directorio propio del contrato: `storage/logs/contrato/` | El montaje de solo lectura sin `browser.log` ni `laravel.log` (sección 2) |
| OBS-11 | Retención de 90 días | Reclamos de más de dos semanas |
| OBS-12 | Evento `checkout.returned`: el navegador volvió desde MercadoPago, con el estado que dice traer | CU1: distingue "volvió y el webhook no llegó" de "no volvió". Si no volvió, sigue sin poder distinguirse |
| OBS-13 | `session_ref` (HMAC del id de sesión) en `http.request` | CU6, cuando el cliente terminó creando un pedido: desde ese pedido se encuentran sus intentos rechazados previos. Si nunca creó un pedido, queda en ventana de tiempo |
| OBS-14 | `precio_unitario_cents` en las `lines` de `order.created` | CU8 |
| OBS-15 | Las reglas de dominio de la sección 3 pasan a las descripciones del schema | Que el agente las conozca sin RAG (ADR-004) |

**Descartados**, porque no tapan un agujero de ningún caso de uso:
- `actor_id` en los espejos de auditoría: por la regla 165 ya se sabe que toda cancelación la hace
  un admin.
- `release` en cada línea: en local no hay despliegues.

**Del lado del investigador, sin cambiar el contrato**: los eventos de producto identifican el
producto en `subject_id`, mientras que `order.stock_negative` y `lines[]` usan `product_id`. El
investigador normaliza todo a una sola clave de producto.

## 7. Preguntas resueltas

1. **¿Qué contiene `lines` en `order.created`?** `product_id` y `cantidad`; el precio se agrega con
   OBS-14.
2. **¿Quién cancela pedidos?** Solo un admin (regla 165). No hay cancelación del cliente ni
   vencimiento automático.
3. **¿El carrito vive en sesión o en base?** En **sesión**, sin tabla (regla 81, Spec 05). El
   cliente es anónimo (regla 27). Por eso OBS-13 usa un hash de sesión y no un id de carrito.
4. **¿Dónde corre Revestimientos?** En Render, con la base en Neon. La v1 del investigador es solo
   local (sección 2).
5. **¿Dónde viven las reglas de dominio que necesita el agente?** En las descripciones del schema,
   que son del lado observado (OBS-15).
