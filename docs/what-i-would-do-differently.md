# What I Would Do Differently

This section is updated as the project progresses — not written retroactively at the end. Honest engineering retrospectives, including things that didn't work on the first attempt.

## Verify system-role support before designing the prompt around it

I wrote the prompt assuming a standard `system` + `user` message split would work, without first confirming the specific serverless deployment actually honored the `system` role. It didn't — the endpoint silently dropped it rather than erroring, and I only caught this because an abnormally low `prompt_tokens` count in the raw response looked wrong. Next time, I'd send one trivial test call and inspect the token usage before building a multi-message prompt structure on top of an assumption like that.

## Rule out rate-limit-driven retries before assuming a deeper bug

Several failed runs during testing looked like unexplained timeouts and briefly had me questioning the model, the network, and the Logic App configuration all at once. The actual cause was simpler: the Logic App's default retry policy was silently retrying a rate-limited (429) call up to 4 times with exponential backoff, turning a fast failure into one that looked like it took 2–4 minutes to "time out." The run-history timestamps had the answer the whole time — a ~3 second run and a ~3 minute run are not the same failure mode, and I'd check that distinction first next time, before testing the model or the network in isolation.

## Wire in Key Vault at the same time as the API call, not after

I provisioned the Key Vault and the Managed Identity role assignment (`Key Vault Secrets User`) before actually building the `Get secret` action into the Logic App — and then, once the core flow was working end-to-end, chose to keep the key inline (with Secure Inputs enabled) rather than go back and wire in the Key Vault call. That's a defensible scope decision, but it means the Key Vault currently sits unused in the resource group. Next time, I'd either build the Key Vault call in from the start, alongside the first working version of the HTTP action, or skip provisioning it until I was ready to actually use it — having it half-wired is the worst of both options.

## Keep the notification's subject line structurally separate from the trigger's filter

The escalation notification originally reused text from the original email's subject line, and the trigger filters on subject-line content — a combination that creates a real, if narrow, self-triggering loop risk if a notification ever lands back in the monitored folder. A burst of test emails briefly looked exactly like that loop happening (it turned out to be several independent test sends, not a real loop, but the moment of genuine uncertainty was the useful part). I'd design the notification's subject to never overlap with the trigger's filter text from the first draft, rather than noticing the risk after building it the other way.

## Confirm the deployment plan type before creating a resource, not after

The first Logic App creation attempt failed on a quota error for `WorkflowStandard VMs` — I hadn't checked, going in, that Standard plans require dedicated compute quota the student subscription didn't have. Consumption was the right choice from the start for a project this size and cost-sensitivity; I'd confirm plan-type quota availability before the first creation attempt next time, rather than discovering the constraint through a failed deployment.
