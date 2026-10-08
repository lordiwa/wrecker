---
id: UC-07
name: Pagar pedido
actor: Comprador
level: user_goal
scope: app_movil
spec_version: uc07-v2.0.0

# fog_min por campo (D-1.4). Una rebanada de nivel f contiene los campos con
# fog_min <= f. Es una PROYECCION DE CAMPOS, no un recorte de texto, y por eso
# "ningun campo de fog 3 aparece en una rebanada de fog 1" es una asercion.
fog:
  id: 1
  name: 1
  actor: 1
  scope: 1
  trigger: 1
  preconditions: 2
  main_flow: 2
  level: 3
  alternate_flows: 3
  exception_flows: 3
  referenced_states: 3
  initial_world: 3
  injects: 4
  success_postcondition: 4
  failure_postcondition: 4
  spec_version: 4

preconditions:
  - sesión activa
  - carrito con al menos un artículo

trigger: "El comprador pulsa Pagar"

# El mundo con el que arranca la sesion. Lo lee y lo escribe el ancla.
initial_world:
  session_active: true
  saved_methods_present: true
  balance_sufficient: true
  cart: { items: 2 }
  order: { status: pending, total: 60 }
  payment: { captured: false, sent: false, charges: 0 }

# Postcondiciones EVALUABLES, no prosa (D-1.2). Cinco operadores y solo cinco:
# eq, exists, absent, count, matches.
success_postcondition:
  all:
    - { op: eq, path: order.status, value: paid }
    - { op: eq, path: payment.captured, value: true }
    - { op: eq, path: cart.items, value: 0 }
    # El pedido se cobra EXACTAMENTE UNA VEZ.
    - { op: eq, path: payment.charges, value: 1 }

failure_postcondition:
  all:
    - { op: eq, path: order.status, value: pending }
    - { op: eq, path: payment.captured, value: false }

referenced_states:
  - session_active
  - cart.items
  - saved_methods_present
  - balance_sufficient
  - gateway_fault

# Eventos del Director aplicables a este mundo. Se suman a los del catalogo
# generico (`design/config/injects.catalog.yaml`), que es de dominio neutro.
# `applies_when` usa el MISMO evaluador de predicados que las postcondiciones
# (cierre derivado 2.8).
injects:
  - name: sin_metodos_guardados
    family: datos
    applies_when: { op: eq, path: saved_methods_present, value: true }
    effect: { saved_methods_present: false }
  - name: saldo_insuficiente
    family: datos
    applies_when: { op: eq, path: balance_sufficient, value: true }
    effect: { balance_sufficient: false }
  - name: pasarela_rechaza
    family: terceros
    applies_when: { op: eq, path: payment.sent, value: true }
    effect: { gateway_fault: rechazado }
  - name: pasarela_muda
    family: terceros
    applies_when: { op: eq, path: payment.sent, value: true }
    effect: { gateway_fault: sin_respuesta }
  # Familia `datos` del catalogo generico (`malformed_upstream_payload`),
  # atada al dominio. No esta plantado: es el tercer modo de fallo de una
  # pasarela, y UC-07 solo enumera dos.
  - name: pasarela_respuesta_malformada
    family: datos
    applies_when: { op: eq, path: payment.sent, value: true }
    effect: { gateway_fault: malformada }
  - name: sesion_expirada
    family: sesion
    applies_when: { op: eq, path: session_active, value: true }
    effect: { session_active: false }
---

## Flujo principal

1. [pulsar_pagar] Comprador: pulsa Pagar → sistema muestra resumen y métodos de pago guardados.
   - efecto: { ui.screen: resumen_pago }
   - goto: 2
2. [elegir_metodo_guardado] Comprador: elige un método guardado → sistema pide confirmación.
   - requiere: { op: eq, path: saved_methods_present, value: true }
   - efecto: { payment.method: guardado, ui.screen: confirmacion }
   - goto: 3
3. [confirmar] Comprador: confirma → sistema envía cobro a la pasarela.
   - efecto: { payment.sent: true, ui.screen: esperando_pasarela }
   - goto: 4
4. [respuesta_pasarela_aprobada] Pasarela: responde aprobado → sistema marca pedido paid, vacía carrito, muestra confirmación.
   - requiere: { op: absent, path: gateway_fault }
   - efecto: { order.status: paid, payment.captured: true, payment.charges: { inc: 1 }, cart.items: 0, ui.screen: confirmacion_pago }
   - goto: fin_exito

## Flujos alternativos

- 2a. condición: no hay métodos guardados
  - ancla: 2
  - applies_when: { op: eq, path: saved_methods_present, value: false }
  - 2a1. [ver_formulario_tarjeta] Sistema: muestra formulario de tarjeta.
    - efecto: { ui.screen: formulario_tarjeta }
    - goto: 2a2
  - 2a2. [introducir_tarjeta] Comprador: introduce tarjeta válida → vuelve a 2.
    - requiere: { op: exists, path: input.card }
    - efecto: { saved_methods_present: true }
    - goto: 2
- 2b. condición: comprador elige pagar con saldo
  - ancla: 2
  - 2b1. [pagar_con_saldo] Sistema: descuenta saldo → vuelve a 4.
    - requiere: { op: eq, path: balance_sufficient, value: true }
    - efecto: { payment.captured: true, payment.charges: { inc: 1 }, payment.method: saldo }
    - goto: 4

## Flujos de excepción

- 3a. condición: comprador cancela en la confirmación
  - ancla: 3
  - recuperación: vuelve al carrito intacto
  - 3a1. [cancelar] Comprador: cancela en la confirmación.
    - efecto: { order.status: pending, payment.captured: false, ui.screen: carrito }
    - goto: fin_fracaso
- 4a. condición: pasarela responde rechazado
  - ancla: 4
  - applies_when: { op: eq, path: gateway_fault, value: rechazado }
  - recuperación: ofrece otro método
  - 4a1. [respuesta_pasarela_rechazada] Sistema: muestra motivo y ofrece otro método → vuelve a 2.
    - efecto: { ui.screen: motivo_rechazo, payment.sent: false, gateway_fault: null }
    - goto: 2
- 4b. condición: pasarela no responde en 30 s
  - ancla: 4
  - applies_when: { op: eq, path: gateway_fault, value: sin_respuesta }
  - recuperación: muestra "no pudimos confirmar" y ofrece reintentar
  - 4b1. [timeout_pasarela] Sistema: no pudimos confirmar el pago.
    - efecto: { order.status: pending, payment.captured: false, ui.screen: no_pudimos_confirmar }
    - goto: fin_fracaso

## Estados referidos

- session_active, cart.items, saved_methods_present, balance_sufficient, gateway_fault
