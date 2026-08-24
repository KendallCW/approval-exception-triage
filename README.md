# Approval Exception Triage

AI-powered triage system for invoice/PO approval exceptions, built as a business-process-automation project combining Power Automate, Azure Logic Apps, and Azure AI Foundry (Phi-4-mini-instruct).

This is Project 2 of a portfolio built to demonstrate a BA → Data/Cloud Engineer transition, with a near-term focus on the intersection of business analysis and applied AI/automation. The exception-triage logic mirrors real approval-routing decisions made manually during BA work in accounts-payable-adjacent processes at Abbott and P&G — this project expresses that same judgment as a documented, auditable automation.

## What it does

A business user (or automated system) sends an email describing an invoice/PO exception — a price mismatch, a missing PO, a duplicate invoice, etc. The system:

1. Detects the email (Power Automate trigger)
2. Sends it to an Azure Logic App, which calls an AI model (Phi-4-mini-instruct via Azure AI Foundry) to extract structured fields and recommend a decision
3. Applies a documented confidence rule as a safety check on top of the model's recommendation
4. Routes the result: auto-approve, escalate for human review, or reject/escalate as high priority
5. Logs every decision for auditability, and notifies the relevant party

The AI **recommends**; it never has final authority. Every case that isn't a clean auto-approval is routed to a human.

## Architecture

```
Outlook (folder: Excepciones-Triage)
        │  trigger: When a new email arrives (V3)
        ▼
  Power Automate
        │  HTTP POST (Premium connector)
        ▼
  Azure Logic App (Consumption)
   ├─ HTTP → Phi-4-mini-instruct (Azure AI Foundry, Foundry Models / OpenAI v1 endpoint)
   ├─ Parse JSON (raw chat-completion response)
   ├─ Parse JSON (extracted decision object)
   └─ Response → back to Power Automate
        │
        ▼
  Condition on `decision`
   ├─ auto_aprobar → reply to sender
   └─ escalar / rechazar_alta_prioridad → notify supervisor mailbox
```

See [`docs/design-decisions.md`](docs/design-decisions.md) for the reasoning behind every non-obvious choice in this diagram.

## Decision policy

| Condition | Decision |
|---|---|
| `numero_po` missing, duplicate invoice, monto > $5,000, or discrepancy > 15% | `rechazar_o_escalar_alta_prioridad` |
| Monto < $500 **and** discrepancy < 5% | `auto_aprobar` |
| Everything else | `escalar` |
| Model confidence < 85%, regardless of the above | forced to `escalar` |

The confidence override is the most important rule in the system: it is what prevents a small, occasionally-wrong model from ever auto-approving something it isn't sure about.

## Stack

- **Power Automate** — email trigger, only (kept off the premium HTTP path where avoidable)
- **Azure Logic Apps (Consumption)** — orchestration and the AI call, chosen specifically to avoid the dedicated-compute quota required by Logic Apps Standard
- **Azure AI Foundry — Phi-4-mini-instruct** — extraction, classification, and decision recommendation, via serverless (pay-per-token) deployment
- **Azure for Students subscription** — individually owned, so the project's infrastructure lifecycle isn't tied to institutional access

## Repo structure

```
approval-exception-triage/
├── README.md
├── docs/
│   ├── design-decisions.md      — every non-obvious architectural/debugging decision, with reasoning
│   └── cost-notes.md            — cost controls and budget approach
├── flows/
│   └── logic-app-workflow.json  — sanitized Logic App definition (credentials removed)
├── prompts/
│   └── extraction-classification-prompt.md
├── sample-data/
│   └── expected-outputs.json    — test cases used to validate the three decision branches
└── audit-log-schema/
    └── table-schema.md
```

## What I'd do differently

- **Key Vault for the API key.** The Phi-4-mini-instruct API key currently lives directly in the Logic App's HTTP action header (with Secure Inputs/Outputs enabled to keep it out of run-history logs). The correct production pattern is Key Vault + Managed Identity — I set up the Key Vault and the role assignment (`Key Vault Secrets User`) but made a conscious call to keep the key inline for this iteration to keep momentum. Documented here rather than hidden.
- **Document Intelligence for attachments.** Right now the system reasons over the email body text only; PDF/image attachments aren't parsed. Wiring in Azure AI Document Intelligence to OCR/extract attachment content before it reaches the model is the natural next step.
- **Notification subject line.** The escalation email currently reuses text from the original subject line. Since the trigger filters on subject-line content, this created a theoretical (and briefly, an apparent) self-triggering loop risk if a notification ever landed back in the monitored folder. Fixed by making sure the notification subject never contains the trigger phrase.
- **System-role prompting.** The Phi-4-mini-instruct deployment silently dropped the `system` role message (visible via `prompt_tokens` in the response staying suspiciously low) — the fix was fusing instructions and data into a single `user` message. Worth revisiting if/when this moves to a model with confirmed system-role support.

## Status

End-to-end validated: three decision branches (auto-approve, escalate, reject/escalate) tested and confirmed against the policy table above, through the full chain from email trigger to Logic App decision to notification.
