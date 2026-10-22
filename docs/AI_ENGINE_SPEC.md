# SkoFy — AI Engine Spec (intended design)

> Condensed from `SKOFY_AI_ENGINE.html` (v1.0, June 2026). This is the **design intent** for the AI layer — not a status report. For what's actually built vs. stubbed, see [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md).

## Premise

Six purpose-built decision systems, not a chatbot. Every other home-services platform uses `category + location = show providers`. SkoFy's spec replaces that with behavioral scoring, real-time eligibility checks, AI-verified skill assessment, and intelligent applicant ranking.

**Key design constraint**: zero ML training required at launch. All six components are rule-based math or Claude API calls — none need historical data, which solves the cold-start problem outright. ML (XGBoost, then deep learning) only enters once enough labeled outcomes exist (Phase 3+, see [ROADMAP.md](ROADMAP.md)).

## The six components

| # | Component | Type | Build time (spec) |
|---|---|---|---|
| 1 | HCI Engine | Rule-based math | 2 weeks |
| 2 | Distribution Agent | Rule-based code | 1.5 weeks |
| 3 | Hiring Agent | Rule-based code | 1.5 weeks |
| 4 | Skill Test Agent | Claude API | 1 week |
| 5 | Feasibility Engine | Pure math (Haversine) | 3 days |
| 6 | Support AI | Claude API | 1 week |

They're meant to work as a pipeline: HCI feeds Distribution and Hiring; Distribution feeds Hiring; Feasibility gates acceptance; Support AI sits alongside, reading from the same account/job context.

### 1. HCI Engine — Human Capital Index

10-dimension weighted score, 0–1000 (full dimension table in [VISION.md](VISION.md)). Recalculated on a cadence (spec: every 7 days) with:
- **Confidence band** — blends signal completeness (40%) and job-history maturity (60%); communicates *how much* to trust the score, not just the score itself.
- **Trajectory vector** — 5-job rolling average compared to prior period → Improving / Declining / Stable.
- **Audit trail** — every point change logged with dimension + reason, surfaced to providers as improvement recommendations.

### 2. Distribution Agent — job → providers

```
Customer posts job → Geo query (PostGIS, initial radius)
  → Hard filter (busy, skill < 60%, blocked, no-show rate > 30%)
  → Score: skill match 35% + distance 25% + HCI 25% + fairness boost 15%
  → Push to top N providers (Firebase FCM)
  → Auto-expand if no response (Urgent: 10min · Normal: 30min · Slot: 2hr)
```
Fairness boost: new providers (<5 jobs) get +15% so a handful of top performers can't monopolize all job exposure. Every decision logged to `agent_decisions`.

### 3. Hiring Agent — applicant ranking

Customer sees applicants ranked, not in raw list order:

| Signal | Weight |
|---|---|
| Skill-test match for required skill | 30% |
| Average job rating | 25% |
| Job completion rate | 20% |
| No-show rate (penalty) | −15% |
| Customer rehire rate | 10% |
| Premium provider badge | +8pt fixed |

**Hiring preference modes**: customer can pick **Fast** (weight distance/availability), **Best** (weight skill/rating), or **Fair** (stronger new-provider boost). Includes risk flags (first job with this customer, recent cancellations, low trust score) shown alongside the ranking — not a black box.

### 4. Skill Test Agent — AI-generated assessments

```
Provider selects skill → check question bank
  → if empty: Claude generates N questions, stored permanently
  → provider answers → rule-based evaluation (no AI cost) → score
  → Skill-O-Meter updated → HCI Skill dimension recalculated
```
**Cost model**: one Claude API call *per skill*, not per test. Spec: ~50 skills × ~$ a few cents per call ≈ a few hundred dollars total, one time. After seeding, every test costs $0 in AI spend — evaluation is pure rule-based code. 7-day retake cooldown; score is a weighted average of best attempts (anti-gaming).

### 5. Feasibility Engine — scheduling conflict check

Haversine distance + travel-time estimate, checked before every job acceptance:

| Verdict | Meaning |
|---|---|
| SAFE | Gap > 15 min after travel |
| TIGHT | Gap 5–15 min |
| RISKY | Gap 0–5 min |
| NOT FEASIBLE | Overlap or impossible travel |

Spec explicitly notes this logic already existed client-side as `schedulingEngine.ts` in the provider app and just needs porting to a backend Python endpoint.

### 6. Support AI — customer support

Claude API agent with full account-context injection (active jobs, payment history, account status). Handles job-status queries, cancellation policy, payment disputes, login issues, "why wasn't my job shown to me" — without a human, until an escalation rule fires:

- Refund request above a $ threshold
- Safety/abuse complaint
- Three failed resolution attempts
- User explicitly asks for a human

## Data strategy: launching with zero data

Three layers solve cold start without needing any ML:

1. **Registration signals (Day 1)** — ID verification, profile photo, self-reported experience, address verification, first skill test pass → a fully-registered provider starts at ~380–420 HCI / ~28% confidence, labeled transparently as "New Verified Provider."
2. **Pre-seeded question bank (run once before launch)** — Claude generates the MCQ bank for every skill ahead of time; one-time cost, reused forever afterward.
3. **Beta cohort before public launch** — real providers, real jobs, real outcomes, used to validate that HCI rank actually correlates with customer satisfaction before opening to the public.

### `agent_decisions` — the ML seed table

Every agent decision (Distribution's provider list, Hiring's ranking, HCI's score) is logged with full `input_features` (JSONB), the `decision` made, and the eventual `outcome`/`outcome_rating`. This table is the training set for the Phase-3 XGBoost upgrade — it has to be populated from day one regardless of when ML actually starts, or there's nothing to train on later.

### Training maturity curve

| Phase | Job volume | What's running |
|---|---|---|
| 1 — Launch | 0–500 | Pure rules + Claude API, no training |
| 2 — Validation | 500–5,000 | Manual weight tuning against real outcomes, first A/B tests |
| 3 — First ML | 5,000–50,000 | XGBoost Hiring Agent trained on `agent_decisions`, replaces rule weights |
| 4 — Full pipeline | 50,000+ | Weekly retraining, dynamic pricing, churn prediction |

## Why this is meant to be hard to copy

The spec's argument: existing players (Urban Company-style, Sulekha/Justdial-style, Upwork/Fiverr-style) each have a structural reason they can't replicate this — manual/expensive skill verification, no scoring at all, or a single-dimension star average that's trivially gamed. SkoFy's compounding loop: more jobs → more behavioral signal → more accurate HCI → better Distribution/Hiring decisions → higher job success → more trust → more jobs. A competitor can copy the formulas; they can't copy the behavioral data that makes the formulas accurate, because that data only exists because providers and customers already trusted the platform.

For what of this is actually live in the codebase today, continue to [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md).
