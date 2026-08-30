# OpenAI Model Selection Recommendation

Prepared: 2026-08-10

## Recommendation

Use `gpt-5.6-terra` with `medium` reasoning as the everyday default. Keep `gpt-5.6-luna` for low-risk, high-volume work and use `gpt-5.6-sol` only when the task is unusually difficult or operationally high-risk.

Current local configuration was:

```toml
model = "gpt-5.6-sol"
model_reasoning_effort = "xhigh"
```

## Evidence reviewed

The activity window was 2026-08-03 through 2026-08-10.

- GitHub: 11 repositories pushed, 88 authored commits on default branches, and 10 pull requests opened and merged.
- Local checkout: 101 commits reachable across local refs, approximately 311 repository-file entries touched, and roughly 89,658 insertions versus 4,921 deletions.
- Large line-volume changes were generated ontology, SHACL, schema, fixture, audit, and release artifacts. Substantive work was mainly Python, shell, YAML, Docker Compose, CI, TypeScript, and documentation.
- Representative work included stateful Docker-volume changes, platform Airflow, Proxmox and Cloudflare deployment work, federation routing, LinkML/SHACL contracts, atomic PostgreSQL promotion and rollback, and release verification.

This is not primarily frontier algorithm research, but it is context-heavy and operationally consequential. Terra is a safer everyday balance than Luna, while Sol is unnecessary for most individual tasks.

## Routing policy

| Work | Model | Reasoning |
| --- | --- | --- |
| Read-only status, log summaries, inventories, straightforward documentation, simple YAML edits | `gpt-5.6-luna` | `low` or `medium` |
| Normal implementation, debugging, tests, CI, Docker, Compose, and one-to-two-repository changes | `gpt-5.6-terra` | `medium` |
| Cross-repository architecture, production data promotion or rollback, security, recovery, or ambiguous ownership | `gpt-5.6-sol` | `high` or `xhigh` |

Escalate from Terra when the task crosses several repositories, can damage production state, or fails verification twice.

## Cost comparison

Current official API prices per one million tokens:

| Model | Input | Output | Relative to Sol |
| --- | ---: | ---: | ---: |
| GPT-5.6 Sol | $5.00 | $30.00 | 100% |
| GPT-5.6 Terra | $2.00 | $12.00 | 40% |
| GPT-5.6 Luna | $0.20 | $1.20 | 4% |

A practical mix is approximately 70% Terra, 20% Luna, and 10% Sol. Assuming similar token usage, that is about 60% lower model cost than using Sol for every task. API prices do not necessarily equal Codex subscription-plan accounting.

## Automatic model selection

OpenAI's current official API guidance documents explicit model selection. The `gpt-5.6` alias routes to `gpt-5.6-sol`; it is not a task-aware cost router. I found no official API setting that automatically chooses Sol, Terra, or Luna from request risk or complexity.

Use Terra as a fixed default with explicit escalation, or implement a small client-side router using task labels such as `routine`, `implementation`, and `high-risk`. If a Codex client displays an `Auto` option, treat it as client-specific until its selected model and billing are documented for your account.

## Suggested default

```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
```

## Official OpenAI sources

- [Model guidance](https://developers.openai.com/api/docs/guides/latest-model)
- [GPT-5.6 Sol](https://developers.openai.com/api/docs/models/gpt-5.6-sol)
- [GPT-5.6 Terra](https://developers.openai.com/api/docs/models/gpt-5.6-terra)
- [GPT-5.6 Luna](https://developers.openai.com/api/docs/models/gpt-5.6-luna)
