
---
version: 1
actualizado: 2026-08-03
ocm_filter_field: gab_ocm_bool
ocm_filter_value: true
purchase_states: [purchase, done]
ventana_dias: 7
autorizacion_requeridas: [gab_ocm_rq_bool, gab_ocm_av_bool, gab_ocm_dg_bool]
autorizacion_una_de: [gab_ocm_wh_bool, gab_ocm_vb_bool]
prefijos_anticipo_omitir: [PBA, PBN]
match_amount_tolerance: 0.005
ocm_product_id: 1277
unidad_uom_category_id: 1

ai_model: openrouter/auto
---
# Reglas de conciliación OCM–Facturas (Grupo GAB)
Solo editar los valores de arriba (YAML). No cambiar los nombres de las llaves — n8n y Claude las leen tal cual.