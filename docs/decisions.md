[Русский](./decisions.ru.md) · **English**

# Architecture Notes — Call Analytics System

> **Disclaimer.** This is a public architectural description of a real
> system the author worked on. Specific clients, domain names,
> financial indicators, source code, and proprietary implementation
> details are not disclosed. The content is limited to architectural
> decisions and principles publicly discussed for systems of this kind.

Walkthrough of key architectural decisions: context (what was being
solved) and implementation (how it was solved).

---

## 1 · Call processing — a single Celery task with an orchestrator

### Context

Processing one call is a sequence of 5–7 steps: ingestion,
preprocessing, STT, diarization, LLM analysis, postprocessing, result
push.

### Implementation

Processing one call is a single Celery task. Inside it,
`PipelineOrchestrator` runs the steps sequentially. The final state is
`call_analysis.status` (`pending` / `processing` / `completed` /
`failed`).

Recovery works through write idempotence:
`INSERT ... ON CONFLICT (external_call_id, project_id) DO UPDATE`.
Any rerun of processing for the same call updates the existing record
and doesn't create a duplicate. A failed call can be re-run safely:
manually from the admin UI or via an auto-recovery task that detects
records «stuck» in `processing` and returns them to the queue.

---

## 2 · AI presets as encrypted JSON in `system_config`

### Context

Different clients score calls by different criteria (sales, technical
support, complaints) and connect different models (top-tier LLMs or
cheap self-hosted ones). An «AI preset» entity is needed, with the
ability to switch without releases.

### Implementation

Presets are stored in the `system_config` table under a fixed key
`ai_presets`. The value is a Fernet-encrypted JSON array. Presets are
created, edited, and activated via the UI. Sensitive fields
(`stt_api_key`, `diarization_api_key`, `llm_api_key`) are masked in
API responses; the full text is visible only at creation / edit time.
A project binds to an active preset by id; multiple presets can be
created and switched between.

---

## 3 · Staged table for manual call selection before processing

### Context

Not every call from a CRM is worth processing. Some are short, test,
or irrelevant. STT and LLM cost money; in some projects an admin wants
to decide themselves which calls go through.

### Implementation

When a call arrives from a CRM, it's inserted into `staged_calls` with
status `pending`. If the project has `auto_process=True`, the call is
immediately enqueued for Celery; otherwise, it waits for an admin's
manual action. Statuses: `pending | processing | completed | failed |
skipped`. The behavior is controlled by a single project-level flag
(fully auto vs manual selection).

---

## 4 · Project.config_json (JSONB) for project settings

### Context

A project stores several clusters of settings: CRM connection,
AI-preset choice and prompt parameters, access control, Telegram
notifications. Each of these can evolve independently.

### Implementation

`Project.config_json` (JSONB) holds a nested structure:
`connection`, `ai`, `prompt`, `access_control`, `telegram`. Pydantic
models (`CRMConnectionConfig`, `AIConfig`, `PromptConfig`,
`AccessControlConfig`, `TelegramConfig`) validate the structure on
save and read. Config changes are journaled in the audit log.

---

## 5 · On-premise readiness via provider abstractions

### Context

Some prospective clients work in compliance-sensitive industries
(healthcare, finance, government), where audio and transcripts cannot
leave the customer's perimeter. That means external SaaS providers for
STT/LLM cannot be used — local models in the customer's infrastructure
are required.

### Implementation

The architecture uses a single codebase for SaaS and on-premise.
Differences are at the config level:

- STT/LLM providers — via the AI-preset abstraction, switching to
  local services without code changes.
- Fernet secret encryption — local, no external KMS required.
- DB, queues, file storage — all locally hostable.
- CRM integrations — on the customer's side (if a CRM exists in
  their perimeter).

The current production install is in SaaS mode. To launch on-premise
at a customer: a deploy package (Docker images or systemd units),
installation docs, compliance checks per the industry.

---

## 6 · Webhook + polling under one CRM interface

### Context

Different CRMs deliver new calls differently: some send a webhook to
our endpoint when a call appears, others require polling on a schedule.

### Implementation

A registry (`integrations/registry.py`) holds implementations per CRM
type. Each implementation knows whether it supports webhooks, whether
it supports polling, and the data format. After ingestion (either
way), the call lands in `staged_calls` — from there a single path.

The CRM type (webhook vs polling) is determined in the adapter;
manual sync is unavailable for webhook-type CRMs (validated in the
API).

---

## 7 · Fernet encryption of sensitive DB fields

### Context

The system has several classes of sensitive data: API keys and tokens
of STT/LLM providers (in AI presets), CRM integration credentials,
proxy tokens/passwords.

### Implementation

Sensitive values are encrypted with `encrypt_value()` before write and
decrypted on read. The encryption key is stored in env; provider keys
and `system_config.value` for AI presets are wholly encrypted JSON
objects. In API responses, keys are masked (`****abcd`); plaintext is
visible only at creation/edit time. Backups and physical DB dumps
don't contain plaintext.
