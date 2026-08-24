# Cost notes

This project is built under a hard constraint: minimize Azure spend at every step, and always know what a given action will create before running it.

## Cost-relevant decisions

- **Logic Apps Consumption**, not Standard — billed per execution (fractions of a cent each), not per hour of dedicated compute.
- **Phi-4-mini-instruct, Serverless API deployment**, not Managed Compute — billed per token (~$0.07 / $0.23 per million input/output tokens) rather than per GPU-hour.
- **Azure for Students subscription** — individually owned credit, not shared/institutional budget.
- Power Automate's HTTP (Premium) action was covered by the 90-day trial rather than a paid license, since this is a demo/portfolio flow, not a production workload with sustained volume.

## Recommended before running this yourself

- Set a budget alert on the resource group (a low threshold, e.g. $5–10 USD, is more than sufficient for demo-level usage of this stack).
- Be aware that rapid repeated testing against a serverless model deployment can hit its requests-per-minute limit (20 req/min on this deployment's tier) — this doesn't cost extra, but it can produce confusing retry-driven timeouts if you're mid-debugging (see `docs/design-decisions.md`).
- Turn off / disable the Power Automate flow when not actively testing or demoing, since it's the always-listening piece of the system.
