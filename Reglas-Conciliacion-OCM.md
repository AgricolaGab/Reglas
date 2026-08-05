---

version: 3 actualizado: 2026-08-05

# === Filtro: qué OCMs procesar ===

ocm_filter_field: gab_ocm_bool ocm_filter_value: true purchase_states: [purchase, done] ventana_dias: 30 # ventana del flujo n8n (reporte). La tarea de conciliación directa usa 7.

# === Autorización requerida para poder ligar ===

autorizacion_requeridas: [gab_ocm_rq_bool, gab_ocm_av_bool, gab_ocm_dg_bool] autorizacion_una_de: [gab_ocm_wh_bool, gab_ocm_vb_bool]

# === Conciliación / criterios de match ===

criterios_activos: [C2, C3, C4] match_amount_tolerance: 0.005 # C4 exige monto igual; 0.005 = tolerancia de centavos por flotantes prefijos_anticipo_omitir: [PBA, PBN]

# === IDs de Odoo para el vínculo visual (botón Compras) ===

## ocm_product_id: 1277 unidad_uom_category_id: 1 # categoría "Unidad" (Units) = OCM normal servicio_uom_category_id: 27 # categoría "Service Unit" = OCM de servicio (sin purchase_line_id)

# Reglas de conciliación OCM–Facturas (Grupo GAB)

Esta nota es la **fuente de la verdad**: n8n la lee (desde GitHub) y hace lo que dice. Solo edita los **valores** del bloque YAML de arriba. **No cambies los nombres de las llaves.**

## Config (llaves del YAML)

- **ocm_filter_field / ocm_filter_value** — procesa EXCLUSIVAMENTE OCMs (`gab_ocm_bool = true`). Nunca OC regulares. Filtro obligatorio, nunca se quita.
- **purchase_states** — estados de la orden que se consideran: `purchase`, `done`.
- **ventana_dias** — cuántos días hacia atrás busca, por `date_approve` (OCMs) y por `invoice_date`/`create_date` (facturas).
- **autorizacion_requeridas** — TODAS deben estar en `true` (RQ, AV, DG).
- **autorizacion_una_de** — además, al menos UNA en `true` (WH o VB). Es decir: siempre 4 de 5, nunca las 5.
- **criterios_activos** — qué criterios de match se aplican (C2, C3, C4).
- **match_amount_tolerance** — tolerancia de monto para C4 (monto prácticamente exacto).
- **prefijos_anticipo_omitir** — OCM cuyo nombre empieza con estos prefijos se omiten (anticipos): PBA, PBN.
- **ocm_product_id** — producto "OCM" que se pone en la línea de la factura (1277).
- **unidad_uom_category_id / servicio_uom_category_id** — categoría de UdM: 1 = normal (con botón Compras), 27 = servicio (se liga pero queda sin vínculo visual).

## Procedimiento (lo que debe hacer el flujo)

1. **Ventana** — OCMs y facturas dentro de `ventana_dias`.
2. **Filtro OCM** — solo `gab_ocm_bool = true`, estados en `purchase_states`.
3. **Ligación preexistente (C1)** — si la OCM ya tiene `invoice_ids`, se deja tal cual: no se re-liga, no se reporta, no cuenta como "Ligada", aunque le falten firmas o su factura esté en draft.
4. **Autorización** — según `autorizacion_requeridas` + `autorizacion_una_de`. Si falta alguna requerida, o ninguna de "una_de" está en true → **NO_AUTORIZADA** (se anota qué falta, no se toca).
5. **Anticipos** — nombre que empieza con un prefijo de `prefijos_anticipo_omitir` → **ANTICIPO_OMITIDO**.
6. **Facturas** — `in_invoice`, no `cancel`. Si `state == draft` → **DRAFT_PENDIENTE**, nunca se toca.
7. **Criterios (orden estricto, una factura = una OCM)**:
    - **C2**: `invoice_origin` de la factura = `name` de la OCM.
    - **C3**: `ref` de la factura contiene el `name` de la OCM.
    - **C4**: mismo proveedor + monto igual (dentro de tolerancia) + una sola factura candidata. Si empatan 2+ o la factura ya está ligada en la base → **SIN_COINCIDENCIA** silencioso (no revisión manual).
8. **Verificaciones antes de ligar (todas C2/C3/C4)**:
    - mismo `partner_id` OCM = factura (si no → REVISION_MANUAL "proveedor no coincide").
    - `payment_state == not_paid` (una factura con pago aplicado NUNCA se liga, ni lógica ni visualmente).
    - **C4 además**: la factura no está ligada a NINGUNA OCM en toda la base; y la OCM no tiene ya una factura `posted` vinculada.
9. **Ligar** — `invoice_ids += factura`. Éxito → LIGADA_C2/C3/C4.
10. **Vínculo visual (botón Compras)** — para toda ligada nueva, reabre la factura y ajusta la línea producto. Aborta a **LIGADA_SIN_VISUAL** (sin forzar) si: la factura tiene pago aplicado, cae en periodo cerrado (`fiscalyear_lock_date`/`tax_lock_date`), o tiene más de una línea producto. Verifica el monto antes/después; si no cuadra, **ERROR_VISUAL** (no postea).
11. **Etiqueta de la línea** — SIEMPRE `"<número de OCM>: <descripción de la línea de la OCM>"` (el `name` de la `purchase.order.line`). **NUNCA** `"<número>: OCM"`. Aplica a OCMs normales y de servicio.
12. **OCM de servicio** (UdM categoría `servicio_uom_category_id` = 27) — se liga y se pone producto+etiqueta CON descripción, pero **sin** `purchase_line_id`, y queda **LIGADA_SIN_VISUAL**. Nunca forzar `purchase_line_id`.
13. Una factura = una OCM por corrida. Sin tope de ligaciones.

## Notas

- Las OCMs/facturas sin calificar reaparecen cada corrida mientras estén en la ventana (esperado). Solo se evita re-ligar lo ya ligado (punto 3).
- El vínculo visual es delicado: cambiar el producto de una línea posteada borra impuestos hasta restaurarlos; por eso se verifica monto antes/después y se aborta a ERROR_VISUAL si no cuadra.