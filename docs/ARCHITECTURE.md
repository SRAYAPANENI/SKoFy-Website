# SkoFy — Architecture (as implemented)

> Code-derived. Last verified: 2026-06-23. If this drifts from the code, the code wins — re-verify before trusting line/file references older than a few weeks.

See also: [AI_ENGINE_SPEC.md](AI_ENGINE_SPEC.md) (intended design of the AI layer) · [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md) (spec vs. what's actually built) · [TECH_DEBT.md](TECH_DEBT.md).

## Repo layout

Three repos under one root, no monorepo tooling tying them together:

| Path | What |
|---|---|
| `skofy-backend/` | FastAPI service — the only backend, serves both apps |
| `skofy-customer-app/` | Expo Router app, customers post jobs / hire |
| `skofy-service-provider-app/` | Expo Router app, providers bid / work jobs |

## Stack

- **API**: FastAPI 0.115 (async), Pydantic v2, Uvicorn
- **DB**: PostgreSQL + PostGIS 16-3.4, SQLAlchemy 2.0 async ORM, Alembic migrations
- **Async/cache**: Redis 7 (cache + Celery broker/result backend), Celery worker + beat
- **Auth**: JWT (`python-jose`) — 15-min access / 30-day refresh, role embedded in token payload; bcrypt password hashing
- **Mobile**: Expo Router 6, React Native 0.81, React 19 — separate codebases, same API contract
- **AI**: Groq-hosted models, all set in `config.py` (verified 2026-10-06): `GROQ_MODEL` (`openai/gpt-oss-120b`) for voice chat and text problem analysis, `GROQ_VISION_MODEL` (`qwen/qwen3.8-27b`) for photo analysis, `GROQ_SAFETY_MODEL` (`openai/gpt-oss-safeguard-20b`) to screen voice-chat input, `whisper-large-v3` for speech-to-text. Groq has retired models without notice before (llama-guard-3-8b, llama-4-scout) — that's why each is a setting. `CLAUDE_MODEL`/Anthropic keys exist in config but are unused.
- **Payments**: Stripe (config placeholder wired; see [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md) — router not mounted yet)
- **Comms**: Twilio (OTP SMS), Cloudinary (image/doc uploads), Firebase FCM (push)

US-market vendor choices (Stripe/Twilio, `Provider.state` as 2-letter US abbreviation) are deliberate — see [VISION.md](VISION.md) market-mismatch callout before assuming anything India-specific.

## Backend layout (`skofy-backend/app/`)

Layered: routers parse/validate → services hold business rules → repositories own SQL → models map tables.

| Layer | Contents |
|---|---|
| `api/v1/` | One router per domain: `auth`, `customers`, `providers`, `skills`, `jobs`, `ai`, `chat`, `ws`, `health`. Mounted in `router.py`. `payments` and `tracking` routers exist as commented-out imports ("Week 6+" / "Week 7+") — **not live**. |
| `services/` | `auth_service`, `customer_service`, `provider_service`, `job_service`, `distribution_agent`, `skill_test_service`, `chat_service` |
| `core/` | `config` (pydantic `Settings` from `.env`), `database`, `redis_client`, `security` (JWT/hashing), `ws_manager` (in-memory WebSocket registry — **single-process only**, no Redis pub/sub backing), `dependencies`, `exceptions`, `logging_config` |
| `models/` + `repositories/` | SQLAlchemy models (UUID + timestamp mixins) paired 1:1 with repositories for queries |
| `tasks/` | Celery tasks: `otp_tasks`, `job_tasks`, `notification_tasks`, `hci_tasks` |
| `agents/` | Empty package — scaffold only, no code. The real distribution logic lives in `services/distribution_agent.py`, not here. |

## Data model (13 tables)

| Table(s) | Purpose |
|---|---|
| `users` / `customers` / `providers` | Base identity + role profile (rating, `hci_score`, verification status) |
| `addresses` | Saved lat/lng locations |
| `provider_slots` / `provider_documents` | Availability windows; KYC/document uploads |
| `skills` / `provider_skills` | Skill catalog; per-provider verified skills |
| `skill_questions` / `skill_test_attempts` | Claude-generated MCQ bank + attempt history |
| `jobs` | Central record: PostGIS location, status, urgency, budget, distribution radius, payment status |
| `job_skills` | Required skills per job |
| `job_distributions` | Which providers were notified, with a match score |
| `job_applications` | Provider bids |
| `reviews` | Post-job rating, feeds HCI |
| `chat_messages` | Per-job message history |
| `payments` | Stripe intent + platform-fee bookkeeping (model exists, route not mounted) |
| `otp_logs` | Rate-limited OTP attempts |
| `refresh_tokens` / `fcm_tokens` | Auth + push tokens |
| `agent_decisions` / `hci_score_history` | Audit trail for agent decisions and score changes over time |

Job status machine: `POSTED → DISTRIBUTED → ACCEPTED → IN_PROGRESS → COMPLETED`, with `CANCELLED` / `DISPUTED` off-ramps.

## Core flow: life of a job

1. **Customer describes the problem** — `POST /v1/ai/analyze-problem`: photos/text → Groq (vision model when photos are attached) → structured `{problem_summary, profession, skills, urgency}`. Or by voice: `/ai/transcribe` + `/ai/voice-chat`, in the customer's chosen language (12 supported; picker in the voice screen).
2. **Job posted** — `POST /v1/jobs`: creates row with PostGIS point, required skills, budget, urgency. Status `POSTED`.
3. **Distribution Agent matches providers** (`distribution_agent.py`, verified in code):
   - **City-wide by default** (`DISTRIBUTION_CITY_WIDE`, 2026-10-06): every qualified, available provider within `DISTRIBUTION_CITY_RADIUS_KM` (50km), nearest first, ignoring providers' own `preferred_radius_km` — providers see distance and an estimated drive time and decide. Capped at `DISTRIBUTION_CITY_MAX_PROVIDERS` (300) notifications per job. With city-wide off, the original search applies: `DISTRIBUTION_INITIAL_RADIUS_KM` (5km), retried at `DISTRIBUTION_MAX_RADIUS_KM` (25km) if fewer than `DISTRIBUTION_MIN_PROVIDERS` found.
   - An on-site job with no location falls back to the customer's default saved address (`JobService.create_job`); with no address it's rejected — it used to be saved and never distributed.
   - Provider must qualify on `is_available`, have a location, and have verified skills covering ≥ `JOB_SKILL_MATCH_THRESHOLD` (0.5) of the job's required skills.
   - Score formula: `dist_factor*40 + hci_score*0.3 + match_ratio*30` (distance-heavy; no fairness-boost term currently).
   - Capped at `DISTRIBUTION_MAX_PROVIDERS` (10), recorded in `job_distributions`.
4. **Providers bid** — notified providers see the job, submit a `JobApplication`. Customer sees applicants **in application order** (`ORDER BY ja.created_at`) — no ranking/scoring is applied here (see gap in [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md)).
5. **Customer hires** — sets `assigned_provider_id`, `hired_at`.
6. **Chat & tracking** — WebSocket at `/v1/ws`, keyed by job + user id via `ConnectionManager`.
7. **Completion / payment / review** — status → `COMPLETED`; `Review` feeds the (currently stubbed) HCI recalculation.

Scheduled/background work (Celery beat, intended — see stubs below): `expand_stalled_distributions` (every 15 min) and `recalculate_all_active_providers` (nightly) are both defined but **log-only stubs today**.

## Infrastructure (Docker Compose, `skofy-backend/docker-compose.yml`)

| Service | Image/command | Role |
|---|---|---|
| `db` | `postgis/postgis:16-3.4` | Primary datastore |
| `redis` | `redis:7-alpine` | Cache + Celery broker |
| `api` | `uvicorn app.main:app --reload` | HTTP + WS server, port 8000 |
| `celery_worker` | `celery worker --concurrency=4` | Async tasks |
| `celery_beat` | `celery beat` | Scheduled tasks |

Schema changes via Alembic (`migrations/`). One Dockerfile builds the image used by all four app containers.

## Mobile apps

Both are Expo Router file-based-routing apps hitting the same backend; role lives in the JWT, not in separate codebases for the API.

**`skofy-customer-app`**: post-requirement wizard (`step1 → step2 → payment → summary`), `my-jobs`, `job-history`, `applicants`, `offers`, `track-provider`, `chat`, `provider-details`. Uses `react-native-maps`, `expo-location`, `react-native-webrtc`, `expo-secure-store`.

**`skofy-service-provider-app`**: `jobs` tab (inbound/bidding), `job-details`, `customer-profile`, `messages` tab, `chat`, `support-chat`.

## Known structural limitations (verified in code)

- `ws_manager.ConnectionManager` is in-memory and single-process — won't survive multi-worker deployment without a Redis-backed pub/sub layer.
- `app/agents/` package is empty — currently aspirational, not where matching logic actually lives.
- `payments` and `tracking` API routers are written but not mounted (`router.py` comments: "Week 6+" / "Week 7+").

For the AI-layer gap analysis (HCI engine, Hiring Agent, Feasibility Engine, Support AI — what's spec'd vs. what's actually live), see [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md).
