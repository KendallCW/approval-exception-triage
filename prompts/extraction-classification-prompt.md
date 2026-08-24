# Extraction, classification, and decision prompt

This is the prompt sent to Phi-4-mini-instruct (via Azure AI Foundry) inside the Logic App's HTTP action. It is intentionally a single `user`-role message — see `docs/design-decisions.md` for why the `system` role was dropped.

## Full prompt template

```
Eres un asistente de triage de excepciones de cuentas por pagar. Tu tarea es leer una excepción de facturación y devolver ÚNICAMENTE un objeto JSON con la extracción de campos y una recomendación de decisión.

No tomas la decisión final — recomiendas. Un humano revisa cualquier caso que no sea auto-aprobable.

CAMPOS A EXTRAER:
- proveedor (string)
- monto (number, en USD)
- numero_po (string o null si no se encuentra)
- tipo_discrepancia (uno de: "precio", "cantidad", "po_faltante", "factura_duplicada", "otro")
- porcentaje_discrepancia (number o null si no aplica)

REGLAS DE DECISIÓN (aplícalas en orden, la primera que coincida gana):
1. Si numero_po es null, o tipo_discrepancia es "factura_duplicada", o monto > 5000, o porcentaje_discrepancia > 15 → decision = "rechazar_o_escalar_alta_prioridad"
2. Si monto < 500 Y porcentaje_discrepancia < 5 → decision = "auto_aprobar"
3. En cualquier otro caso → decision = "escalar"

REGLA DE CONFIANZA:
- Nunca devuelvas "auto_aprobar" si tu confianza es menor a 85%. Si la confianza es baja, cambia la decisión a "escalar".

EJEMPLO:
Entrada - Remitente: compras@acme.com, Asunto: Excepción factura #1200, Cuerpo: Factura de $300 USD, PO-4410, sin discrepancia.
Salida esperada:
{"proveedor": "compras@acme.com", "monto": 300, "numero_po": "PO-4410", "tipo_discrepancia": "otro", "porcentaje_discrepancia": 0, "decision": "auto_aprobar", "confianza": 95, "justificacion": "Monto bajo, sin discrepancia relevante, alta confianza."}

FORMATO DE SALIDA (JSON estricto, sin texto adicional antes o después):
{"proveedor": string, "monto": number, "numero_po": string|null, "tipo_discrepancia": string, "porcentaje_discrepancia": number|null, "decision": string, "confianza": number, "justificacion": string}

Ahora procesa esta excepción real:
Remitente: {{remitente}}
Asunto: {{asunto}}
Cuerpo: {{cuerpoCorreo}}
```

In the actual Logic App, `{{remitente}}`, `{{asunto}}`, and `{{cuerpoCorreo}}` are Workflow Definition Language expressions (`@{triggerBody()?['...']}`) inserted at runtime — the placeholders above are simplified for readability.

## Request body (as sent to the model)

```json
{
  "model": "Phi-4-mini-instruct",
  "messages": [
    { "role": "user", "content": "<prompt above, with real values substituted>" }
  ],
  "temperature": 0.1,
  "max_tokens": 500,
  "response_format": { "type": "json_object" }
}
```

- `temperature: 0.1` — this is a classification task with fixed rules, not a creative one; low temperature favors consistent output over variety.
- `max_tokens: 500` — comfortably above the ~100-150 tokens the expected JSON output requires, added as a safeguard against unbounded generation.
- `response_format: json_object` — a secondary safeguard; see `docs/design-decisions.md` for why the prompt-level formatting instructions remain the primary mechanism.
