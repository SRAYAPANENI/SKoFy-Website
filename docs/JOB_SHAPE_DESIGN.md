# Job Shapes — supporting any situation without naming it

**Status:** phases 1–3 built 2026-10-09 (see §8) · **Owner:** founder · **Implements:** ROADMAP item 7 (breadth at launch)

## 1. Problem

Dodorez launches with a mix of categories (in person, errands, online). New situations keep appearing — a pet staying at the provider's home, house-sitting, a car left at a garage, weekly tutoring — and there will be many nobody has thought of yet.

Today the app knows three hard-coded situations:

| Today | Field | Assumption baked in |
|---|---|---|
| Work at the customer's address | `job_type = STANDARD`, `service_mode = ON_SITE` | The provider travels to the customer |
| Pickup and drop-off | `job_type = PICKUP_DROPOFF` | Something moves A → B |
| Online | `service_mode = REMOTE` | Nobody travels |

Each one is handled by `if job_type == ...` checks spread through `job_service.py` and `job_repo.py` (33 occurrences). Adding a situation means adding more of them, and the AI assistant refuses anything outside the skills catalogue ("Dodorez is a home services platform…").

**Goal:** a new situation should need *no new code* in the common case — only data (a skill, a rate) — while still getting the right matching, pricing and protections.

## 2. Core idea: describe the *shape* of a job, not its name

Every job is described by a small, fixed set of answers. The AI fills them in from the customer's own words; the server validates them; everything else (matching, pricing, protections, app screens) is derived from them.

```
JobShape
  venue         CUSTOMER_PLACE | PROVIDER_PLACE | ROUTE | ONLINE
  timing        ONCE | WINDOW | STAY | RECURRING
  custody       NONE | ITEM | LIVING_BEING | HOME
  pricing_unit  VISIT | HOUR | DAY | NIGHT | ITEM | FIXED | QUOTE
  starts_at     datetime | null          (null = as soon as possible)
  ends_at       datetime | null          (required for WINDOW / STAY)
  recurrence    rule | null              (required for RECURRING, e.g. weekly on Mon/Thu)
  requirements  list[str]                free-form needs: "fenced yard", "speaks Spanish", "has a van"
  details       dict[str, str]           situation-specific answers the AI collected
```

*Who travels* is not stored — it is derived from `venue` (one source of truth):

| venue | Who travels |
|---|---|
| CUSTOMER_PLACE | Provider |
| PROVIDER_PLACE | Customer |
| ROUTE | Provider, carrying something |
| ONLINE | Nobody |

### Examples (none of these need a named feature)

| Customer says | venue | timing | custody | pricing |
|---|---|---|---|---|
| "My sink is leaking" | CUSTOMER_PLACE | ONCE | NONE | QUOTE |
| "Keep my dog at your place Fri–Mon" | PROVIDER_PLACE | STAY | LIVING_BEING | NIGHT |
| "Watch my house while I travel" | CUSTOMER_PLACE | STAY | HOME | DAY |
| "My car makes a noise" | PROVIDER_PLACE | ONCE | ITEM | QUOTE |
| "Walk my dog every weekday" | CUSTOMER_PLACE | RECURRING | LIVING_BEING | VISIT |
| "Pick up my parcel and bring it home" | ROUTE | ONCE | ITEM | FIXED |
| "Tutor my son weekly, online" | ONLINE | RECURRING | NONE | HOUR |
| "Fix my website" | ONLINE | ONCE | NONE | FIXED |

### Mapping from today's fields (no data loss)

| job_type | service_mode | → venue |
|---|---|---|
| STANDARD | ON_SITE | CUSTOMER_PLACE |
| PICKUP_DROPOFF | ON_SITE | ROUTE |
| any | REMOTE | ONLINE |

Existing jobs backfill to `timing = ONCE`, `custody = NONE`, `pricing_unit = QUOTE` — exactly today's behaviour.

## 3. Protections come from the shape, not the category

A single **policy** turns a shape into the ordered steps a job goes through and the trust bar a provider must meet. This is the part that makes unforeseen situations safe: a job nobody planned for still gets the right protections because its shape is known.

```
policy_for(shape) -> JobProtocol
  steps:          ordered list of Step
  provider_bar:   minimum checks the provider must have passed
  release_rule:   when escrowed money is released
```

| Shape condition | Steps / rules added |
|---|---|
| venue = CUSTOMER_PLACE | `TRACK_EN_ROUTE` → `ARRIVAL_CODE` (customer's code + provider GPS at the job address) |
| venue = PROVIDER_PLACE | `CHECK_IN_CODE` (provider's code + customer GPS at the provider's address) — tracking is not needed, the customer travels |
| venue = ROUTE | `PICKUP_PROOF` (geofenced photo, as today) → `TRACK_EN_ROUTE`; when the item is in custody, `HANDBACK_CODE` at drop-off (the pickup proof already serves as the hand-over) |
| venue = ONLINE | Nothing before the price — quoted from the description, as today. A deliver → customer-approves step is a **pending decision** (§9) |
| pricing_unit = QUOTE | `INSPECT` (paid inspection, only when there's something physical to see) → `QUOTE` (invoice the customer accepts and pays into escrow) |
| any other pricing unit | `PAY_UPFRONT` — the price is known at booking (per night, fixed…) and paid into escrow |
| custody ≠ NONE | `HANDOVER_PHOTOS` at hand-over, `HANDBACK_CODE` at return |
| custody ∈ {LIVING_BEING, HOME} | `PERIODIC_UPDATES` (photo/check-in at a set interval); provider_bar requires identity **and** background check |
| timing = STAY / WINDOW | release_rule = after hand-back / end of window |
| timing = ONCE | release_rule = on completion (as today) |
| always | `COMPLETE` → `RATE_BOTH_WAYS` |

The policy is a **pure function** (no database, no I/O): easy to test exhaustively and to reason about.

## 4. Architecture (SOLID)

### 4.1 Components

```
app/domain/job_shape.py        JobShape value object + enums + validation   (no dependencies)
app/domain/job_policy.py       policy_for(shape) -> JobProtocol              (pure)
app/services/job_steps/        StepHandler per step (handlers.py), JobFlow (flow.py),
                               the shared location check (geofence.py)
app/services/matching.py       MatchingStrategy per venue
app/services/job_capability.py JobCapabilities: is a shape runnable today (all steps have handlers
                               AND its venue has matching AND its timing is built — repeating jobs
                               are refused until series exist) — the one answer job creation and
                               the voice assistant both use
app/services/job_intake.py     checks the voice assistant's job summary before posting (pure)
app/repositories/job_step_event_repo.py   job_step_events audit trail
app/services/pricing.py        estimate(shape, rate_card) -> PriceEstimate  (pure, phase 4)
```

### 4.2 Principles applied

- **Single responsibility.** `JobShape` only describes; `job_policy` only decides steps; each `StepHandler` only performs one step (e.g. verifying an arrival code); `JobService` stops branching on job types and delegates.
- **Open/closed.** A new kind of protection is a new `StepHandler` plus one rule in the policy; a new way of matching is a new `MatchingStrategy`. Existing handlers and `JobService` don't change.
- **Liskov.** Every `MatchingStrategy` returns the same result type (ranked candidate providers) and every `StepHandler` honours the same contract (`can_perform`, `perform`), so callers never special-case an implementation.
- **Interface segregation.** Step handlers get only what they need (a narrow `JobStepContext`: job id, actor, location, payload) — not the whole `JobService`.
- **Dependency inversion.** `JobService` depends on the `MatchingStrategy` / `StepHandler` abstractions, chosen by a registry keyed on venue / step type; concrete classes depend on repositories through constructor injection (as the codebase already does).

### 4.3 Matching strategies (by venue)

| Strategy | Venue | How |
|---|---|---|
| `NearCustomerStrategy` | CUSTOMER_PLACE | Today's `DistributionAgent.distribute` (radius widening 5 → 25 km → city) |
| `NearProviderBaseStrategy` | PROVIDER_PLACE | **New.** Providers whose *base address* is within reach of the customer, who accept hosting, with capacity free across the dates |
| `RouteStrategy` | ROUTE | Today's pickup/drop-off distribution |
| `OnlineStrategy` | ONLINE | Today's `distribute_remote` |

All strategies also filter on `requirements` (matched against provider profile tags) and on the policy's `provider_bar`.

### 4.4 Pricing

`pricing_unit` × quantity derived from timing (nights between `starts_at` and `ends_at`, hours in a window, occurrences of a recurrence) × the provider's rate for that unit. `QUOTE` keeps today's inspection-fee → invoice flow. Providers get a small **rate card** (per visit / hour / day / night / item) instead of only `visiting_fee` and `hourly_rate`.

**Long stays and day prices** (built 2026-10-28, `best_price` in `app/domain/rate_card.py`):
- The rate card can also carry a **per week** and a **per month** price for each skill.
- A job is never priced per week. The customer picks dates, and each provider's total is the cheapest their prices allow:
  - **Stays:** 7 nights or days can be one week and 30 can be one month, mixed with the rest ("1 week × $250 + 3 nights × $40"). A stay rounds up to a week when that's cheaper.
  - **A few hours** can use the day price when it's cheaper (a day of daycare).
- **What applicants show:** each applicant shows the breakdown under their total.
- **Stay length:** stays of items can run 365 days; pets and homes stay at 90.

### 4.4a What's left in care, and what a host takes in

Built 2026-10-28 (`app/domain/care_sheet.py`, migration 087).

- **Care sheet** (`jobs.care_sheet`), asked when the customer picks "A pet" or "An item":
  - **Pets:** for each pet (up to 6) the kind, name, breed, size (dogs), age, vaccinated, spayed/neutered, and how they are with dogs, cats and children; food (who brings it — the owner, or the host — what and when); medicines; walks; notes; the vet; an emergency contact. Required: kind, name, a dog's size, the vaccination answer, who brings the food.
  - **Items:** what, how many (up to 50), the size (bag / box / car trunk / van), the value, whether it's fragile.
  - Optional on the server: the AI assistant and older apps send none, and providers see the assistant's free-form notes instead.
  - Applicants see everything except the vet and the emergency contact (`care_sheet_for`).
- **Host preferences** (`providers.hosting_prefs`):
  - **Customers can come** at all, with a limit on bookings at once.
  - **Pets:** which kinds, the biggest dog, pets at once, whether they provide food.
  - **Items:** items at once, the biggest size.
  - Limits count animals and items, not bookings: three dogs take three places, and pets and items each have their own limit (`host_refusal`).
  - Matching skips hosts whose preferences rule a job out. Hosts who haven't set hosting up are still asked to.
  - Empty preferences mean the old behaviour: a host takes anything up to `host_capacity` (`effective_prefs`).
- **Not yet:** a house-sitting sheet ("My home": keys, alarm, plants, pets in the home, mail). The description covers it for now.

### 4.4b The US catalogue

Loaded 2026-10-28 by `scripts/load_catalogue_v2.py` from `data/catalogue/`:
- **Size:** 1,418 skills in 341 professions and 32 categories (`skills.category`, migration 089). Each skill's customer phrases are stored as its description, and the AI's skill matching reads them.
- **Kept skills:** a skill with the same name as before is kept and updated, so providers' skills and past jobs carry on.
- **Hidden skills:** India-era skills the catalogue doesn't have are hidden (`is_active = false`), never deleted. "Personal Errands" stays, because the Pickup & Delivery switch uses it.
- **Carried providers:** a provider verified in a hidden skill's profession got its catalogue profession's skills, verified (`PROFESSION_MAP`). Every provider was a test or demo account then.
- **The pickers** filter by category.
- **The voice assistant** is given profession names only, by category. The model describes the task, and the server matches it to a skill of that profession.

### 4.5 The skills catalogue grows from demand

1. The AI maps each request to existing skills when it can.
2. If nothing fits, the job is still posted as **General help** — matched on trust score and requirements only — **but only for low-risk shapes** (custody NONE or ITEM). Anything with LIVING_BEING or HOME custody needs a real skill.
3. Every unmatched request is logged (`unmatched_requests`: text, shape, count, city).
4. When a need repeats, the admin portal proposes a new skill; an admin approves it and the existing AI test generator builds its skill test.

### 4.6 Fixed boundaries (never accepted, whatever the shape)

**Decided:** illegal work; sexual/adult services; transporting people (ride-hailing — out of scope). Refused by the assistant (prompt + safety model). Server-side, every posted job must map to a skill in the catalogue, so anything with no profession can't be posted however the request is crafted — the catalogue itself is the enforced boundary.

**Pending (§9 #1):** care of people and licensed medical work. The live catalogue currently *includes* both — Caretaker (elder companion care, baby care, child supervision, post-hospitalization care) and Physiotherapist — so they are accepted today. Blocking them means removing/deactivating those professions, not a prompt change.

## 5. AI assistant changes

- Reword from "US home-services marketplace" to "skilled help, in person or online".
- The completion payload (`DONE:`) gains `shape`, `requirements`, `details`. The assistant asks shape questions **only when the answer isn't obvious** (e.g. "at your place or theirs?", "drop-off and pick-up dates?").
- Before posting, the assistant reads back a one-line summary derived from the shape ("Your dog stays at the sitter's home Fri 6pm – Mon 9am; you'll get daily photos") — this is the customer's confirmation step.
- The server **never trusts the AI's shape blindly**: it validates the enums and dates, recomputes the policy itself, and rejects contradictions (e.g. ONLINE + LIVING_BEING).

## 6. App changes

The server returns a job's protocol (`steps`, current step, `next_actions`). Both apps render **generic step components** — "show my code", "enter the customer's code", "take hand-over photos", "post an update", "approve delivered work" — instead of per-category screens. A new situation reuses existing components.

Provider app additions: base address, "I can host at my place" with capacity, rate card, requirement tags (e.g. "fenced yard", "has a van").

## 7. Data model changes

`jobs`: `venue`, `timing`, `custody`, `pricing_unit` (strings, like `job_type` today), `ends_at`, `recurrence` (JSONB), `requirements` (JSONB), `details` (JSONB), `series_id` (recurring jobs).
`providers`: `base_location` (geography), `base_address`, `hosts_at_base` (bool), `hosting_capacity` (int), `tags` (JSONB).
`provider_rates`: provider × pricing_unit → amount.
`job_step_events`: job × step × actor × time × evidence (photos, codes verified) — the audit trail every step writes.
`unmatched_requests`.

`job_type` and `service_mode` stay during migration and are kept in sync; they are removed only after every reader uses the shape.

Recurring jobs: a `series` row plus one ordinary `Job` per occurrence, so every existing feature (payment, rating, disputes) works per occurrence unchanged.

## 8. Delivery plan

Each phase ships with tests, keeps today's behaviour for existing jobs, and updates `IMPLEMENTATION_STATUS.md`.

| Phase | Scope | Behaviour change |
|---|---|---|
| **1 · Foundation** ✅ | `JobShape` + `policy_for` + migration 045 with backfill; `create_job` stores the shape; responses return `shape` + `steps`; policy covers today's in-person and pickup flows, proven by parity tests | None |
| **2 · Step engine** ✅ | `StepHandler`s for today's steps; `JobFlow` enforces step order and refuses shapes whose steps aren't built; `JobService` delegates (no job-type branching left); `MatchingStrategy` per venue; one shared location check; `job_step_events` audit (migration 046) | None — online jobs keep today's quote flow; arrival-code/inspection/pickup actions on a job that doesn't have that step are now refused (apps never call them) |
| **3 · AI shapes** ✅ | Prompt widened ("in person or online"); `DONE:` carries shape/requirements/details; server checks it (`app/services/job_intake.py`) and has the assistant follow up in the customer's language when it's incomplete or not runnable yet; `UNAVAILABLE:` marker; demand log (`unmatched_requests`, migration 047); plain-language summary on the confirm card; customer app posts the shape, trusts the server's verdict (its stale 14-profession list removed) and hands pickups to the Pickup & Drop flow; 10 online professions / 50 skills added to the seed | Assistant takes online work and refuses gracefully what Dodorez can't run yet |
| **4 · Provider's place + stays** ✅ (2026-10-22, migration 075) | Base address, hosting, capacity, rate card (`app/domain/rate_card.py`, `RateCardService`); `HostMatching` (near the provider's base, hosts only); `PAY_UPFRONT` / check-in code / hand-over photos / periodic updates / hand-back code steps; per-hour/day/night pricing counted in the customer's time zone; cancel terms for bookings paid up front (`app/domain/upfront_terms.py`, MONEY_RULES_DESIGN.md §14). Apps: three plain questions when posting (where / how long / leaving anything), each applicant's total, Book and pay, the check-in and hand-back codes, photos and updates; provider Rates & hosting and a step-by-step care panel | Pet boarding, pet sitting, house-sitting, hourly help. One-off FIXED/VISIT/ITEM jobs from the AI still become quotes. A visit priced by a quote can't also leave something in care or be at the provider's place (`QUOTE_WITH_CARE` in `job_capability.py`) — the quote and care steps aren't joined up, so that work is booked for a set time instead |
| **5 · Recurring** ✅ (as a series of visits — MONEY_RULES_DESIGN.md §13) | Series + occurrences; recurring pricing | Weekly tutoring, dog walking |
| **6 · Catalogue growth** | Admin review of unmatched requests → new skills + generated tests | Catalogue follows demand |
| **7 · Apps** | Generic step components in both apps driven by the server's protocol | New situations need no app release |

## 9. Decisions needed from founders/partners

1. **Care of people and licensed medical work**: the catalogue has Caretaker (elder/child care, post-hospital care) and Physiotherapist today, so they're accepted. Childcare, elder care and physiotherapy are licensed in many US states — recommendation: deactivate those skills for launch, revisit with legal advice.
2. **Stays:** deposit at booking vs. full amount in escrow; cancellation rules for multi-day bookings.
3. **Periodic updates:** default interval for living beings / homes (proposal: every 24 h, at least one photo).
4. **General help:** allowed for low-risk shapes at launch, or only catalogue skills?
5. **Insurance** for custody jobs (pets, homes, vehicles) before launch.
6. **Online work: deliver → approve?** Today an online job is quoted, paid and completed like any other. Option: the provider submits the work and the customer approves it before payment is released (Upwork/Fiverr style).
