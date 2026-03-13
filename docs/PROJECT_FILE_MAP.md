# Project File Map (JobIntel)

This document captures a complete file-by-file understanding of the repository (excluding `__pycache__`).

## Runtime entrypoints and environment

- `manage.py`: Django CLI entrypoint; loads `.env`, ensures PostgreSQL DB exists (`ensure_database`), then executes management commands.
- `bootstrap_db.py`: standalone database bootstrap helper with the same database-creation routine used in `manage.py`.
- `requirements.txt`: Python dependencies for Django app, Gmail API integrations, OpenAI, Celery/Redis stack, and supporting libraries.

## Project package: `jobintel/`

- `jobintel/__init__.py`: package marker.
- `jobintel/settings.py`: primary Django settings (apps, middleware, templates, Postgres config, static/media, login redirects, OAuth/OpenAI settings).
- `jobintel/settings1.py`: alternate settings variant kept in repo (legacy/backup style duplicate of main settings).
- `jobintel/urls.py`: root URLConf routing to dashboard/home and app URL namespaces.
- `jobintel/urls1.py`: alternate/legacy URLConf variant kept in repo.
- `jobintel/asgi.py`: ASGI application bootstrap.
- `jobintel/wsgi.py`: WSGI application bootstrap.
- `jobintel/context_processors.py`: exposes Gmail connection state (`gmail_connected`) for authenticated users to templates.

## App: `accounts/` (auth + Gmail OAuth token storage)

- `accounts/__init__.py`: package marker.
- `accounts/apps.py`: app config.
- `accounts/admin.py`: registers `GmailToken` for Django admin.
- `accounts/models.py`: defines `GmailToken` (OAuth credentials, scopes, expiry, active/error flags, Gmail history pointer, user one-to-one link).
- `accounts/views.py`: signup/login/logout + Google OAuth start/callback; persists tokens; triggers Gmail watch/fetch/process after connect.
- `accounts/urls.py`: routes signup/login/logout/Gmail connect/callback endpoints.
- `accounts/tests.py`: test scaffold placeholder.
- `accounts/migrations/__init__.py`: migrations package marker.
- `accounts/migrations/0001_initial.py`: creates `GmailToken` base schema.
- `accounts/migrations/0002_gmailtoken_is_active_gmailtoken_last_error.py`: adds token activity/error fields.
- `accounts/migrations/0003_gmailtoken_last_history_id.py`: adds history-id tracking field.
- `accounts/migrations/0004_rename_last_history_id_gmailtoken_gmail_history_id.py`: renames history field to `gmail_history_id`.

## App: `gmail_sync/` (Gmail ingestion + webhooks)

- `gmail_sync/__init__.py`: package marker.
- `gmail_sync/apps.py`: app config.
- `gmail_sync/admin.py`: admin module scaffold.
- `gmail_sync/models.py`: model module scaffold (domain models live in `jobs`).
- `gmail_sync/services.py`: Gmail API service creation, MIME decoding/body extraction, message persistence to `RawEmail`, recent/incremental fetch, watch initialization.
- `gmail_sync/views.py`: manual fetch endpoint and webhook endpoint handling Pub/Sub payload and incremental sync trigger.
- `gmail_sync/webhooks.py`: webhook handler variant for push notifications with token checks and incremental fetch/process execution.
- `gmail_sync/pubsub.py`: helper for starting Gmail watch subscription.
- `gmail_sync/urls.py`: routes fetch and webhook endpoints.
- `gmail_sync/tests.py`: test scaffold placeholder.
- `gmail_sync/migrations/__init__.py`: migrations package marker.

## App: `jobs/` (core data model + dashboard/thread UX)

- `jobs/__init__.py`: package marker.
- `jobs/apps.py`: app config.
- `jobs/models.py`: core models:
  - `RawEmail`: ingested Gmail messages (metadata, body, processed/classification flags).
  - `JobThread`: canonical thread state (company, role, status, timestamps, follow-up state).
  - `ThreadEvent`: timeline events linked to thread and source email.
- `jobs/admin.py`: rich Django admin for thread/event/email browsing with list filters/search/inlines.
- `jobs/views.py`: authenticated dashboard analytics + thread detail + follow-up dismissal endpoint.
- `jobs/urls.py`: routes dashboard, thread detail, and dismiss-followup actions.
- `jobs/tests.py`: test scaffold placeholder.
- `jobs/migrations/__init__.py`: migrations package marker.
- `jobs/migrations/0001_initial.py`: creates `JobThread`, `ThreadEvent`, and `RawEmail` schemas.
- `jobs/migrations/0002_alter_threadevent_options.py`: adds ordering options on `ThreadEvent`.
- `jobs/migrations/0003_rawemail_body_text.py`: adds extracted `body_text` field.
- `jobs/migrations/0004_alter_jobthread_status_alter_threadevent_event_type.py`: updates status/event choice constraints.
- `jobs/migrations/0005_rawemail_classification_source_and_more.py`: adds classification provenance/confidence flags.
- `jobs/migrations/0006_jobthread_followup_dismissed.py`: adds per-thread follow-up dismiss state.

## App: `intelligence/` (classification + thread synthesis)

- `intelligence/__init__.py`: package marker.
- `intelligence/apps.py`: app config.
- `intelligence/admin.py`: admin module scaffold.
- `intelligence/models.py`: model module scaffold (logic operates on `jobs` models).
- `intelligence/classifier.py`: heuristic text classifier (`classify_email`) with job-context scoring and keyword signals.
- `intelligence/semantic_classifier.py`: embedding-assisted classification via OpenAI embeddings + cosine similarity against labeled exemplars.
- `intelligence/processor.py`: orchestration pipeline to process unprocessed `RawEmail`, classify, create/update `JobThread` + `ThreadEvent`, infer company/role, compute follow-ups.
- `intelligence/views.py`: endpoint to manually invoke processing.
- `intelligence/urls.py`: routes processing endpoint.
- `intelligence/tests.py`: test scaffold placeholder.
- `intelligence/migrations/__init__.py`: migrations package marker.

## App: `followups/` (follow-up draft generation + improvement)

- `followups/__init__.py`: package marker.
- `followups/apps.py`: app config.
- `followups/templates_engine.py`: static template text blocks used by follow-up generation.
- `followups/context_builder.py`: builds lightweight context payload from `JobThread` and associated events.
- `followups/generator.py`: template selection and follow-up draft rendering using context.
- `followups/ai_service.py`: AI rewrite/improvement call wrapper for follow-up text.
- `followups/views.py`: AJAX endpoints to generate and improve follow-up drafts.
- `followups/urls.py`: routes generate/improve follow-up endpoints.

## Global templates (`templates/`)

- `templates/base.html`: shell layout, navigation, auth controls, Gmail connection badge/button.
- `templates/login.html`: login form view.
- `templates/signup.html`: signup form view.
- `templates/dashboard.html`: dashboard KPIs/charts/listings and thread navigation.
- `templates/thread_detail.html`: thread timeline and interactive follow-up generation/improvement UI.
- `templates/thread_detail1.html`: alternate/legacy thread detail template variant.

## Architecture summary

1. User authenticates and connects Gmail through OAuth (`accounts`).
2. Gmail data sync is initialized (watch + fetch) via `gmail_sync` services/webhooks.
3. Ingested raw emails are classified and transformed into structured thread/event records by `intelligence`.
4. Dashboard and thread views in `jobs` render analytics + timeline state.
5. `followups` generates and AI-improves follow-up drafts based on thread context.
