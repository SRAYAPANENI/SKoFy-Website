# SkoFy — Roadmap

> Reconciles the source vision/AI-engine docs' phase plan with (a) the US-first launch decision and (b) what [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md) shows is actually built. The source docs' *structure* (phases, week-by-week build, data-maturity curve) is sound and kept; the *specifics* that assumed India are flagged for a swap.

## Immediate priorities (closing the gap to the original 7-week AI plan)

These are the AI Engine components spec'd as Day-1/Week-1-7 deliverables that aren't built yet, in the order they unblock each other:

1. **HCI Engine** — currently just a `hci_score` field with nothing computing it. Blocks: honest provider scoring, Hiring Agent (needs a real score input), the "Trusted Pro" / confidence-band UX promised in [VISION.md](VISION.md). Implement `hci_signals_builder.py`-equivalent reading from `jobs`/`reviews`/`provider_skills`, wire `recalculate_provider_hci` to actually compute instead of stub.
2. **Hiring Agent** — ✅ now ranks applicants (2026-08-06) by reusing Distribution Agent's 3-factor formula (skill-match + HCI + distance) instead of chronological order — see [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md). Still a gap vs. the original 8-signal spec (no completion-rate/no-show/cancellation weighting, no Fast/Best/Fair mode selector, no risk flags) and still leans on whatever `hci_score` currently holds — the HCI Engine itself computing that score for real is still the deeper unblock.
3. **Distribution Agent auto-expand** — `expand_stalled_distributions` stub needs to actually re-run `_find_providers` at a wider radius for jobs with zero responses past a threshold, not just rely on the one-shot fallback at post time.
4. **Feasibility Engine** — smallest lift in the original spec (3 days, pure math, no DB dependency) and currently fully absent. Worth doing early since it's cheap and unblocks safer double-booking prevention.
5. **Support AI** — lowest priority of the five; `ai.py`'s problem-triage endpoint already covers part of the "AI on every job" surface area. Build once the others are stable, since it needs full account-context plumbing (jobs + payments + disputes) — payments data now exists to plumb (router is mounted, see below), so this is no longer blocked on that, just not yet built.

## Platform features blocking real usage

From [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md):

- **~~Payments router not mounted~~ — resolved, this bullet was stale.** Verified 2026-09-12: the `payments` router is mounted and `PaymentService` is a full, production-shaped Stripe integration (escrow via separate charges+transfers, Connect Express payouts, webhook verification, refunds, both mobile apps wired client-side) — see [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md)'s Payments row. What's left is **config, not code**: real Stripe API keys in the deployed environment (currently empty placeholders) and registering the webhook URL with Stripe. Once that's done, real transactions can flow and `agent_decisions` data can start accumulating for the AI roadmap below.
- **Live tracking router not mounted** — lower priority than payments; the customer app already has a `track-provider.tsx` screen waiting for it.
- **Admin panel doesn't exist** — `resolve-dispute` endpoint has no UI. Per [[tech-debt-skofy]] memory, this is deliberately deferred until after KYC integration, but dispute resolution is flagged as the natural first screen once it's picked up.

## Market-specific items that need a US pass

The source vision doc's phase plan names India-specific vendors/targets that need US equivalents identified before they're actionable:

| Source doc says (India) | Needs a US equivalent |
|---|---|
| Razorpay | Already addressed — code uses Stripe |
| MSG91 (OTP SMS) | Already addressed — code uses Twilio |
| Hyderabad beta launch city | **Unresolved** — pick a US launch metro |
| NSDC / DigiLocker government partnership | **Unresolved** — no US equivalent identified yet (state workforce boards? Apprenticeship.gov? Needs research, not a guess) |
| ₹ pricing, ₹500 question-bank seeding cost, ₹400Cr competitor-switching-cost figure | Needs USD figures — orders of magnitude will differ, don't just convert at exchange rate since US labor-market economics differ from India's informal economy assumptions the whole thesis is built on |
| KYC vendor (implicitly Aadhaar/PAN-based in the India version) | Per [[skofy-market-usa]] memory: user will choose the vendor themselves (Persona/Onfido/Stripe Identity/Veriff are the US-market candidates) — don't build this unprompted |

This table is the single most important thing to resolve before treating the source vision doc's "Phase 1" beta-launch plan as executable — right now it's executable *in India*, and the project isn't launching there.

## Build sequencing (from the source AI Engine spec, structure preserved)

The original 7-week parallel build plan, useful as a *sequencing template* even though the actual repo is already past Week 1 for several items:

| Week | Planned | Actual status |
|---|---|---|
| 1 | Feasibility + HCI Engine core | Neither built |
| 2 | Distribution Agent | Core built, periphery (auto-expand, fairness) missing |
| 3 | Hiring Agent + Skill Test Agent | Skill Test built (exceeded spec); Hiring Agent not built |
| 4 | HCI live DB connection + audit trail | Not built |
| 5 | Support AI + agent-decision logging | Neither built (`agent_decisions` table exists per schema but check it's actually being written to before relying on it for ML later) |
| 6–7 | API exposure + mobile integration + perf testing | Chat/WS done; payments/tracking routers not mounted |

Reading this: the team did **not** follow the week-by-week order — Skill Test Agent (a Week 3 item) is the most complete piece of the whole AI engine, while Week 1 items (Feasibility, HCI) are untouched. That's fine (skill verification was probably the highest-leverage thing to ship first), but it means future planning should go off [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md), not off this week numbering.

## Longer-term phases (from VISION.md, for context — not imminent work)

| Phase | Scope | Trigger to start |
|---|---|---|
| 2 | Universal HCI foundation, +Blue-Collar & Education verticals, portable score link, first B2B API customers | After Phase 1 (home services) HCI is actually validated against real outcomes — can't abstract a vertical-adapter system from an engine that doesn't exist yet |
| 3 | Medical + Research verticals (regulated — license/HIPAA-equivalent verification), first XGBoost ML model | Needs 5,000+ labeled `agent_decisions` rows — currently zero, since payments aren't live and the table isn't being written to |
| 4 | Blockchain portable credential, HCI as loan-rate collateral signal, 10+ verticals, government partnership | Multi-year horizon; the government-partnership leg specifically needs the US-equivalent body identified first |

## Open strategy questions (pending partner discussion)

Raised in a working session, not yet decided or built — captured here so the reasoning survives until revisited. Each has a recommendation + the trade-off that makes it not a slam-dunk. Update this section (don't just delete it) once a decision is made — move the resolved item to the relevant section above and note what was decided and why.

### 1. Multi-skill providers — what single "profession" to show

A provider can legitimately span several professions (AC repair + electrician + software dev) — `Skill.profession` (`skofy-backend/app/models/skill.py:19`) and `ProviderSkill` (many-to-many) already allow this at the data layer.

- **Recommendation**: don't store a headline profession. Derive it dynamically — whichever profession has the most completed jobs (or highest HCI-weighted skill score) for that provider — recomputed as their history grows, never frozen at signup, never freeform text (breaks search/matching, e.g. "AC guy" vs "HVAC Technician").
- **Trade-off**: a single derived headline is right for compact UI (map pin, list card) but loses the multi-skill picture — pair it with a skill-chips row on the full profile so the compact view shows one label while the detail view shows everything.

### 2. Inspection fee — wasted-trip / provider retention risk

**Update 2026-10-21 — designed in [MONEY_RULES_DESIGN.md](MONEY_RULES_DESIGN.md):** who gets the visiting fee when either side cancels (by stage: before set-off, after set-off, after arrival, customer not there), proof that an inspection happened (the customer confirms at the end, with evidence checks as the fallback), and a $15 minimum visiting fee. Built 2026-10-21.

**Update 2026-09-12**: this is no longer blocked on payments — `Job.inspection_fee` (`job.py:52`) is now actually charged and escrowed via `PaymentService.create_inspection_fee_intent`/`release_inspection_fee` (see [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md)'s Payments row). The question below is about the *policy* (refundable vs. not), which the code currently implements as non-refundable-if-not-hired/credited-if-hired — i.e. the recommendation below is already what's built, not still open.

**Update 2026-09-25 — who sets the amount changed**: the customer no longer picks anything at posting time (the distance-tier picker that used to drive both `search_radius_km` and this fee was removed entirely — it was redundant with the Distribution Agent's own default 5km→25km expansion, and forcing a distance/fee decision on the customer was pure friction with no upside). Each **provider** now sets their own "Visiting Fee" (+ Hourly Rate) on their profile — self-typed or from a real market-average suggestion (`GET /providers/me/rates/suggest`, not an LLM call) — and `Job.inspection_fee` gets snapshotted from whichever provider ends up hired, at hire time (`JobService._finalize_hire`). The upfront/non-refundable/credited-toward-`final_amount` policy below is unchanged; only the source of the number changed, from a customer-chosen distance tier to the hired provider's own rate. UI copy across both apps now says "Visiting Fee," not "Inspection Fee" — the DB column/payment `purpose` string are unchanged internal names, not user-facing. Also removed in the same pass: the 20-minute minimum on-site inspection duration (no longer enforced, backend or UI) — a provider can mark inspection complete immediately after starting it.

- **Recommendation**: charge it upfront, non-refundable if the customer doesn't hire after the visit (the provider's cost was the trip, not the outcome). Credit it toward `final_amount` if the customer does hire that provider, so it only feels "wasted" when the customer was never serious.
- **Trade-off**: charging upfront for an intangible (a visit) adds booking friction for price-sensitive/first-time customers. The retention risk (provider eats a bad visit, stops trusting the platform) is arguably better solved by the platform absorbing some of it — e.g. a small guaranteed minimum payout per qualifying visit, funded from the existing ~12% take rate — rather than passing 100% of the risk to either side.

### 3. Overcharging / price-gouging fact-check (quoting 5000 for a 1000 job)

`Job.budget_min`/`budget_max`, `JobApplication.proposed_fee`, and `CustomerReview.negotiation_rating` all already exist in the schema but aren't wired together.

- **Recommendation**: can't cap or reject a quote — that would break the C2C/anti-monopoly thesis in [VISION.md](VISION.md) (the platform doesn't set prices, full stop). Instead, flag `proposed_fee` as a statistical outlier against the job's own budget range and the median quote from other applicants for that skill/area ("60% above typical for this skill nearby"), and feed `negotiation_rating` into that provider's HCI Trust dimension over time so serial overchargers get a visible reputation hit without SkoFy ever dictating price.
- **Investor framing**: the honest answer to "how do you stop price gouging" is "we don't — we make it visible and let reputation price it in," not "we prevent overcharging." That's a deliberate philosophical stance, not a gap to paper over.

### 4. Competitive positioning vs. GigNGo, TaskRabbit, DoorDash

GigNGo (commission-free, flat worker subscription, "GigFeed" short-video discovery, Google-indexed profiles, live in all 50 states as of mid-2026) is the closest active US competitor; TaskRabbit (IKEA-owned, % fee from customer, star ratings only) and DoorDash (pure algorithmic dispatch, opaque pay, no "hire this specific person" concept) are the other reference points.

| | GigNGo | TaskRabbit | DoorDash | SkoFy (thesis) |
|---|---|---|---|---|
| Model | C2C-ish, flat worker subscription | B2C-flavored, % fee from customer | Pure dispatch/logistics | C2C, flat ~12% regardless of who wins |
| Reputation | Short-video "GigFeed" + reviews — a discovery/branding layer | Star ratings only, no verification | Almost none | HCI: skill-tested + behavioral, portable |
| Skill verification | None found | None | None | Claude-generated skill tests per trade |
| Vertical scope | Home services only | Home services + errands | Food delivery only | Designed to reach medical/research |
| Pay-to-rank | No (subscription-based) | No | N/A | Structurally can't (no employment relationship to leverage) |

None of the three competitors has a portable, verified trust score — GigNGo solved *discovery* (video feed, SEO), which is a different axis from SkoFy's bet on *trust verification*. But the honest gap: GigNGo is live with real users in all 50 states; per [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md), SkoFy's HCI Engine, Hiring Agent, and Feasibility Engine are still mostly stubs, and nothing in SkoFy solves GigNGo's actual innovation (discoverability) — a differentiated trust layer means nothing if no one can find a new provider yet. That's the problem item 5 below addresses.

### 5. New-provider discoverability (the actual competitive risk)

**Status update 2026-09-12**: this section was stale — items 1 and 4 were already built, and items 2 and 3 have now been built too. Only the underlying positioning question (should SkoFy *lead* with discoverability as a differentiator, and is a map-based directory + badges enough vs. GigNGo's video/SEO approach) remains for partner discussion; the four originally-listed technical levers are done:

1. **✅ Provider directory, reachable without posting a job.** `skofy-customer-app/app/provider-map.tsx` was already a real browse surface (not just job-triggered distribution) — reachable directly from the home screen's "Providers Near You" card, with a search bar and profession quick-filter chips. No further build needed here; this bullet in a prior version of this doc was inaccurate.
2. **✅ Fairness boost, fully implemented 2026-09-12.** Built in two passes after reverification caught a gap in the first one. Pass 1: `DistributionAgent.score_candidate`/`score_candidate_remote` (`app/services/distribution_agent.py`) add a flat `DISTRIBUTION_FAIRNESS_BOOST_POINTS` (15, config in `app/core/config.py`) bump for providers with fewer than `DISTRIBUTION_FAIRNESS_BOOST_MAX_JOBS` (5) completed jobs — fixes Hiring Agent's applicant ranking (`JobService._applicant_score`, called after providers have already applied). Pass 2, after reverification found pass 1 didn't help *selection*: `_find_providers`/`_find_providers_remote` pick which providers get notified about a job via their own `ORDER BY`/`LIMIT` (distance for on-site, raw `hci_score DESC` for remote) — before any scoring/boost runs — so a brand-new provider could still never be notified in the first place if enough established candidates ranked above them. Fixed by `_find_fairness_topup_onsite`/`_find_fairness_topup_remote`: a second, small query that reserves `DISTRIBUTION_FAIRNESS_GUARANTEED_SLOTS` (2) *additional* notification slots for under-threshold providers, additive on top of the normal pool (never displaces an already-selected provider — chosen deliberately over reweighting the main selection query itself, which would trade match quality for reach).
3. **✅ "New Verified Provider" badge, implemented 2026-09-12 (with a correctness fix the same day).** The real HCI Engine (`app/services/hci_service.py` — see corrected [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md) HCI Engine row) already computes a genuine `hci_confidence` per provider; that field is now exposed via `/providers/nearby` and `/providers/{id}` and surfaced as a badge (customer app: `provider-map.tsx`'s detail sheet and `provider-details.tsx`'s header) whenever confidence is below a threshold (40). **Reverification found the badge (both the new version and the pre-existing static "Verified" label it replaced) had no connection to the real `is_identity_verified` field** — it could claim "Verified" for a provider who never actually passed ID verification. Fixed same day: `is_identity_verified` is now also exposed via both endpoints, and the badge (either variant) only renders when it's true; an unverified provider gets no badge at all rather than a false one.
4. **✅ Work-proof portfolio gallery.** Already built — `provider-details.tsx`'s "Work Proof" tab renders `ProviderDocument` (WORK_PROOF/SKILL_PROOF) images/videos in a gallery with a full-screen media viewer. This bullet in a prior version of this doc was also inaccurate.

**Decided 2026-10-07 — build before launch**: public, Google-indexable provider profile pages (skills, trust score, reviews, "Book on Dodorez") plus category/city pages, matching GigNGo's discovery approach. Not built yet; the investor brief (`dodorez_competitor_analysis.pdf`) already presents it as part of the launch product, along with Stripe Connect payouts and iOS VoIP ringing — all three must be done before launch.

### 6. Off-platform leakage / disintermediation (raised by Mouli, 2026-07-19)

**Update 2026-10-21:** the "loyalty/tiered commission" and "reduced commission for repeat volume" ideas below are now one designed model, Fair Cap ([MONEY_RULES_DESIGN.md](MONEY_RULES_DESIGN.md) §1): a banded fee, at most $119 a job and $249 a month, then 2% (option B, 2026-10-22). Built 2026-10-21; the numbers still need provider validation.

**Update 2026-10-22, built ([MONEY_RULES_DESIGN.md](MONEY_RULES_DESIGN.md) §10):** chat screening hides phone numbers and emails and warns on talk of paying another way; every customer rating asks whether the provider asked them to pay outside the app; support decides each case as a strike or no action. The "did they solicit off-app?" survey and the cheapest playbook items are now done. Still open: the provider security deposit and risk scoring. **Update 2026-10-22:** the platform warranty (the Dodorez guarantee) and repeat bookings are built ([MONEY_RULES_DESIGN.md](MONEY_RULES_DESIGN.md) §12–13).

Standard marketplace risk — customer and provider meet through the app, then transact directly outside it on repeat jobs, cutting SkoFy out of its take rate. Mouli proposed a 12-point mitigation playbook (staged inspection→quote→approval workflow, escrow, GPS-confirmed job start, platform warranty, loyalty/tiered commission, provider security deposit, AI risk-scoring on job events, post-inspection "did they solicit off-app?" survey, QR-code job start, 24–48h automated follow-up, reduced commission for repeat volume). None of it is naive — this is close to how Thumbtack/Handy/Upwork actually operate.

Reconciled against actual code:
- **Already built**: GPS+timestamp job-start confirmation (`navigate-to-site.tsx` → `markStarted(lat, lng)` → backend `NOT_AT_LOCATION` check) — functionally the same guarantee as Mouli's QR-code proposal, so skip QR, no need for two mechanisms doing the same job. **Also now built (2026-09-12 correction)**: escrow (inspection fee + job cost, both held/released/refunded via Stripe — see Payments row in [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md)) and the staged inspection→invoice→payment workflow (`ACCEPTED` → inspection → `INVOICE_PENDING` → `JOB_COST` escrow → completion/release). This closes off two items Mouli's playbook flagged as unbuilt.
- **Schema exists, no logic behind it**: `Review`/`CustomerReview` rating fields (collected, but HCI Engine that would turn them into a trust/reputation score is an unbuilt stub — this is already [ROADMAP.md](ROADMAP.md)'s #1 immediate priority for unrelated reasons).
- **Not built, no longer payments-blocked**: platform warranty, loyalty/tiered commission, provider security deposit, and most of the AI risk-scoring signals — these were waiting on payments infrastructure existing, which it now does; what's missing now is the business-model numbers (see below), not a technical blocker.
- **Not built, but no payments dependency** — the two cheapest possible wins if early signal is wanted before payments ship: the post-inspection "did the provider offer to continue outside the app?" one-question survey, and the 24–48h automated follow-up check-in. Both are just a notification + a DB flag.
- **Business-model decisions, not coding decisions** — need partner sign-off on actual numbers before building: security deposit amount/terms, warranty coverage terms, loyalty-tier commission thresholds, risk-score point values/threshold.

**Recommendation**: payments being live (see 2026-09-12 correction above) unblocks most of what was queued behind it — warranty/loyalty/deposit/risk-scoring now just need business-model numbers, not code. Worth noting: the direct-booking feature (customer picks one exact provider off the map and sends them a request) plausibly *raises* leakage risk relative to the old broadcast-only flow, since customer and provider identify each other before any quote or payment exists, on the very first job — the eventual risk-scoring system should weight direct-request jobs differently than broadcast ones.

Not yet decided or built; revisit once discussed with Mouli.

### 7. Expansion beyond home services — wedge-vs-breadth, and which quadrant first (raised 2026-08-07)

SkoFy's stated vision is "any skill, any need," but everything actually built (Distribution Agent's geo-radius matching, Skill Test Agent seeded per trade) is shaped around one narrow case: a physically-present, skill-verified home-service job. The customer app's home screen already leans broad (Explore Services lists Software Dev/Teacher next to Plumber/Electrician) while the backend hasn't caught up — that mismatch is the concrete symptom of an undecided strategic question, not a UI bug.

**The core unresolved call**: launch narrow (home services only, prove liquidity/trust there, expand later — how Uber/Airbnb/Amazon actually grew) vs. stay broad (keep building "any skill" now, accept a less-optimized first experience). Not resolved in this session.

A working 2×2 emerged for reasoning about *which* expansions are even the same kind of problem:

| | Physical presence needed | Remote/no presence needed |
|---|---|---|
| **Skill-verified needed** | Home services (built) | Software dev, consultancy, tutoring — **not built, and architecturally different**: no geo-radius term at all, needs availability+skill-match instead of distance+skill+HCI |
| **No skill verification** | Generic errands, pickup/dropoff tasks | Trivial remote tasks |

Three concrete candidate expansions came out of this — **all three are now built** (2026-08-07), ahead of the wedge-vs-breadth call being resolved (the user chose to build the technical capability for all three rather than sequence them):

1. **✅ Urgent Needs quick-action** — a slim banner in the customer app's home-screen hero card (not a new Explore Services tile row, per the original recommendation), pre-selecting the existing `urgency` field's "Urgent" chip. No backend change needed — `urgency` was already fully wired.
2. **✅ Pickup/dropoff tasks** — `Job.job_type` (`STANDARD`/`PICKUP_DROPOFF`) + a second `dropoff_location` point (migration `027`). Pickup-location verification is enforced **server-side** (`CreateJobRequest` validator against `settings.PICKUP_ALLOWED_PLACE_TYPES`, an allowlist of Google Places business types) — client-side Places `includedPrimaryTypes` filtering is UX only, never trusted alone. Both apps show a two-point map/summary. **Known v1 gap, flagged deliberately**: no separate "arrived at dropoff" geofence — OTP inspection only verifies the pickup point.
3. **✅ Remote knowledge-work jobs** — `Job.service_mode` (`ON_SITE`/`REMOTE`, migration `028`). A genuinely different matching path, not an extension of Distribution Agent: `DistributionAgent.distribute_remote()` + `score_candidate_remote()` (HCI+skill-match only, no distance term) — see [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md)'s Hiring Agent row for how this also changed applicant ranking. On-site inspection/OTP is skipped entirely for `REMOTE` jobs — `raise_invoice()` is reachable directly from `ACCEPTED`. **Deliberate scope decision needing confirmation, not an oversight**: this means no inspection-fee escrow step for remote jobs in v1 — a real revenue question, not just a technical one. Content-side, this still needs remote-eligible professions ("Software Developer", "Consultant") actually seeded into the skill catalog — the matching engine works, but the catalog wasn't extended as part of this pass.

A real prerequisite bug was found and fixed along the way: the provider app defaulted a job's missing coordinates to `(0, 0)` instead of `null`, which would have shown bogus "7,432 mi away" distances the moment any location-less job existed — fixed before the REMOTE job type could expose it.

**Explicitly out of scope, decided**: person transport (Ola/Uber-style ride-hailing). Raised and rejected in the same discussion — moving *people* is a categorically different, heavily-regulated business (TNC licensing, commercial insurance, per-km fare mechanics) from task/skill matching, not a bolt-on feature. Revisit only as a deliberate, separate decision, not as a natural extension of pickup/dropoff tasks.

**Decided 2026-10-07 — breadth at launch**: the founder chose to launch with a mix of categories (in-person skilled work, errands/urgent help, and online work), not home services first. Pitch positioning: "hire skilled people you can trust, in person or online" (see `dodorez_competitor_analysis.pdf`). Accepted trade-off: early liquidity is spread across more categories; mitigated by multi-skill providers and online supply not being tied to the launch city.

**Still open — needed before launch to make the breadth real**: seeding online professions/skills (software development, web/graphic design, online tutoring…) into the skills catalogue; widening the voice assistant's prompts in `app/api/v1/ai.py`, which still describe Dodorez as a "home services platform" and refuse other requests; and the inspection-fee-for-remote-jobs revenue question flagged above.

## Data flywheel dependency chain

This is the order things have to happen in, regardless of calendar time, because each step's data feeds the next:

```
Payments live → jobs actually complete with $ attached
  → Reviews + completion data accumulate
  → HCI Engine has real signals to compute from
  → Hiring Agent has a real HCI input to weight
  → agent_decisions accumulates labeled outcomes
  → (5,000+ rows) → XGBoost can replace rule-based Hiring weights
```
**Update 2026-09-12**: "payments live" was the top blocker in this chain for code-readiness reasons; that code is now done (see IMPLEMENTATION_STATUS.md's Payments row). The chain is now blocked only on **real Stripe keys + a live deployment** actually processing real jobs — an ops/launch step, not an engineering one. Nothing downstream (HCI real signals, agent_decisions, eventual ML) can start accumulating data until real transactions are flowing, so this is still the top priority to close, just a different kind of task than it used to be.
