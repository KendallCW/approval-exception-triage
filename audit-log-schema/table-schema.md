# Audit log schema

Every decision the system makes — regardless of the outcome — should be recorded for traceability. This is the schema for that log, whether implemented as an Azure Table Storage table or a SharePoint list.

| Field | Type | Notes |
|---|---|---|
| `timestamp` | datetime | When the decision was made |
| `remitente` | string | Original sender of the exception email |
| `proveedor` | string | Extracted vendor/proveedor |
| `monto` | number | Extracted invoice amount (USD) |
| `numero_po` | string \| null | Extracted PO number |
| `tipo_discrepancia` | string | precio / cantidad / po_faltante / factura_duplicada / otro |
| `porcentaje_discrepancia` | number \| null | Extracted discrepancy percentage |
| `decision` | string | auto_aprobar / escalar / rechazar_o_escalar_alta_prioridad |
| `confianza` | number | Model's self-reported confidence (0-100) |
| `justificacion` | string | Model's one-line rationale |
| `run_id` | string | Logic App run identifier, for cross-referencing with Azure run history |

## Status

Schema defined; wiring this into an actual Azure Table Storage action in the Logic App (post-decision, before the Response action) is a near-term next step — currently, decisions are only visible via the Logic App's own run history, which isn't a durable or queryable audit trail on its own.
