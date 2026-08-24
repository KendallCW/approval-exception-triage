# Design decisions

This document captures the reasoning behind the non-obvious choices in this project, including the debugging path that led to some of them. The goal is to be honest about what didn't work on the first try, not just what the final architecture looks like.

## Why Logic Apps Consumption, not Standard

First attempt at creating the Logic App failed with `SubscriptionIsOverQuotaForSku` on `WorkflowStandard VMs` — the student/institutional Azure subscription had a hard quota of 0 for the dedicated compute that Standard plans require. Consumption plans are serverless (billed per execution, no dedicated VM), so they sidestep this quota entirely. This is also a better cost fit for a portfolio project: near-zero cost when idle.

## Why Phi-4-mini-instruct instead of a GPT model

- Cost: roughly $0.07 / $0.23 per million input/output tokens, versus several dollars per million for GPT-4o-class models — effectively free at portfolio-demo volume.
- Narrative: demonstrates working with Microsoft's own model family via Azure AI Foundry's model catalog, not just wrapping the OpenAI API.
- Deployed via **Serverless API (Global Standard)**, not Managed Compute — Managed Compute bills by dedicated GPU-hour and would have hit the same kind of quota wall as Logic Apps Standard did.

## Why the system-role message was removed

During testing, the model returned content completely disconnected from the input (e.g., a generic "weekend report" instead of an invoice exception) even after the prompt was reinforced multiple times. The `usage.prompt_tokens` field in the raw HTTP response was the diagnostic signal: it stayed at ~16 tokens per call, far too small to include the several-hundred-token system prompt. This indicated the endpoint was silently discarding the `system`-role message rather than erroring on it — a known limitation on some smaller-model serverless deployments.

**Fix:** merged the system instructions (rules, output format, one-shot example) and the user's actual data into a single `user`-role message. After this change, `prompt_tokens` rose to the expected range and the model's output became consistent with the real input.

## Why a one-shot example was added to the prompt

Instruction-only prompting (rules + format spec, no example) was not enough to reliably anchor a small model's output to the task. Adding one concrete input → output example resolved this. This is a general lesson for small-model prompting: explicit examples do more work than additional abstract instructions.

## Why `response_format: json_object` is present but not solely relied upon

It was added as a secondary safeguard once available, but the project doesn't assume it's honored — smaller models sometimes accept the parameter without actually enforcing constrained decoding. The prompt-level instructions ("first character must be `{`, last must be `}`") remain the primary mechanism.

## Why the API key lives in the HTTP action header, not Key Vault (for now)

A Key Vault (`kv-approval-triage-dev`) was provisioned and a `Key Vault Secrets User` role assignment was configured for the Logic App's managed identity — the groundwork for the correct pattern is in place. The key itself, however, is currently pasted directly into the HTTP action's header, with **Secure Inputs/Outputs** enabled on that action so the value doesn't appear in run-history logs. This was a deliberate scope decision to keep the project moving; wiring in the actual `Get secret` action is tracked as a documented next step, not an oversight.

Any exported/committed version of the Logic App definition in this repo has the key replaced with a placeholder — see `flows/logic-app-workflow.json`.

## Why Consumption Logic Apps don't support the "default value vs. secure runtime value" pattern

Initially assumed workflow parameters could be declared as `securestring` with no default and a separate runtime-only value (a pattern valid in Logic Apps *Standard*). In Consumption, parameter values are stored directly in the resource definition — there's no separate secure runtime store. This is why the API key ended up directly in the action instead of behind a workflow parameter.

## Why the rate-limit / retry-policy interaction mattered for debugging

Several "timeout" failures during testing were not caused by the model or the network — they were caused by the Logic App's **default retry policy** silently retrying the HTTP action up to 4 times (with exponential backoff) whenever it received a `429` (rate limit exceeded) from the model endpoint, which happens fast during rapid iterative testing (the model's tier allows 20 requests/minute). A failed run that actually takes 2–4 minutes to fail (versus the normal ~3 seconds to succeed) is a strong signal of hidden retries, not a real connectivity problem. Confirmed by testing the model endpoint directly (outside any flow) and getting an instant, correct response.

## Why the trigger lives on the university Outlook account, not a personal one

Considered creating a dedicated personal mailbox to avoid any dependency on institutional account lifecycle. Decided against it after confirming the university account will remain active — simplifies the setup without the downside that made this a concern in the first place (accounts being deactivated post-graduation).

## Why the trigger filters by folder + subject text, and the risk that created

The trigger only fires on emails in a dedicated `Excepciones-Triage` folder with a specific subject-line pattern. Early in testing, a burst of near-identical-subject test emails looked like a self-triggering loop (each escalation notification appeared to retrigger the flow) — investigation showed it was actually several independently-sent test emails threaded together by Outlook's conversation view, not a real loop. Still, the underlying risk is real: if a notification email's subject retains the trigger phrase and ever lands in the monitored folder, a genuine loop is possible. Mitigated by keeping the notification subject line free of the trigger phrase entirely.
