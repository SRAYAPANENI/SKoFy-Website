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
| Never sets off (no-show timer, built), or arrives more than 30 min after the booked time or the ETA they were given *(dial)* | 100% back, a free cancel, and a **$10 Dodorez credit** *(dial)*, paid by Dodorez, not the provider | Counts against reliability |
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
3. If the customer's phone isn't to hand, their app shows a **4-digit end
   code** to read out, the same as the start code.
4. **Accepting the quote counts as confirming.** Normal jobs get no extra step.

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
| **Confirmed** | The customer tapped "Yes" or read the end code, or accepted the quote | Released at once |
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
  ├─ customer confirms / reads end code / accepts quote → visiting fee released (Fair Cap fee)
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
  work (the inspection, the end code, the quote, or "done"): Dodorez
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
- **The provider cancels.** Full refund, and a lapse for the provider (§3).
- **The provider isn't there.** At their place, 15 minutes after the start
  (`HOST_WAIT_MINUTES`), the customer can report "nobody's answering" from
  there. The booking is called off with a full refund and a credit, and it
  counts as a no-show for the provider. At the customer's place, a provider
  running late makes cancelling free, with a credit, as for a visit (§3).
- **How long.** "A few hours" is up to 24 hours; a stay is up to 90 days,
  so a mistyped date can't turn into an outsized charge.
- **Dates are fixed once priced.** After providers apply, the customer
  can't move the start, since every applicant's total is for those dates.
  They cancel and post again.
- **Updates.** While a pet or home is in their care, the provider sends a
  photo update at least every 24 hours (`CARE_UPDATE_HOURS`). They're
  reminded when one is due. If it's 6 hours late (`CARE_UPDATE_GRACE_HOURS`),
  the customer is told and can call.
