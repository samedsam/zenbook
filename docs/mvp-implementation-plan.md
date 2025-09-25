# Rendez-vous Zen MVP Implementation Plan

## 1. Vision and Scope Alignment

The MVP targets seniors in France and their primary caregivers by providing a frictionless appointment companion available on Android (native Kotlin) and as a lightweight PWA. The first release must ship within **4 weeks**, covering passwordless onboarding, appointment logistics, proactive reminders, accessible checklists, and post-visit summaries while staying compliant with French accessibility and privacy expectations.

### Must-Have Outcomes
- Passwordless authentication (SMS OTP) for **Senior** and **Aidant** roles.
- Appointment lifecycle management with location, notes, reminders, and checklist association.
- Intelligent push/SMS reminders with configurable offsets and SMS quota enforcement.
- Trust circle (one caregiver) with shared notifications and summaries.
- Accessible itinerary deeplinks optimized for low-effort routes.
- Pre-built French checklists (5 templates) with interactive completion tracking.
- One-tap late notification SMS for offices with caregiver alerts.
- Local audio capture with EU-cloud transcription and text summary sharing.
- Accessibility baseline: 18–22 pt typography, high contrast theme, large touch targets, text-to-speech.

## 2. Release Milestones (4-Week Sprint Plan)

| Week | Objectives | Key Deliverables |
|------|------------|------------------|
| 1 | Authentication, core data model, appointment CRUD, essential UI flows | OTP backend endpoints, database schema/Flyway migrations, Android onboarding/auth/home/appointment creation screens, basic PWA list viewer |
| 2 | Notifications, caregiver linkage, checklist infrastructure, appointment detail view | Quartz jobs + FCM/SMS integration, caregiver consent flow, checklist templates + instantiation, appointment detail UI |
| 3 | Route assistance, late notifications, audio pipeline (transcribe & summarize) | Routing preferences + deeplinks, SMS late alert endpoint, S3 audio upload path, ASR provider integration, summary share flow |
| 4 | Accessibility polish, RGPD tooling, observability, beta preparation | High-contrast mode, font scaling, TTS hooks, data export/delete endpoints, instrumentation dashboards, pilot feedback loop |

## 3. System Architecture Overview

### Backend (Spring Boot 3, Java 21)
- **Modules**: Auth, User & Caregiver Management, Appointments, Notifications, Checklist, Visit Notes, Infrastructure (Flyway, Quartz, Observability).
- **Persistence**: PostgreSQL with provided schema, managed via Flyway migrations; soft delete strategy (`deleted_at`).
- **Storage**: S3-compatible bucket in EU for audio with 7-day lifecycle policies; transcripts/summaries persisted in Postgres.
- **Integrations**: SMS provider REST API with per-user quota enforcement and exponential backoff; FCM for push; OpenRouteService/Here for itinerary options; ASR/Summarization provider within EU.
- **Security**: JWT-based session, phone verification via OTP, TLS termination, secrets handled via Vault/SSM. Logging in JSON with Micrometer/Prometheus exporters.
- **Background Jobs**: Quartz scheduler storing jobs in Postgres; cron endpoint (`/internal/notifications/dispatch`) triggered by infra; ensures reminder offsets (J-2 10:00, J-1 18:00, J 2h before, J +30 min) and fallback logic (skip push if SMS succeeds).

### Mobile Android (Kotlin)
- **Architecture**: MVVM with Jetpack Compose for accessibility-friendly UI; WorkManager for background reminder sync and OTP polling fallback.
- **Key Screens**: Align with specification (Welcome, OTP auth, Home, New Appointment, Appointment Detail, Caregiver, History, Settings).
- **Accessibility**: Compose Material 3 with scalable typography, custom theme for high contrast, semantics for TalkBack, haptic feedback on critical actions.
- **Audio Handling**: Local recording with fallback for offline; upload via secure pre-signed URL; handle <5 min limit with progress messaging.
- **Notifications**: FCM tokens registered per device; support SMS fallback awareness (display if SMS consumed quota).
- **Offline Strategy**: Cache appointments/checklists locally via Room; queue updates for sync when network returns.

### PWA (Web)
- **Scope**: Read-only list of upcoming appointments and visit summaries for caregivers; built with React/TypeScript + Vite.
- **Auth**: Reuse JWT via Auth API, store in secure cookies with short TTL; silent refresh by OTP if expired.
- **Accessibility**: High contrast theme, large font defaults, ARIA roles for screen readers.

## 4. Data & Domain Model Notes
- Ensure referential integrity per schema, cascade deletes on reminders and visit notes when appointments are removed.
- Checklist templates seeded via Flyway; instances cloned at appointment creation to allow per-appointment customization.
- Visit notes tie to transcripts & summaries with audit fields for caregiver access logging.
- Enforce single caregiver link per senior in MVP using unique constraint on `caregiving_link.senior_id`.

## 5. Notification Rules & Quotas
- Default reminder offsets preconfigured; allow future extension to custom offsets via table entries.
- SMS quota: maintain per-user daily counters; log to `notification_event` table for audit (future extension).
- Aidant notification fan-out triggered on appointment lifecycle events and late alerts; ensure idempotent dispatch.

## 6. Compliance & Privacy Checklist
- Explicit consent screens for notifications, caregiver sharing, transcription.
- `DELETE /me` performs soft delete plus triggers async purge of audio objects.
- Audit log for caregiver reads (store user, appointment, timestamp).
- Data residency: ensure all third-party services have EU endpoints and DPAs.

## 7. Observability & QA
- Logging: structured JSON with correlation IDs (per request/job).
- Metrics: Micrometer + Prometheus (request latency, reminder dispatch success, SMS quota usage).
- Analytics: PostHog self-hosted events for activation metrics (RDV created, caregiver linked, summary shared).
- Testing: Unit (≥70% service layer), integration with MockWebServer for SMS/ASR, Espresso UI tests for critical flows, manual accessibility audit (RGAA) targeting ≥90% compliance.

## 8. Risk Mitigation
- **SMS Costs**: Track usage, surface to product; plan push-first fallback with voice reminders in backlog.
- **Senior Adoption**: Weekly UX test cadence, incorporate voice guidance, minimize steps (<3 screens per action).
- **ASR Privacy**: Offer offline mode fallback for sensitive users; auto-delete audio after 7 days.
- **Address Quality**: Provide manual override and background geocoding retries.

## 9. Next Steps
1. Finalize provider selections (SMS, ASR, routing) with EU compliance.
2. Produce API contracts in OpenAPI 3 for backend endpoints.
3. Set up CI/CD pipelines (GitHub Actions) with unit + integration tests and security scans.
4. Prepare beta testing plan (30 seniors, 10 caregivers) with feedback loops.
5. Create design system tokens to enforce accessibility from the start.

