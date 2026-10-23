# SkoFy — Steering Doc

This file is the always-loaded index. Detailed docs live in `docs/` — read the specific one you need rather than everything, to keep context cheap.

| Doc | Read this when... |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | You need to know how the system is actually built today — stack, repo layout, data model, request flow |
| [docs/VISION.md](docs/VISION.md) | You need the business thesis — C2C model, Universal HCI, anti-monopoly design, moat |
| [docs/AI_ENGINE_SPEC.md](docs/AI_ENGINE_SPEC.md) | You need the *intended* design of the 6 AI components (HCI Engine, Distribution/Hiring Agents, Skill Test, Feasibility, Support AI) |
| [docs/IMPLEMENTATION_STATUS.md](docs/IMPLEMENTATION_STATUS.md) | You need to know what's actually built vs. stubbed vs. missing — code-verified, the most likely to drift, check dates |
| [docs/ROADMAP.md](docs/ROADMAP.md) | You're planning what to build next |
| [docs/TECH_DEBT.md](docs/TECH_DEBT.md) | You're touching an area with known, deliberately-deferred debt |
| [docs/CALLING_SYSTEM.md](docs/CALLING_SYSTEM.md) | You're touching voice calls, ringing, call notifications, CallKit/PushKit or the call signalling — it has the target model, every case audited, and the fix list |
| [docs/JOB_SHAPE_DESIGN.md](docs/JOB_SHAPE_DESIGN.md) | You're adding a new kind of job/situation, or touching job matching, steps, pricing or the AI job summary — jobs are described by shape (venue/timing/custody/pricing), not by named categories |
| [docs/CALL_RECORDING_DESIGN.md](docs/CALL_RECORDING_DESIGN.md) | You're touching call audio, the media server (`dodorez-media`, self-hosted LiveKit), call recordings or their retention — components, lifecycles, access, consent, rollout |
| [docs/MONEY_RULES_DESIGN.md](docs/MONEY_RULES_DESIGN.md) | You're touching fees/commission, checkout, cancellations and who gets the visiting fee, inspection completion, disputes, strikes, chat screening for paying outside the app (§10), added work, tips, the guarantee, chargebacks, payouts or repeat bookings (§11–13) — Fair Cap replaces the flat 12%; built 2026-10-21, some parts switched off pending the lawyer (see its Status) |
| [docs/LEGAL_QUESTIONS.md](docs/LEGAL_QUESTIONS.md) | You're building something with legal exposure (classification, carrying goods, recording, licensed work, ID data), or the user mentions the lawyer — add new questions there |
| [docs/BUSINESS_PROVIDERS_DESIGN.md](docs/BUSINESS_PROVIDERS_DESIGN.md) | You're touching business providers (companies with staff), team members, assigning jobs to a member, the owner's view or tracking of employees, or one person working both for a business and for themselves — built 2026-10-23, switched off (`BUSINESS_ACCOUNTS_ENABLED`) until the lawyer confirms |
| [docs/SUPPORT_SYSTEM.md](docs/SUPPORT_SYSTEM.md) | You're touching support tickets, escalation, support levels or departments, or who can see/handle a ticket — topics → departments, per-department levels, moves, the safeguards |

## Critical facts to hold in mind

- **Three repos, one backend**: `skofy-backend/` (FastAPI) serves both `skofy-customer-app/` and `skofy-service-provider-app/` (Expo Router/React Native). No shared code between the two mobile apps.
- **Launch market is the USA, not India.** The two source strategy documents (`skofy_definitive_vision.html`, `SKOFY_AI_ENGINE.html`) are written entirely around an India go-to-market (₹, Hyderabad, Razorpay, MSG91, NSDC). The *model* (C2C, Universal HCI, multi-vertical) still holds; the *specifics* don't transfer as-is. See [docs/VISION.md](docs/VISION.md)'s mismatch callout before treating those source docs literally. Don't introduce India-specific vendors (KYC, payments, SMS) into new work.
- **The AI engine is mostly unbuilt.** Of the 6 spec'd components, only the Skill Test Agent is solidly built (and exceeds spec). HCI Engine, Hiring Agent, Feasibility Engine, and Support AI are stubs or missing. Distribution Agent's core path works; its auto-expand/fairness-boost periphery doesn't. Don't assume scoring/ranking exists just because the spec describes it — check [docs/IMPLEMENTATION_STATUS.md](docs/IMPLEMENTATION_STATUS.md).
- **Payments are live in Stripe test mode; provider payouts aren't.** The payments router is mounted and customer charges + escrow work. Stripe Connect isn't enabled on the Stripe account, so real payouts to providers can't happen yet; `DEMO_PAYOUTS=true` in `.env` (test keys only) records payouts without sending them, for demos.
- **The admin panel exists** — `dodorez-admin-portal/` (Vite + React): providers, customers, identity verification, disputes, support tickets, broadcasts, reports, roles.
- **Groq, not Claude, runs the AI endpoints.** Models are settings in `config.py` (`GROQ_MODEL`, `GROQ_VISION_MODEL`, `GROQ_SAFETY_MODEL`) because Groq retires models without notice.
- **Known debt is deliberately deferred** until after feature work + KYC vendor integration. Don't proactively suggest fixing it unless asked or directly in the way.

## Source documents (kept, not replaced)

`skofy_definitive_vision.html` and `SKOFY_AI_ENGINE.html` at the repo root are the original, full-detail strategy/spec documents — keep them as-is for prose/visuals. The `docs/` files above are the token-cheap reference versions of the same content, reconciled against actual code and the US-launch decision. `skofy_architecture.html` is a standalone visual version of `docs/ARCHITECTURE.md` for sharing/presenting.
