# SkoFy — Vision & Strategy

> Condensed from `skofy_definitive_vision.html` (v3.0, June 2026). That file has the full prose, diagrams, and tables — this doc is the reference version for quick recall.

## ⚠️ Market mismatch to resolve

The source vision document is written entirely around an **India** launch: ₹ pricing, Hyderabad as launch city, Razorpay for payments, MSG91 for OTP SMS, NSDC/DigiLocker as the government-partnership target.

Actual project direction (per [[skofy-market-usa]] memory, confirmed by code): **SkoFy launches in the USA first.** The backend already reflects this — `Provider.state` is a 2-letter US state code, config has `STRIPE_SECRET_KEY`/`TWILIO_*` placeholders, not Razorpay/MSG91.

**Read everything below as strategic intent, not literal go-to-market detail.** The *model* (C2C, Universal HCI, anti-monopoly, multi-vertical) is market-agnostic and still the thesis. The *specifics* (currency, city, payment/SMS vendor, government body) need a US pass before they're operational truth. See [ROADMAP.md](ROADMAP.md) for the reconciled version.

## The core thesis: C2C, not B2C

Every competitor (Urban Company, TaskRabbit-style models) runs a B2C employment-agency model: the platform employs or formally rosters providers, controls supply, and owns the customer relationship. **SkoFy is consumer-to-consumer** — any person can be an employer, any person can be a provider, and the platform is purely the trust layer connecting them.

| | Traditional (B2C) platform | SkoFy (C2C) |
|---|---|---|
| Who can provide | Only company-rostered workers | Individual, freelancer, small firm, large firm — all visible |
| Who can hire | Customer picks from pre-approved list | Anyone — individual, business, government |
| Pricing | Platform dictates rates | Market sets rates; platform takes a flat %% |
| Provider reputation | Locked to the platform | Portable — HCI follows the provider everywhere |
| Vertical scope | One vertical (rebuild required to expand) | Same engine, any vertical |

**Why this matters structurally**: a platform that owns its supply becomes a chokepoint that extracts maximum rent from both sides. SkoFy can't do that — it doesn't own providers, so it only earns by verifying trust, not by controlling who's allowed to compete.

## The Human Capital Index (HCI) — one score, every field

A 0–1000 behavioral trust score, **field-agnostic by design** — the same 10 dimensions apply to a plumber, a nurse, or a researcher; only the underlying signals and skill-test content change per vertical.

| Dimension | Weight | What it measures |
|---|---|---|
| Trust | 20% | Dispute rate, no-shows, ID verification, complaint history |
| Skill | 20% | Skill-test scores, verified skill breadth |
| Reliability | 15–17% | Completion rate, cancellations, on-time arrival |
| Efficiency | 10–12% | Jobs/week, earnings/job, response time |
| Emotional IQ | 10% | Customer behavior ratings, communication quality |
| Adaptability | 8% | New skills per month, cross-vertical capability |
| Network Capital | 7% | Repeat clients, rehire rate |
| Learning Velocity | 5% | Test-improvement speed, time to first pass |
| Well-being | 0–3% | Overwork detection (deferred — needs data volume) |
| Civic Signal | 1–2% | Referral quality, community contribution |

**Confidence bands** are the honesty mechanism: a Day-1 provider scores ~380–420 at ~28% confidence and is labeled "New Verified Provider," not hidden and not falsely presented as proven. Confidence rises with signal volume (registration → ID verified → skill test passed → N jobs completed).

**Trajectory vector**: recalculated periodically, compares current vs. previous score to label a provider Improving / Declining / Stable — customers see direction, not just a snapshot.

## Anti-monopoly architecture

The platform is structurally prevented from becoming a kingmaker:

- **Any provider can register** — solo operators, small firms, and large companies rank in the *same* pool, purely by HCI score, price, and availability. No pay-to-appear.
- **Market sets prices** — SkoFy takes a flat ~12% (`PLATFORM_FEE_PERCENTAGE` in code) regardless of which provider wins, so it has no incentive to favor anyone.
- **HCI is portable** — a provider's score is theirs, not the platform's leverage over them.
- **Same engine, any vertical** — expanding to a new field costs a config file + a skill-test seed run, not a rebuild.

## Multi-vertical expansion path

The HCI engine is intentionally abstract so it scales horizontally without rebuilding:

| Vertical | Phase | Examples |
|---|---|---|
| Home Services | 1 (launch) | Plumbing, electrical, HVAC, carpentry, cleaning |
| Blue-Collar Trades | 1–2 | Welders, machinists, drivers, construction |
| Medical & Healthcare | 2 | Home nurses, caregivers, physiotherapists — license/HIPAA-grade verification |
| Research Fields | 2–3 | Data analysts, field surveyors — institutional/ethics verification |
| Education | 3 | Tutors, coaches, trainers |
| Anything else | ∞ | Legal, financial, creative, agriculture — same infra, new question bank |

The reason incumbents can't follow: their model requires employing or formally training providers, which is a regulatory/liability problem in medical and research fields. SkoFy's C2C model sidesteps that — the platform verifies, the employer decides, SkoFy never employs anyone.

## Why the moat is structural, not feature-based

Algorithms can be copied. The model can't be, without an incumbent destroying their existing business:

1. **Model incompatibility** — switching to C2C means firing employed providers and writing off training infrastructure.
2. **Data lock-in via cross-vertical history** — once providers have portable, behaviorally-verified HCI histories across verticals, a single-vertical competitor can't replicate the network.
3. **Regulatory/government partnership** — being the preferred private-sector behavioral-credential layer for a national digital-infrastructure program (the source doc names India's NSDC/DigiLocker; the US equivalent has not been identified yet — flag for [ROADMAP.md](ROADMAP.md)).

## One-paragraph statement (source v3.0)

> SkoFy is an open C2C trust economy — the infrastructure layer that lets any person prove their professional worth, and any person or organization find verified skilled talent, across every field where trust and skill matter. Starting with home services, expanding through blue-collar trades, medical, research, and beyond — SkoFy's Universal Human Capital Index creates a portable, behavioral, AI-verified professional identity for every skilled worker. No company owns the providers. No platform controls the price. No monopoly dictates who gets seen.

## Security model (10 layers, as designed)

| Layer | Protects against | How |
|---|---|---|
| JWT auth | Unauthenticated access | 15-min access + 30-day refresh, role in payload |
| bcrypt cost 12 | Password cracking | DB breach alone doesn't expose passwords |
| OTP rate limit | SMS spam / SIM swap | 3 sends/hour per number (Redis) — `OTP_MAX_SENDS_PER_HOUR` in code |
| RBAC on all routes | Privilege escalation | Customer routes reject provider tokens and vice versa |
| Ownership checks | IDOR | Every query scoped to `user_id = current_user.id` |
| Pydantic v2 validation | Malicious/invalid input | Type-checked at every API boundary |
| SQLAlchemy ORM | SQL injection | No raw string-interpolated SQL |
| File MIME check (designed) | Malicious upload | Should read MIME from bytes, not extension — **not yet implemented**, see [TECH_DEBT.md](TECH_DEBT.md) #2 |
| Payment webhook signature | Fake payment callbacks | HMAC signature verification on every webhook — **payments router not mounted yet** |
| WebSocket JWT auth | Unauthorized realtime access | Token required at connection time |

## Investor framing (source doc's pitch, for context)

- **Market**: home services alone is large; blue-collar + medical + research multiplies it 5–10x.
- **Defensibility**: structural (C2C + Universal HCI + government partnership), not feature-based.
- **Category**: no direct global comparable — closer to "LinkedIn for informal workers + Airbnb-style trust + government digital-infrastructure rails" than to a services marketplace.
- **Scalability**: new vertical ≈ one config file + one skill-test seed run, not a platform rebuild.

For the technical roadmap reconciling this with an actual US launch and the current code state, see [ROADMAP.md](ROADMAP.md). For what the AI side of this vision (HCI engine, Distribution/Hiring agents, etc.) was actually specced to do, see [AI_ENGINE_SPEC.md](AI_ENGINE_SPEC.md).
