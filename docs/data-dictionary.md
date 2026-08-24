# Data dictionary

## Power Automate → Logic App request payload

Sent as the HTTP POST body from Power Automate to the Logic App's trigger.

| Field | Type | Required | Notes |
|---|---|---|---|
| `remitente` | string | yes | Original sender's email address |
| `asunto` | string | yes | Subject line of the exception email |
| `cuerpoCorreo` | string | yes | Body text of the exception email — currently the only source of exception details the model sees |
| `adjuntoNombre` | string | no | Filename of an attachment, if present — not yet parsed (see "what I would do differently" for Document Intelligence as a next step) |
| `adjuntoContenidoBase64` | string | no | Base64 content of an attachment, if present — not yet parsed |

## Logic App decision response

Returned by the Logic App's `Response` action, and the object Power Automate's `Condition` action branches on.

| Field | Type | Notes |
|---|---|---|
| `proveedor` | string | Vendor/sender extracted from the exception |
| `monto` | number | Invoice amount in USD |
| `numero_po` | string \| null | Purchase order number, or `null` if not found in the exception text |
| `tipo_discrepancia` | string | One of: `precio`, `cantidad`, `po_faltante`, `factura_duplicada`, `otro` |
| `porcentaje_discrepancia` | number \| null | Discrepancy percentage, or `null` when not applicable (e.g. for `po_faltante`) |
| `decision` | string | One of: `auto_aprobar`, `escalar`, `rechazar_o_escalar_alta_prioridad` |
| `confianza` | number | Model's self-reported confidence, 0–100 |
| `justificacion` | string | One-line rationale for the decision, referencing which policy rule applied |

## Decision policy

Applied inside the prompt (see [`prompts/extraction-classification-prompt.md`](../prompts/extraction-classification-prompt.md)), in order — the first matching rule wins:

| Order | Condition | Decision |
|---|---|---|
| 1 | `numero_po` is null, or `tipo_discrepancia` is `factura_duplicada`, or `monto` > $5,000, or `porcentaje_discrepancia` > 15% | `rechazar_o_escalar_alta_prioridad` |
| 2 | `monto` < $500 **and** `porcentaje_discrepancia` < 5% | `auto_aprobar` |
| 3 | Anything else | `escalar` |
| — | `confianza` < 85%, regardless of the above | forced to `escalar` |

## Planned audit log (not yet implemented)

Every decision — regardless of outcome — should be recorded for traceability. This is the intended schema for that log, whether implemented as an Azure Table Storage table or a SharePoint list; wiring this into the Logic App (post-decision, before the `Response` action) is a near-term next step. Currently, decisions are only visible via the Logic App's own run history, which isn't a durable or queryable audit trail on its own.

| Field | Type | Notes |
|---|---|---|
| `timestamp` | datetime | When the decision was made |
| `remitente` | string | Original sender of the exception email |
| `proveedor` | string | Extracted vendor |
| `monto` | number | Extracted invoice amount (USD) |
| `numero_po` | string \| null | Extracted PO number |
| `tipo_discrepancia` | string | See enum above |
| `porcentaje_discrepancia` | number \| null | Extracted discrepancy percentage |
| `decision` | string | See enum above |
| `confianza` | number | Model's self-reported confidence (0–100) |
| `justificacion` | string | Model's one-line rationale |
| `run_id` | string | Logic App run identifier, for cross-referencing with Azure run history |
