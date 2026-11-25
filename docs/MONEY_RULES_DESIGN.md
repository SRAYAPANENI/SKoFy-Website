# Money rules: fees, cancellations, inspection proof, disputes

**Status:** built and deployed 2026-10-21 (migrations 065–069), in Stripe test
mode. This replaces the flat 12% take rate and fills three gaps in how money
moves on a job. Numbers marked *(dial)* are settings in `app/core/config.py`,
to validate with 10–20 real providers before launch.

**Switched off or partly built, on purpose:**
- Strike consequences (levels 2 and 3) wait for the terms of service:
  `STRIKE_CONSEQUENCES_ENABLED=false`. Strikes are recorded, shown and appealable.
- The customer fee replaces the card surcharge (`CUSTOMER_FEE_MODE=SERVICE_FEE`),
  still subject to the lawyer's answer ([LEGAL_QUESTIONS.md](LEGAL_QUESTIONS.md) §5);
  `CARD_SURCHARGE` restores the old behaviour.
- Fixed plans are recorded by an admin (portal → provider → Fees) once paid;
  self-serve billing isn't built.
- Instant payout isn't built: it needs Stripe Connect payouts, which aren't
  enabled on the account yet.
- The AI photo check fails open: if the model is unavailable, the photos count
  as unchecked, never as unrelated.
- The minimum time on site only flags; it never blocks.
- "On the way" means actually travelling: half the visiting fee needs the
  provider to be at least 0.3 km closer than when they set off (migration 070);
  unknown location gets the benefit of the doubt.
- After an inspection report, a cancel doesn't reopen the job to other
  applicants — the visiting fee settles on that report's proof, which a reopen
  would wipe. The customer reposts.
- A deposit is paid when work starts (decided 2026-10-21); a refund after it
  returns only what's still held, and support recovers the deposit if due.

What this covers, in the order money moves on a job:

1. [Fair Cap](#1-fair-cap--what-providers-pay): what providers pay Dodorez.
2. [Customer fee and payment methods](#2-customer-fee-and-payment-methods).
3. [Cancellations by stage](#3-cancellations-by-stage): who gets what when either side cancels.
4. [Proving an inspection happened](#4-proving-an-inspection-happened).
5. [Disputes and customer strikes](#5-disputes-and-customer-strikes).
6. [The whole workflow](#6-the-whole-workflow), [what to build](#7-what-to-build), [build order](#8-build-order), [open decisions](#9-open-decisions).
10. [Paying outside the app](#10-paying-outside-the-app) (built 2026-10-22, migration 071).
11. [Less pain on a job](#11-less-pain-on-a-job): visit counts toward the quote, parts paid at the start, added work, tips, typical prices, ways to pay (migration 072).
12. [Promises that cost Dodorez](#12-promises-that-cost-dodorez): the guarantee, chargebacks, dispute deadlines, payout dates, the reserve (migration 073).
13. [Repeat bookings](#13-repeat-bookings) (migration 074).
16. [Pickup & Delivery fares](#16-pickup--delivery-fares): priced up front by the trip, like a delivery app (migration 099).

## Principles

- **Charge only when the provider is paid.** No lead fees, no sign-up fee, no
  required subscription.
- **Big jobs must not hurt.** A flat percentage punishes exactly the jobs most
  likely to move off-platform.
- **No surprises for either side.** Every amount (fee, cancellation, refund) is
  shown before the person confirms.
- **Money follows evidence.** Where someone actually was, what they recorded and
  what the other person confirmed decide who gets paid, not one side's word.
- **No fines on providers.** Their consequences are reliability (HCI) and pauses.
  Deducting pay from independent contractors is painful and looks like
  employer control ([LEGAL_QUESTIONS.md](LEGAL_QUESTIONS.md) §1).
- **One rules module.** Every rule below is a pure function in `app/domain/`, so
  fees, cancellations and releases can't contradict each other.

## What exists today

| Area | Today | Code |
|---|---|---|
| Provider fee | Flat 12% of every payment, visiting fee included | `PLATFORM_FEE_PERCENTAGE`, `offer_pricing.platform_fee_for` |
| Customer cost | Job amount plus the card fee (2.9% + 30¢), uncapped | `offer_pricing.checkout_breakdown` |
| Visiting fee | Set by each provider, held at hire, credited toward the final price if the customer goes ahead | `providers.visiting_fee`, `JobService._finalize_hire` |
| Visiting fee release | Only when the visit is done: inspection completed, item picked up, or pickup refused | `domain/settlement.settlement_for`, `job_service._visit_done` |
| Settlement | All or nothing: release, refund, or the goods split | `Settlement` |
| Cancel cause | Recorded on every cancel; only PROVIDER and NO_SHOW count against reliability | `domain/cancellation.py` |
| Inspection start | The customer reads the provider a code, with a location check | `request_inspection_otp`, `verify_inspection_otp` |
| Inspection done | One tap, with no proof, no customer confirmation, and the fee paid instantly | `JobService.complete_inspection` |
| Disputes | An admin picks the outcome: refund the customer or pay the provider. Nothing is recorded about the customer's pattern | `JobService.resolve_dispute` |

The gaps, in short:
- 12% is too much on big jobs.
- The customer's card fee is uncapped, and may count as an illegal surcharge.
- A provider who travels gets nothing if the customer cancels before the inspection.
- There's no "customer wasn't there" option.
- "Inspection done" proves nothing.
- Disputes don't record whether a claim was false.

---

## 1. Fair Cap: what providers pay

Four rules:

1. **Only on paid jobs.** No fee on leads, applications, or jobs that don't go ahead.
2. **Banded rate, like tax brackets:** **10%** on the first **$500** of a job and
   **4%** on the part above *(dial: 10% / 4% / $500)*.
3. **Per-job cap: at most $119** for any one job *(dial)*.
4. **Monthly cap: $249** in fees per calendar month *(dial)*. After that,
   **2%** for the rest of the month *(dial, `FEE_RATE_AFTER_CAP`)*.

**Decided 2026-10-22: 2% after the cap, not 0%.** At 0%, a provider earning
$20,000 a month paid the same $249 as one earning $2,500, so revenue stopped
growing with the busiest providers. At 2% it keeps growing, and 2% is still far
too little to be worth leaving the platform over.

**Decided 2026-10-24: in a business, each member has their own monthly cap.**
A business is one payee, so with a single cap a 10-person company paid at most
$249 a month, the same as one person. Individuals could also form a "business"
just to share a cap. Instead, **each member's fees for the business count
against their own $249 cap**. Past it, their work pays 2% for the rest of the
month. The business's cap is therefore $249 × the members who earned for it
that month. A one-person business pays what an individual does, and a team
pays exactly what its people would pay alone.

Not one shared pot of $249 × members: with uneven earnings that charges the
busy member full fees on the quiet member's unused room, so the team pays more
than its people would alone. It is still one bill and one cap figure on the
owner's earnings screen. A member's own work (two hats) has its own separate
cap. The per-job cap ($119) is unchanged.

Code: `FeeRepository.fee_context` counts the member's share,
`member_month_fees` and `business_month()` give the owner's figure.
`scripts/smoke_fee_cap.py` was run against a copy of the database.

The monthly cap works like transit fare capping: pay per ride until the cap,
then ride for a token fare. It's a subscription the provider never buys in
advance. It is also the answer to leakage (ROADMAP #6): the busiest providers
are the most tempted to take customers off-platform, and once capped, staying
costs them almost nothing. It competes directly with GigNGo's flat subscription (ROADMAP #4)
without making part-timers pay up front.

**What one "job" is.** The bands and the per-job cap apply to the total the
provider is paid for the job: visiting fee plus job cost. The visiting fee is
already credited toward the final price, so it must not be charged its own
first $500 band.

**Formula (pure):**

```
job_fee(total)       = min(0.10 × min(total, 500) + 0.04 × max(total − 500, 0), 119)
room                 = max(249 − fees_already_charged_this_month, 0)
charged(payout)      = added            if added ≤ room       (added = what the payout adds to job_fee)
                     = min(room + 0.02 × the payout's share past the cap, added)   otherwise
```

The fee is taken when money is released to the provider, not when it's held.
A later refund (a dispute won by the customer) reverses that job's fee and
lowers the month's total.

**What the provider keeps:**

| Job | Today (12%) | Fair Cap fee | Keeps |
|---|---|---|---|
| $100 | $12 | $10 (10%) | $90 |
| $500 | $60 | $50 (10%) | $450 |
| $1,000 | $120 | $70 (7%) | $930 |
| $3,000 | $360 | $119 (4%) | $2,881 |
| $10,000 | $1,200 | $119 (1.2%) | $9,881 |
| A $1,000 job after the month's $249 | $120 | $20 (2%) | $980 |

Rough reference points, to check before using in investor material: Fiverr
takes 20%, rideshare 25%+; a Thumbtack pro buying five ~$40 leads to win one
$1,000 job effectively pays $200.

**Options:**
- **Fixed plan:** $199/month paid up front gives 0% all month *(dial)*. It suits
  businesses that know they'll hit the cap and want a fixed cost. It's cheaper
  than the earned $249 cap, in exchange for paying in advance.
- **Business plan (later):** charges for *features* (team members, quotes,
  branded invoices), never for a lower rate. Same fee rules for everyone.
- **Instant payout:** same-day payout for about 1% (roughly Stripe's own
  cost); standard payout in 2–3 days stays free.
- **Never sold:** ranking, visibility or leads. The HCI score decides who
  sees a job.

**Offers and discounts.** Today an offer's discount comes out of the platform
fee (`offer_discount`). That stays: a discount can lower Dodorez's fee on a job,
never the provider's pay.

**Shown to the provider:** every job card and quote screen says "You'll keep
$X". The earnings screen shows the month's fees against the $249 cap.

## 2. Customer fee and payment methods

- **One customer fee, the same for every payment method:** **3%, at most $15**
  *(dial)*. It replaces the card surcharge, and every customer pays the same as
  today or less ($100 job: $3 instead of $3.20; $3,000 job: $15 instead of
  $87.30). Subject to the lawyer's answer on surcharging
  ([LEGAL_QUESTIONS.md](LEGAL_QUESTIONS.md) §5).
- **Bank payment (ACH) for jobs over $500,** offered as the default. Stripe
  charges at most $5, against $87 by card on a $3,000 job. Escrow already holds
  the money, so ACH's slower settlement doesn't matter.
- **Milestones for jobs over $1,000:** for example a deposit, then the rest on
  completion, each held in escrow and released separately. Customers feel safer
  and providers are paid sooner. Each milestone is a "payment" for Fair Cap's
  monthly total, but the bands and per-job cap apply to the job as a whole.

**Dodorez's net per job** (provider fee + customer fee − Stripe's cost):

| Job | By card | By bank |
|---|---|---|
| $100 | $9.80 | — |
| $1,000 | $55.70 | $80 |
| $3,000 | $46.70 | $129 |

Small jobs earn about what they do today. Big jobs earn less than 12% does on
paper, but at 12% big jobs mostly leave, and earn $0. The model depends on two
things: getting big jobs onto bank payment, and keeping repeat work on-platform
(the monthly cap).

## 3. Cancellations by stage

The stage comes from what the app already records: hire, "Set off" (with live
GPS), arrival (location-checked), the visit (start code, then inspection or
pickup).

**The customer cancels:**

| When | Provider gets | Customer gets back |
|---|---|---|
| Within 5 min of hiring *(dial)*, or before the provider sets off | $0 | 100% |
| After the provider sets off | 50% of the visiting fee *(dial)* | 50% |
| After the provider arrives (location-verified) | 100% of the visiting fee | 0% |
| **Customer not there** (see below) | 100% of the visiting fee | 0% |

**Customer not there:**
1. The provider arrives and taps "I'm here".
2. The customer gets a push, with a call button.
3. A 15-minute wait starts *(dial)*.
4. After the wait, the provider can tap "Customer not here".
5. The provider must be within the arrival radius for the whole wait.

The job is cancelled with cause `CUSTOMER_NO_SHOW`, which doesn't count against
the provider.

**The provider cancels or lets the customer down:**

| When | Customer | Provider |
|---|---|---|
| Cancels before setting off | 100% back; the job reopens to other applicants (built) | Counts against reliability (built) |
| Never sets off (no-show timer, built; an urgent deadline: by an hour before it, or 15 min after the hire if later; a teammate given the job gets the grace from then, up to an hour past the first limit), or arrives more than 30 min after the booked time or the ETA they were given *(dial)* | 100% back, a free cancel, and a **$10 Dodorez credit** *(dial)*, paid by Dodorez, not the provider | Counts against reliability |
| 3 of the above in 30 days *(dial)* | — | Paused from new jobs for 7 days |

**Rules:**
- **Dodorez takes 0% of cancellation money.** Every cent of a trip or visiting
  fee from a cancellation goes to the provider.
- The **Cancel** screen always shows the split before the customer confirms:
  "Cancel now: $20 back to you, $20 to Sam for the trip."
- New cancel causes (`app/domain/cancellation.py`): `CUSTOMER_NO_SHOW` and
  `PROVIDER_LATE`. `PROVIDER_LATE` joins `PROVIDER_FAULT`.
- Pickup jobs already follow this shape: no cancelling once the item is
  collected, and the trip is paid on a refused pickup. The same stage table
  applies before collection.
- **While paused (built 2026-11-18):** no new job is sent, none comes in on
  moving into range, and jobs sent before the pause leave the feed — only
  ones already applied to stay (`_INBOUND_FROM_WHERE`). The provider is told:
  their profile carries `paused_until`, `pause_reason` (`LAPSES` or
  `OFF_APP_PAYMENT`) and `pause_lapses` (each with its date and job), and the
  dashboard shows a red banner with them (`components/paused-banner.tsx`).
  Before this, a paused provider saw only an empty job list.
- **Lifting a pause (built 2026-11-18, migration 114):** the portal's provider
  panel has a *New jobs* section — paused until when, why, the lapses — and
  **Lift pause** (`POST /providers/admin/{id}/lift-pause`, `providers.manage`,
  a reason required). It ends the pause now and **forgives** the lapses so far
  (`provider_lapses.forgiven_at`): kept on record, no longer counted, or the
  next lapse would pause them again at once. The last lift, who and why, is
  kept on the provider (`pause_lifted_*`), and the provider gets a push.

## 4. Proving an inspection happened

The app can't see inside a home. It can make faking an inspection hard and make
an honest one easy to prove. The best witness is the person who's there: the
customer.

### Layer 1: the customer confirms at the end

The start code already proves both people are there. Add the mirror at the end:

1. The provider taps **Inspection done** and submits the **inspection report**:
   - what they found (required, a few lines);
   - at least one photo, taken in the app;
   - the result: a quote, or "no work needed".
2. The customer's app shows the report: **"Sam says the inspection is done."**
   - **Yes, it's done:** confirmed; the visiting fee is released at once.
   - **Not done yet:** the provider is told to carry on.
3. **Accepting the quote counts as confirming.** Normal jobs get no extra step.

**One way to confirm (2026-11-05).** There used to be a 4-digit end code as
well, for the customer to read out to the provider. It added nothing: the
code was in the customer's own app, so anyone who could read it could tap
"Yes" instead. Both apps dropped it; the server keeps its routes until every
installed app has updated. The start code stays: it proves the customer was
there when the provider arrived.

### Layer 2: evidence, when the customer can't or won't confirm

| Check | Proves | Today |
|---|---|---|
| Start code entered | The customer was there and let them in | Built |
| Time on site: the provider's location keeps updating from the start code to "done" | They stayed, and for how long | Location updates exist for travel; extend them through the inspection |
| In-app camera only (no gallery), each photo stamped by the server with time and location | The photos were taken there, then | New |
| Photo fingerprint compared against other jobs | No reused photos | New |
| AI check of photos and report against the job (`GROQ_VISION_MODEL`) | It's about this job (an AC job shows an AC unit) | Model configured; check is new |
| Time on site under a minimum per skill (e.g. 3 min) *(dial)* | It wasn't a ten-second tap | New. A **flag only**: the old 20-minute hard minimum was removed on purpose (ROADMAP #2) |
| Location at "done" inside the job radius | Submitted on site | New |

### Confidence and money

| Level | When | Visiting fee |
|---|---|---|
| **Confirmed** | The customer tapped "Yes", or accepted the quote | Released at once |
| **Evidence-backed** | Not confirmed; every Layer 2 check passes | Released after 24 hours unless disputed *(dial)* |
| **Needs review** | Any check fails | Held; support decides |

What this can't tell: whether the inspection was *good*. Quality is for
ratings, reviews and disputes.

## 5. Disputes and customer strikes

### Deciding a dispute

A customer can dispute the visiting fee ("Inspection wasn't done") while it's
held. The dispute opens with an **evidence pack** built automatically:
- the start-code time;
- arrival and the location trail, and time on site;
- the report and its photos, with their time and place;
- the chat and the call log;
- the confidence level and any failed check.

**Both sides then have 48 hours *(dial)*** to add their account, in words and
photos.

Support records two separate things:

- **Outcome (money):** refund the customer, or pay the provider (as today).
- **Finding:**
  - *Provider at fault*: counts against the provider's reliability; the
    3-in-30-days pause applies.
  - *Honest disagreement*: no consequence for anyone. **The default when in
    doubt.**
  - *Customer's claim contradicted by evidence*: one **customer strike**. Only for
    clear evidence: for example, the customer read the start code, the provider
    was on site 25 minutes, the photos were taken at their address, and the
    customer thanked them in the chat.

A confirmed inspection (Layer 1) can still be disputed, but the confirmation
is strong evidence for the provider.

### Customer strikes

Stored in the `strikes` table (was `customer_strikes` until migration 071,
which added `role` and `reason` for §10): person, job, reason, admin, note,
created, **expires 12 months later**. Providers never see a customer's strikes.

| Strikes in 12 months | Consequence |
|---|---|
| 1 | An in-app notice: what was reviewed, what the evidence showed, how to appeal. No restriction. |
| 2 | Their 24-hour hold ends: their visiting fees are released when the report arrives. A future "not done" claim requires photos and a description. |
| 3 | The account goes to a senior agent, who may suspend it (existing deactivate-with-reason). |

**Appeal:** within 14 days, from the notice. It goes to a **higher-level agent
in the Disputes department who wasn't involved**, using the support levels and
departments in [SUPPORT_SYSTEM.md](SUPPORT_SYSTEM.md). A successful appeal
removes the strike.

**Early warnings, not penalties:** the customer's page in the admin portal shows
their dispute history (disputes per job, distinct providers, findings). A
pattern such as four disputes against four providers in two months is flagged
for a person to look at. It never creates a strike by itself.

The terms of service must describe strikes, consequences and appeals before this
is enforced ([LEGAL_QUESTIONS.md](LEGAL_QUESTIONS.md) §7).

## 6. The whole workflow

An on-site job with an inspection, and where money moves:

```
Posted → Hired                      visiting fee held (customer pays + 3% fee, max $15)
  │  customer cancels ≤5 min or before set-off → 100% refund
  ▼
Set off (live GPS)                  customer cancels → 50/50 split
  │  provider never sets off → no-show: refund + $10 credit; counts against provider
  ▼
Arrived (location-checked)          customer cancels → 100% to provider
  │  "I'm here" → 15 min → "Customer not here" → 100% to provider (CUSTOMER_NO_SHOW)
  ▼
Start code (customer reads it) → Inspecting   location keeps updating
  ▼
Inspection report (findings, photo, quote)
  ├─ customer confirms / accepts quote → visiting fee released (Fair Cap fee)
  ├─ no answer, evidence passes → released after 24 h
  ├─ evidence fails → held for review
  └─ customer disputes → evidence pack → 48 h → support: outcome + finding (→ strike?)
  ▼
Quote accepted → job cost held → work → complete → job cost released (Fair Cap fee)
```

Pickup & Drop keeps its own proof steps (pickup photo, item check, drop-off
code), with the same cancellation stages before collection and the same Fair
Cap fee on release.

## 7. What to build

**Pure rules (`app/domain/`), each with its own tests:**
- `fees.py`: `job_fee(total)`, `charged(job_fee, month_to_date)`,
  `customer_fee(amount)`, all settings-driven. Replaces `platform_fee_for`.
- `cancellation_terms.py`: from the job's stage (hired / set off / arrived /
  customer not here / visit done) and who cancels, returns the split and the
  cause.
- `inspection_proof.py`: the Layer 2 checks, returning a confidence level.
- `strikes.py`: active strikes, then the consequence level.
- `settlement.py`: add `PARTIAL` (an amount to the provider, the rest refunded).

**Data (migrations):**
- `jobs`: `arrived_at`, `customer_wait_started_at`, `inspection_report`
  (findings, photo URLs, result), `inspection_confidence`,
  `inspection_confirmed_at`, `inspection_confirmed_by` (`CUSTOMER` / `CODE` /
  `QUOTE` / `AUTO`).
- `payments`: `fee_month` (the calendar month the fee counts toward) and the
  fee actually charged, for the monthly sum.
- `provider_fee_plans`: fixed plan, from–to dates.
- `customer_strikes` (as above), and dispute `finding`.
- `customer_credits`: the $10 credits, applied at the next checkout.

**Server:**
- Checkout uses the customer fee; ACH as a payment method.
- Release applies Fair Cap with the month-to-date sum.
- Cancel endpoints return a **preview** (the split) before confirming.
- "I'm here", "Customer not here", the late free cancel, the 3-in-30 pause.
- The inspection report, customer confirm, end code, 24-hour release sweep.
- The evidence pack, the 48-hour window, finding plus strikes, appeal routing.

**Apps:**
- **Provider:** "You'll keep $X" on cards and quotes; monthly fees against the
  cap; "I'm here" with the wait timer; the inspection report (camera only).
- **Customer:** fee and ACH at checkout; the cancel preview; confirm-or-dispute
  on the report; the end code; the strike notice and appeal.

**Admin portal:** the evidence pack; outcome plus finding; dispute history on
the customer page; appeals at a higher level; the providers' fee plans.

## 8. Build order

Each stage is tested, deployed and verified live before the next one starts.

1. **Rules:** the pure modules and their tests. No behaviour change yet.
2. **Fair Cap and the customer fee:** release and checkout use them, with the
   month-to-date sum; the provider sees "You'll keep $X". Waits on the
   surcharge answer for the customer fee; Fair Cap itself doesn't.
3. **Cancellations by stage:** `PARTIAL` settlement, the cancel preview,
   "customer not here", the late free cancel, credits, the pause.
4. **Inspection proof:** the report, Layer 1 confirmation, Layer 2 checks, the
   24-hour release.
5. **Disputes and strikes:** the evidence pack, the finding, strikes, appeals,
   the portal views.
6. **ACH and milestones**, which need the most Stripe work.

## 9. Open decisions

| Decision | Recommendation |
|---|---|
| A minimum visiting fee for in-person jobs, or let providers choose $0 | **$15 minimum.** A $0 visit makes every cancellation free for the customer and every trip a loss for the provider. |
| The numbers marked *(dial)* | Validate with 10–20 providers before launch |
| Calendar month for the cap: UTC or the provider's state time zone | The provider's time zone (`providers.state`), so "this month" matches theirs |
| The customer fee instead of the card surcharge | Waits on [LEGAL_QUESTIONS.md](LEGAL_QUESTIONS.md) §5 |
| Strikes and consequences in the terms | Waits on [LEGAL_QUESTIONS.md](LEGAL_QUESTIONS.md) §7 |
| Who funds the $10 credit | Dodorez, as a retention cost; never the provider |


## 10. Paying outside the app

Decided and built 2026-10-22 (migration 071). Customer and provider meet
through Dodorez and may agree to pay in cash, cutting out the fees. No
marketplace can prevent it entirely. The aim is to make it visible, make the
risk clear to both sides, and make asking for it a strike. The incentives
stay the main defence. Escrow, refunds, disputes, cancellation pay and the
payment guarantee only cover in-app payments. Fair Cap drops busy providers
to 2% after the cap, so cash saves them almost nothing.

**1. Chat screening** (`app/domain/off_app.py`, pure). Each text message is
scanned as it's sent:

| Found | What happens |
|---|---|
| A phone number (US formats, or any number with a `+` country code) or an email | **Hidden** from the other side (`[hidden]`, the push preview too). Calls and chat stay in the app while the job runs and close when it ends ([job_contact.py](../skofy-backend/app/domain/job_contact.py)), so a number is only a way around that. Under the message: "Phone numbers and emails are hidden — call and chat here instead." |
| A payment app (Venmo, Zelle, Cash App or a `$cashtag`, PayPal, Apple Cash, Google Pay, Chime), "cash", or deal talk ("outside the app", "pay me directly", "text me at", WhatsApp, "my number") | **Delivered** as written, and marked under the message: "Pay only in the Dodorez app — money paid outside it isn't protected." |

Every flagged message also creates an `off_app_reports` row for support,
holding the original text. Ordinary messages are left alone: prices, gate
codes, times, dates and "my phone died" all pass.

**2. The follow-up question.** The customer's rating (every rating screen)
asks "Did *provider* ask you to pay outside the app?" with optional Yes or
No and an optional note. A yes becomes a report about the provider. It's
asked only of customers, and only when they rate. The rating reminder
already chases unrated jobs.

**3. Support decides.** In Portal → Off-app Payments there is one case per
job and person. A case shows every report, whether the customer said yes,
and the person's active off-app strikes. The decision is **strike** (with an
explanation the person reads) or **no action**. Sharing a number to say "I'm
outside" is not the same as asking for cash, so a person always decides.

**Strikes for asking to pay outside the app** (`Reason.OFF_APP_PAYMENT`):
these use the same `strikes` table, 12-month expiry, appeal and removal as
§5. They are counted separately from contradicted-claim strikes and apply to
providers and customers alike.

| Active off-app strikes | Customer | Provider |
|---|---|---|
| 1 | A notice | A notice |
| 2 | A notice | **Paused from new jobs** for `OFF_APP_PAUSE_DAYS` (30) |
| 3+ | Support is prompted to review the account for removal | The same |

The pause applies only with `STRIKE_CONSEQUENCES_ENABLED` (off until the terms
describe it, [LEGAL_QUESTIONS.md](LEGAL_QUESTIONS.md) §7). Strikes are
recorded, notified and appealable regardless. Providers see theirs in the
app under Settings → Account standing; customers see theirs under Credits &
account.

**Not built:** a guarantee for in-app jobs (needs terms and a cost decision);
asking the provider whether the customer offered cash; screening photos.


## 11. Less pain on a job

Decided and built 2026-10-22 (migration 072). Each rule fixes a place where one
side was out of pocket or unsure.

**The visiting fee counts toward the quote** (`app/domain/visit_credit.py`,
`VISIT_FEE_COUNTS_TOWARD_JOB`). When the customer accepts a quote, the
visiting fee they already paid comes off it. They pay the quote, not the
quote plus the visit, like a trade waiving the call-out fee when you go
ahead. The provider still gets the whole quote, and Fair Cap's fee is worked
out on the whole job. A quote that's turned down leaves the visiting fee with
the provider. The quote form shows "they pay $X more", and the customer's
checkout shows "Visiting fee you paid −$X". Anything left under 50¢ (the
smallest card charge) is waived rather than failing the payment.

**Parts paid when work starts, on any job** (`app/domain/milestones.py`). A
quote of $1,000 or more can still ask for up to 50% at the start. A smaller
quote can ask for up to what the parts listed on it cost, never more than
50%. The customer sees the parts list before accepting.

**Added work mid-job** (`app/domain/extra_work.py`, `ExtraWorkService`).
When the provider finds more to do, they ask for it in the app: what it is
and the price ($5–$20,000). The customer approves and pays it into escrow
before it's done, or turns it down and the job goes on as agreed. It carries
the usual fees and is paid out with the job (refunded if the job ends
without it). Each job has one request; it can be asked again after a
"no". Added work left unpaid when the job ends is cancelled.

**Tips** (`app/domain/tips.py`, `TipService`). Once per finished job, within
`TIP_DAYS` (7). Suggested amounts are 10/15/20% of the job, or $5/$10/$20 on
a small one, and the customer can type their own ($1–$500). The provider
gets all of it: no provider fee, no customer fee. It's paid on as soon as
it's held. Tips are offered after rating and in Job history.

**What a job usually costs nearby** (`app/domain/price_guide.py`). The
middle half (25th–75th percentile) of accepted quotes for the same skill
within `PRICE_GUIDE_RADIUS_KM` (40 km) over `PRICE_GUIDE_DAYS` (365), from
at least 5 quotes. With fewer, nothing is shown, because a guess would
mislead. It appears on the post-a-job summary. Beside a quote, a quote more
than 25% above the range is pointed out ("ask what it includes, or
counter").

**Ways to pay** (`app/domain/payment_methods.py`). The card always works,
with Apple Pay and Google Pay. Bank payment (ACH) is offered for the job
price or added work from `ACH_MIN_AMOUNT`. `EXTRA_PAYMENT_METHODS` turns on
Cash App Pay (any payment), and Affirm and Klarna (pay over time, for the
job price). Each needs enabling on the Stripe account first; it's empty
until then.

## 12. Promises that cost Dodorez

Decided and built 2026-10-22 (migration 073).

**The Dodorez guarantee** (`app/domain/guarantee.py`, `GuaranteeService`).
Work paid for in the app is covered for `GUARANTEE_DAYS` (30) after it's
done. The customer claims with a description and photos.

1. The provider is asked first. They can offer to come back and fix it at
   no charge; chat and calls reopen while they arrange it
   (`app/services/contact_gate.py`). The customer then says whether it's
   fixed.
2. If the provider declines, or the fix didn't work, support decides. They
   either refund (Dodorez pays: the charge is refunded from Dodorez's
   balance, and the provider keeps their payout), up to what was paid for
   the work and `GUARANTEE_MAX_REFUND` ($1,000), with an optional lapse when
   the provider was at fault; or they decline, with the reason.

Not covered: a different problem, damage after the job, the customer's own
parts, and work paid outside the app.

**Who bears a lost chargeback** (`app/domain/chargebacks.py`,
`ChargebackService`).

- Lost while the money was still held: it simply goes back to the customer.
  Nobody bears anything.
- Lost after the provider was paid, where the customer had confirmed the
  work (the inspection, the quote, or "done"): Dodorez
  absorbs it.
- Otherwise support decides: Dodorez absorbs it, or it's recovered from the
  provider's next payouts. A recovery is never more than they got from that
  payment, and the provider is told why.

Recoveries are taken in the payout's own transaction and shown on the
provider's earnings.

**Dispute deadlines** (`app/domain/dispute_deadlines.py`, hourly task).

- A visiting fee held for a decision is decided on the evidence if nobody
  has decided it in time:
  - $50 or less (`SMALL_CLAIM_AMOUNT`): when the 48-hour statements window
    closes;
  - more: after `DISPUTE_DECISION_HOURS` (72).
- The evidence decides: a dispute of a visit the checks also flagged is
  refunded; otherwise the provider is paid. This is never a strike or a
  lapse.
- A dispute about the whole job is never decided by rule. Past the deadline
  it's marked overdue, shown first in the portal, and both sides are told a
  senior agent has it.

**When payouts arrive** (`app/domain/payout_timing.py`). Each payout on the
provider's earnings says what it was for (the visit, the job, added work, a
tip) and the day it reaches their bank: `PAYOUT_BUSINESS_DAYS` (2) business
days after release. Instant payout needs Stripe Connect payouts, which
aren't enabled yet.

**The protection reserve** (`app/domain/reserve.py`, portal → Guarantee &
Chargebacks → Reserve). `RESERVE_PERCENT` (5%) of fees is counted toward
credits, guarantee refunds and absorbed chargebacks. The view shows whether
that share covers them, so the promises are priced in. It's a view, not a
separate bank account.

## 13. Repeat bookings

Decided and built 2026-10-22 (migration 074, `app/domain/recurring.py`,
`SeriesService`). This is phase 5 of
[JOB_SHAPE_DESIGN.md](JOB_SHAPE_DESIGN.md): a series, plus one ordinary job
per visit. Payments, ratings, disputes and the guarantee all work per visit,
unchanged. Repeat work is where cash takes over, so repeating has to be
easier in the app than outside it.

- From a finished job (not a delivery), the customer books the same
  provider weekly, every two weeks or monthly, from a first visit they
  choose. They can set an amount up to which a quote is accepted
  automatically. The provider accepts the series once.
- Each visit is created `SERIES_LEAD_HOURS` (48) ahead, as a direct request
  to that provider. The provider confirms it until `SERIES_CONFIRM_HOURS`
  (12) before it starts; after that it goes to other providers like any
  declined request.
- When the provider confirms, the visiting fee is paid from the customer's
  saved card. A quote at or under the customer's amount is accepted and
  paid the same way, still held until the work is done. Above it, or if the
  card needs the customer, they're asked as usual.
- The customer can pause, resume or stop it, and the provider can stop it,
  under Regular bookings / Regular customers.
- Visits keep their local time in the customer's time zone (sent by the
  app), so daylight saving doesn't move them an hour. Monthly visits return
  to the first visit's day: the 31st becomes the 28th in February, then the
  31st again in March.

## 14. Bookings paid up front

Decided and built 2026-10-22 (migration 075, `app/domain/rate_card.py`,
`app/domain/upfront_terms.py`). This is phase 4 of
[JOB_SHAPE_DESIGN.md](JOB_SHAPE_DESIGN.md). A job for a set time (a few
hours, or a stay over days or nights) is priced by each provider's own rate
card instead of a visit and a quote. There's no visiting fee.

- **The price.** Each applicant's total for the customer's dates is shown
  before they choose: rate × units. Nights are counted between local dates
  (at least 1), days include both ends, and hours round up to the half-hour
  (at least 1). Units are counted in the customer's time zone. Rates are
  $5–$5,000 per unit, only for skills the provider is verified in.
- **Long stays and day prices** (2026-10-28):
  - **Weekly and monthly prices:** a provider can add these per skill, and the total is the cheapest mix of their prices ("1 week + 3 nights"). The customer sees the breakdown under each applicant.
  - **A long few hours:** it can use the day price (daycare).
  - **No per-hour price on a skill:** it uses the provider's usual hourly rate.
- **Paying.** The full price is paid when the customer books, and held until
  the job is done. The usual fees apply at payout (§1).
- **Not paid in time.** After hiring, the customer has 60 minutes to book and
  pay (`UPFRONT_PAY_MINUTES`). It's never later than the start, and never
  less than 15 minutes. They're reminded 15 minutes before. Then the
  provider is released with no lapse, nothing is charged, and the job
  reopens to the other applicants. A payment that went through but wasn't
  confirmed books it instead. A bank payment still clearing waits
  (`UnpaidBookingService`, migration 076, every 5 minutes).
- **The customer cancels.** It's free 48 hours or more before the start
  (`UPFRONT_FREE_CANCEL_HOURS`), or within 5 minutes of booking. Inside 48
  hours the provider gets one unit (half, for a visit or fixed price). The
  rest is refunded.
- **The customer doesn't come.** At the provider's place, 60 minutes after
  the start (`DROP_OFF_WAIT_MINUTES`), the provider can report it from
  there. The booking is called off and they get one unit. At the
  customer's place, the usual "Customer not here" wait applies (§3).
- **Picked up early.** The provider is paid for the units used plus one, and
  the rest is refunded.
  - **When a weekly or monthly price was used,** those units are paid at the provider's regular price, kept on the booking at hire (`upfront_base_rate`, migration 088). The discount was for the whole stay.
  - **Example:** 14 nights at the 2-week price of $500, at $40 a night regular, picked up after 3 nights. The provider gets 4 × $40 = $160 and $340 comes back.
  - **Cap:** never more than was paid.
  - **The one unit** for a late cancellation or a no-show works the same way.
- **The provider cancels.** Full refund, and a lapse for the provider (§3).
- **The provider isn't there.** At their place, 15 minutes after the start
  (`HOST_WAIT_MINUTES`), the customer can report "nobody's answering" from
  there. The booking is called off with a full refund and a credit, and it
  counts as a no-show for the provider. At the customer's place, a provider
  running late makes cancelling free, with a credit, as for a visit (§3).
- **How long.** "A few hours" is up to 24 hours. A stay is up to 90 days, or 365 days for things left in storage, so a mistyped date can't turn into an outsized charge.
- **Dates are fixed once priced.** After providers apply, the customer
  can't move the start, since every applicant's total is for those dates.
  They cancel and post again.
- **Updates.** While a pet or home is in their care, the provider sends a
  photo update at least every 24 hours (`CARE_UPDATE_HOURS`). They're
  reminded when one is due. If it's 6 hours late (`CARE_UPDATE_GRACE_HOURS`),
  the customer is told and can call.

## 15. One standard way to price each profession

Decided 2026-10-29 (`app/domain/skill_pricing.py`, `data/catalogue/pricing.py`, migration 090; reviewed in `dodorez_pricing_by_profession.xlsx`). Every profession is priced one of five ways. The customer is only asked what that way needs, the provider only sets that price, and the server refuses anything else (`fit_shape`).

| Type | How it works | Professions |
|---|---|---|
| Quote after seeing it | Visiting fee at hire when the provider travels (§3, it counts toward the job), then their invoice. Online or at their shop: just the quote. | 163 |
| Per hour | Hourly price (or their usual hourly rate) × the hours booked, paid when booking (§14) | 95 |
| Per visit | One set price per visit or session, paid when booking (§14) | 75 |
| Stay, per night | Per night, with weekly or monthly prices when cheaper (§14) | 3 |
| Stay, per day | Per day, with weekly or monthly prices when cheaper (§14) | 5 (Neighbor Storage moved here 2026-11-02) |

- **Places:** each skill also says where its work happens: at the customer's place, at the provider's place, online, or pick-up and drop-off (`skills.venues`, migration 091). The apps only offer those places, and the server refuses others. So a plumber can't be booked online. Pick-up and drop-off work (movers, towing, a courier) can also be booked at the customer's place, where it collects.
- **Quote work leaves nothing in care:** the customer is there for it.
- **Fair Cap is the same for all five (§1–2):** the provider's banded fee capped per job and per month, and the customer's 3% capped at $15, on whatever is paid.
- **A skill can differ from its profession** (`SKILL_OVERRIDE`, reviewed skill by skill 2026-11-02). The test for each skill: can the price be known before anyone sees the job?
  - **No:** a visit, then a quote.
  - **A standard job:** per visit. Inspections, tune-ups, an oil change, hemming, a phone screen, an individual tax return, a TV mount.
  - **Time is the measure:** per hour (party henna, tattoos, a photo booth).
  - **Something or someone left in care over days:** a stay. Board & train is per night. A live-in nanny or caregiver is per day, because an hourly booking stops at 24 hours.
  - **Drop-ins while someone's away** (house sitter mail, check-ins and plant watering; daily check-ins on a parent) are **per day**, not per night: nobody stays over. They aren't per visit either, because per visit books one visit and a repeat booking is weekly at most. Per day covers one day or a two-week trip in one booking.
  - **A half day of doggy day care** is a visit, not a full day.
  - **The result:** 106 skills differ from their profession. 653 skills are quoted, 401 hourly, 337 per visit, 18 per day and 9 per night.
  - **When the loader changes a skill's type,** that skill's old prices no longer apply. Its providers can't apply for its jobs until they set the new price; the app tells them so when they try. The loader's dry run lists each change and how many verified providers it affects. `read_catalogue` fails if an exception names a skill that isn't there.
- **Pickup & Delivery** ("Personal Errands", outside the catalogue) is priced by the trip (§16, `KEEP_SKILL_PRICING`). The profession's other errands stay quoted (`KEEP_PRICING`).
- **Shop-only quote work:** only the Auto Repair Shop's six repairs are left (`test_every_catalogue_skill_can_be_booked_somewhere`). They wait for a drop-off-and-quote flow. Tattoos are now booked by the hour at the studio and piercing per visit. The custom furniture maker also comes to measure.
- **Commission-paid professions** (Insurance Agent, Real Estate Agent and similar) can only be booked as a quoted fee.

## 16. Pickup & Delivery fares

**Built 2026-11-05** (migration 099, `app/domain/trip_fare.py`). A delivery
has nothing to inspect, and its distance is known before anyone moves. So it
isn't priced like a repair (a visiting fee, then an invoice): it's priced up
front, the way a ride or delivery app shows the price before you book.

**Each provider sets their own fare** — Dodorez sets no prices:

```
fare = base fare + price per mile for the size × the trip's miles + the size extra
```

**A big load's own price per mile** (2026-11-17): a car-trunk or van load
costs the driver more *every mile* — fuel, wear, a bigger vehicle — which a
flat extra didn't follow (dear on a short trip, short-paid on a long one). So:

- a car trunk and a van may each have their own price per mile
  (`TRUNK_MILE`, `VAN_MILE` on the rate card — optional). Unset, the usual
  price per mile counts; it can't be saved below the usual one, and if the
  usual one is later raised past it, the usual one counts (never less).
- their size extra is now their **loading fee**, once per trip — and still
  how a provider says they carry that size (even $0), so who gets the job
  doesn't change.
- the fare reads e.g. "$6.00 + 4.2 mi × $2.75 (van) + van loading $10.00".
  The typical guide (no fares nearby) uses $1.90/mi + $5 for a car trunk and
  $2.75/mi + $10 for a van; "Suggest for me" suggests both rates too.

- **Rates & hosting** has two boxes for the Pickup & Delivery skill: a base
  fare ($2–$100) and a price per mile ($0.25–$10). An example shows what a
  5-mile trip would cost. They're stored as rate-card units `TRIP_BASE` and
  `TRIP_MILE`.
- **The trip** is pickup → drop-off, measured **once, when the job is
  posted**, and kept on the job (`jobs.trip_miles`), so the fare never moves.
  - Google's driving distance when `GOOGLE_MAPS_SERVER_KEY` is set
    (`app/services/trip_distance.py`).
  - Otherwise, or when Google doesn't answer in 4 seconds, the straight line
    × 1.3. Posting never fails on the routing service.
  - The job records which it used (`trip_miles_source`: `ROAD` or `ESTIMATE`).
- **The provider's own drive to the pickup isn't charged.** The base fare
  covers getting going, as with a ride app.
- **Size** (migration 100, `jobs.item_size`): the customer picks a bag, a
  box, a car trunk, or a van or pickup truck when posting.
  - **A bag** is always included.
  - **A box** costs the provider's box extra, or nothing more if they haven't
    set one.
  - **A car trunk or a van** only goes to a provider who has set that extra
    (even $0). Setting it is how they say they can carry it. The job isn't
    sent to anyone else — distribution and the catch-up feed both filter on
    it (`can_carry_sql` in `app/models/user.py`). A provider who opens it
    anyway can't apply, and is told what to set.
  - Extras are $0–$200, rate-card units `TRIP_BOX`, `TRIP_TRUNK`, `TRIP_VAN`.
  - The estimate counts only providers who carry the size, and the fare says
    it: "$6.00 + 4.2 mi × $1.50 + van $15.00 = $27.30".

**What the customer sees:**
1. **Before posting:** "About 4.2 mi · usually $12–$16" — the middle of
   nearby providers' real fares for this trip (`GET /jobs/pickup-estimate`).
   With none nearby yet, a typical local fare ($6 + $1.50 a mile), labelled
   as a guide.
2. **On each applicant:** their exact fare ("$12.30 total").
3. **Hiring:** they book and pay the fare into escrow. No visiting fee, no
   invoice.

**The job:** paid up front → pickup photo → under way to the drop-off → the
drop-off code → complete (`policy_for`: `PAY_UPFRONT`, `PICKUP_PROOF`,
`TRACK_EN_ROUTE`, `HANDBACK_CODE`). Nothing is collected until the fare is
paid (`require_ready`), and collecting it starts the job.

**Buying it for the customer**: the goods are held on their card, with the
5% shopping fee ($1.99–$9.99), and settled from the receipt when the job is
done. The provider pays the store **with their own money** and is paid back
then, so buying is opt-in (2026-11-15):

- A provider says they buy for customers by setting **"Buy for customers, up
  to $X"** on the delivery skill's rate card (unit `TRIP_BUY`, $10–$200).
  Empty = only already-paid orders.
- A buy-for-me job reaches only providers whose amount covers its limit
  (`can_buy_sql` in distribution, the fairness top-up, catch-up and the feed;
  `ensure_buys_for` when applying — not at hire: applying is agreeing to buy).
  Matched on the job's own skill (`jobs.skill_id`, the card the trip is
  priced from). A job they've applied to stays in their feed even if they
  lower their amount later. Booking a provider directly for a buy-for-me job
  is refused unless everyone on the shortlist buys that much.
- The platform limit is **$200** (was $1,000): an everyday shop.
- **The shopping fee goes to the provider** (was kept by Dodorez): the
  receipt total plus the 5% fee ($1.99–$9.99) are paid to them at settlement
  (`goods_split` → kept = 0), with no Fair Cap fee on either. It's their reward
  for paying first; Dodorez earns on the fare.
- Posting: the estimate says how many nearby providers buy that much, and
  points to "Already paid" (store's own online payment) when few or none do.
- **Later, not built:** a Dodorez card per order (Stripe Issuing, loaded from
  the hold) so no provider uses their own money — needs Stripe approval and
  the lawyer (LEGAL_QUESTIONS.md §14).

**Once the shopping is bought** (2026-11-15, migration 110):

- **A dispute decides the shopping on its own.** It used to follow the fare's
  outcome, so "refund the customer" refunded the receipt too and the provider
  lost what they'd paid. Now (`pickup.goods_dispute_settlement`):

  | The admin chooses | Receipt | Shopping fee | Allowed when |
  |---|---|---|---|
  | Pay the receipt back (default) | to whoever paid | to them if the fare went their way, else back to the customer | always |
  | Refund the shopping | back to the customer | back to the customer | a reason is given (not bought · receipt doesn't match · items kept) **and** the drop-off code wasn't entered |

  Once delivered, the customer has the items: refunding them would give the
  shopping away at the provider's cost, so it's refused — the fare is decided
  separately. The money waits while disputed (`settlement.py` → WAIT). The
  admin sees the receipt photo beside the choice (DisputeDetail), and the
  provider is told when the dispute opens that their shopping money is safe.
- **A business's job: whoever paid is paid back.** A team member's work pays
  the business, but if they paid the store with their own money the receipt
  belongs to them. At pickup they say **"My own money"** or **"The business's
  card"** (`jobs.goods_paid_with`). Own money: the goods payment is moved to
  them (`payments.business_id` cleared, `goods_to_person`), so the payout, the
  fee, their earnings and their tax form all follow it to their own account —
  and they must have their own payouts set up, or pay with the card. The
  business's card: everything to the business. The fare is the business's
  either way.
- **Paid back isn't income.** `payments.reimbursed_amount` keeps the receipt
  part of each goods payout. Earnings (the provider's and a business's) count
  only the fee; the payout list shows "$41.20 receipt + $2.06 shopping fee".
  Tax forms must draw the same line (LEGAL_QUESTIONS.md §14). Payouts from
  before this were the receipt alone and are backfilled as wholly paid back.
- **A payout retried under an older rule is topped up, never paid twice.**
  The goods transfer keeps its fixed idempotency key (a changed key could pay
  twice after a crash). If Stripe refuses it as the same key with a new amount
  (it went out as the receipt alone before the fee moved to the provider), the
  earlier transfer is found by its metadata and only the difference is sent,
  under its own key; nothing matching, or more already sent → logged
  `payout_retry_needs_review` and stopped for a person. Refunds work the same
  way (`_recover_refund`, found on the charge; `refund_retry_needs_review`) —
  a dispute for the customer returns the fee too, so its refund can differ.
  Top-ups are marked `skofy_top_up_of` and never taken for the original.
- **Shopping already paid back** (the job completed, then was disputed):
  "Refund the shopping" is refused with why — it would come out of the
  provider's payout and needs a manual reversal in Stripe — never silently
  skipped. The admin form only offers the choice while the money is held.

**Held on the card, not charged** (2026-11-16, migration 111,
`app/domain/card_hold.py`). A trip's booking (fare + the most waiting can
cost) and buy-for-me shopping money used to be charged in full and the unused
part refunded — 5–10 days to show, so a $6.15 delivery read as $12.51 on the
customer's statement for a week. Now, like a ride or delivery app:

- **At booking** the card is only authorized (Stripe `capture_method=manual`,
  card / Apple Pay / Google Pay only). The payment counts as HELD as before.
- **At settlement** only what's owed is taken (`_take`): the fare, any
  waiting, the fee on them; the rest is released at once — never charged,
  nothing to refund. Called off before set-off: the hold is released whole.
- **Safety net:** a hold still open after 6 days (a disputed trip) is taken
  in full by the 30-minute sweep (`take_aging_holds`), inside the card
  network's 7-day limit, and settled as any held payment from then on.
- A late Stripe "succeeded" can't turn a paid-out payment back to HELD
  (`mark_held` only from PENDING_CAPTURE/FAILED).
- **What the customer sees** (`booking` on the job, the payment's own
  figures): "Price $6.15 · Service fee $0.18 · You pay (no waiting) $6.33",
  "Only if they wait: up to $6.00 more", "We hold $12.51 on your card and
  take only what's used"; the button "Book — $6.33", Stripe's sheet "Hold
  $12.51". Applicant cards say "$6.15 fare" and "+ only if they wait…".
- Longer work (stays, quoted jobs) is charged and held in escrow as before.

**Offers and credit on a booking** (2026-11-16). Booking a job priced up
front goes through the same checkout as an invoice (`/jobs/{id}/book/offers`,
`/book/preview`, `/book/pay` with `offer_code`): the customer's offers, a
promo code, the bill, then "Book — $X" — after hiring, and from the job's
booking card. An offer is on the **price**, never on a trip's waiting
(`offer_pricing.fixed_on`: worked out on the fare, then fixed), capped by
Dodorez's fees as everywhere, so the provider's share never changes.

Settling only part of a payment (a trip's waiting unused, a stay ended
early — `pay_cancellation_share`) now **uses the offer and the credit** as a
full payment does: they pay the provider's part plus Dodorez's fee on it,
less both; credit beyond that goes back to them; under the card minimum
nothing is taken and Dodorez covers it. Before, the offer was silently lost
on a part, and the credit handed back instead of used. The screens use the
same rule (`offer_pricing.part_of`), so "You pay" is exactly what's taken.

**Fixed:** a payment's worth to the provider (`owed_to_provider`) now adds
back the customer's Dodorez credit, as it did an offer — credit is Dodorez's
money. Before, a provider was paid as if the job were worth the credit less;
Dodorez's recorded take is now less the credit instead.

**Waiting can't be run up** (2026-11-16, migration 112, `trip_wait.py`).
Waiting is charged only where the customer's side keeps the provider waiting,
like a ride app — never a store's wait, like a food app:

1. **No waiting fee at a store's pickup** (`pickup_counts`): only a pickup
   from a person and the drop-off are charged. A store wait is covered by the
   fare. Texts say where it's charged (`where_charged`).
2. **The customer can stop the clock**: "It's handed over" in the app
   (`POST /jobs/{id}/wait/stop`) stops the running stop's clock then and there
   (`*_wait_stopped_at`, reason CUSTOMER); the provider is told. Stalling after
   the hand-over earns nothing.
3. **Told before it's charged**: when a charged stop's free minutes end and
   they're still waiting, the customer gets a push (`wait_tasks.notify_paid_waiting`).
4. **Only while the provider is there**: a location more than
   `WAIT_LEAVE_KM` (0.8 km — wider than the arrival check, for GPS drift)
   from the stop stops its clock (reason LEFT), on every location update
   (`stop_waits_left_behind`).
5. **Watched**: providers whose waiting reached the maximum 3+ times in 30
   days are listed for support (admin → Waiting watch) with how each clock
   ended. A long wait isn't a fault on its own — nothing is paused automatically.
6. **No meter for the provider** (`as_provider_sees`): while a charged stop's
   clock runs, the provider's job view leaves out its minutes and money
   (`counting: true`; earlier stops stay counted). They're told it's counted for
   them, and see the result once it stops: "Waiting stopped · N min · $X" when
   the customer or leaving stops it, "Waiting paid · …" on the way to the
   drop-off and after delivery (`WaitResult`). Hidden in the
   API, not just the app, so nothing on the phone can show it. The customer
   always sees the live count. Nothing to watch tick up, nothing gained by
   dragging it out.

A customer could tap "handed over" early to avoid paying; it's recorded with
who and when, for support if the provider disputes it.

**Calling it off** (`upfront_terms.trip_cancel_terms`):

| When | The provider gets | The customer gets back |
|---|---|---|
| Before the provider sets off | Nothing | Everything |
| After they set off | The base fare (their trip) | The rest |
| The item is refused at the pickup (prohibited, not as described, nobody there) | The base fare | The rest |
| The provider cancels, or is late | Nothing | Everything (and the usual credit when late) |

The base fare is kept on the job at hire (`upfront_base_rate`), so the share
is the provider's own base fare out of the fare paid.

**Pickups posted before trip fares** keep their visiting fee and invoice:
their pricing stays `QUOTE`, and that flow is still there for them.

**Waiting at the pickup and the drop-off** (built 2026-11-05, migration 101,
`app/domain/trip_wait.py`). Decided: **capped and agreed at booking**, so
nothing is ever charged after it.
- **The provider's price:** an optional price per minute on their rate card
  ($0.05–$2.00, unit `TRIP_WAIT`). Without one, waiting is free.
- **The rules are Dodorez's:** the first 10 minutes at each stop are free
  (`TRIP_WAIT_FREE_MINUTES`). After that, each whole minute is charged, at
  most 20 for the whole trip, both stops together
  (`TRIP_WAIT_MAX_MINUTES`). One cap means one hold covers it.
  - The rate card's hint reads these numbers from the server
    (`trip_wait` on `GET /hosting/me`), so changing a setting changes it.
  - The price and both rules are copied onto the job at hire
    (`wait_per_minute`, `wait_free_minutes`, `wait_max_minutes`), so a later
    change never touches a booked trip.
- **The clock, at each stop:**
  - **Pickup:** starts at "I'm here" (location-checked, the existing
    `mark_arrived`). Stops when the item is collected, refused, or the trip
    is called off.
  - **Drop-off:** starts at "I'm at the drop-off"
    (`POST /jobs/{id}/delivery/arrived`, checked against the drop-off,
    `jobs.dropoff_arrived_at`). Stops when the drop-off code is entered.
- **Not for "buy it for me":** the shopping is the job.

**The money:**
1. **Before hiring,** each applicant shows their terms: "Waiting at the
   pickup: free for 10 min, then $0.40/min, up to $8.00".
2. **Booking** pays the fare plus that cap ($8.00) in the one payment.
3. **When it's collected,** what was waited is recorded (`wait_minutes`,
   `wait_charge`). The provider's share of the payment is set to
   (fare + wait) ÷ (fare + cap).
   - **When it's delivered,** the drop-off's minutes are added, out of what
     the pickup left of the cap, and the share is set again.
4. **At completion,** settlement pays that share, with the usual fee on it.
   The rest is refunded to the customer automatically, in the same
   settlement, along with its part of their own fee.
   - A dispute decided for the provider pays the same share, never the whole
     hold (`release_to_provider`).

**Example:** a $12.30 fare at $0.40 a minute holds $20.30. The provider waits
16 minutes, so 6 are charged ($2.40). The provider is paid on $14.70, and
$5.60 comes back to the customer.

**Called off or refused** (`trip_cancel_terms`): the base fare plus the
waiting so far, out of everything held.

**Both sides see the clock:**
- The arrival push at each stop says when the free minutes end.
- The provider's pickup and drop-off screens show "Waiting: 6 min · $2.40"
  (`trip_wait.stage` says which stop's clock is running).
- The customer's booking card shows the same, then the final charge and
  what's coming back.

**The provider app's trip screens.** A trip is paid up front, but it isn't
care work: it uses the pickup and drop-off screens (photo, "I'm here",
Confirm Pickup, drop-off code), never the care steps (`CareJobPanel`: check-in
code, hand-over photos). Before 2026-11-05 it was sent to the care steps, so a
booked trip couldn't be collected from the app.

## 17. Several skills on one job (built 2026-11-25)

A job has one price and one place. `app/domain/skill_groups.py` decides which chosen skills can share it; `services/skill-groups.ts` in the customer app is the same rules (keep them alike — the server refuses anything else).

| Skills | Job | Price |
|---|---|---|
| All quoted | One | One quote |
| Quoted + set price | One, quoted | The provider quotes the whole job |
| All set price | One | Added up, line by line |
| All per hour, day or night | One | Once, at the highest price among them (each at its cheapest) |
| Anything else, or no place in common | Separate jobs, posted together | Each its own |

- `jobs.all_skills_priced` (migration 123): priced from every skill, and sent to / taken by only providers verified in **all** of them (`skills_needed_sql`, `_match_threshold`), with a price for each. Off for every earlier job: they keep their price and the 50% match. A repeat visit keeps its first visit's.
- The make-up of the price is kept on the application (`price_text`) and the booked job (`upfront_text`).
- The summary screen has one section per job and posts them in turn; a failure keeps those already posted and retries the rest. Voice keeps the first job and says the others are posted after it. An older app posting skills that don't fit gets a refusal naming each.
- The AI's suggestions are matched to real skills on the server. In the form only the main one is chosen for the customer (with other quoted ones, when it's quoted): a suggestion never adds to a price by itself. A job can't go out with no skill chosen.
- The base rate used for early ends and cancellations is the rate of the skill that set the price.
