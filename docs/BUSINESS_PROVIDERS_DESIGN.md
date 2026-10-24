# Business providers: companies with staff, alongside individuals

**Status:** built 2026-10-23 (migrations 077–080). Being switched on before
the lawyer's review (owner's decision, 2026-10-23). The lawyer's answers to
[LEGAL_QUESTIONS.md](LEGAL_QUESTIONS.md) §1 and §11 are pending and will be
applied when they come. `BUSINESS_ACCOUNTS_ENABLED=false` switches it off
again, and with it off nothing changes for anyone. Smoke-tested against a copy of the real database
(`scripts/smoke_business_providers.py`, 37 checks, rolled back).

**Not in this first version, on purpose:**
- **Calls.** The owner can't join or place a job's calls yet. The live job
  socket is strictly two-party and also carries call signalling.
- **Live chat for the owner.** The owner's job conversation view refreshes
  every few seconds rather than over the live socket.
- **Fixed fee plans for a business.** A business always pays Fair Cap.
- **Waiving the no-poaching rule.** A business can't waive it yet (§7).
- **Repeat bookings with a business.** These are refused for now, so a
  member can't take the business's customer on.
- **Direct requests.** These are always to a person, never to a business.
- **Business prices.** A business's prices (rate card, hosting, visiting fee)
  are its owner's. A business's work at "the provider's place" happens at the
  owner's place, and shares its capacity with the owner's own stays.

**Rules added on re-verification (2026-10-23):**
- **Licence:** a business needs an approved, in-date trade licence to be
  hired, as an individual does.
- **Background-check consent:** joining needs the member to authorise their
  background check (Fair Credit Reporting Act) unless they already have a
  clear one. This is recorded as `check_authorized_at`.
- **Renaming:** changing the name customers see after approval sends the
  business back for review.
- **Mid-job changes:** nobody is removed from a team, or leaves, while on the
  business's unfinished jobs.
- **Own team's jobs:** a business can't apply for a job its own team posted.
- **Paused members:** a paused member can't be given a job.

Today every provider is one person: an independent contractor with one login,
verified and paid as an individual (Stripe Connect `business_type="individual"`
in `PaymentService`). This adds **business providers**: a company such as "ABC
Plumbing LLC" that joins with its own staff. It keeps the C2C thesis in
[VISION.md](VISION.md). Individuals and businesses compete side by side, and
Dodorez still sets no prices.

## 1. The model

- **The business is the provider.** It applies, is hired, holds the reputation
  and is paid to its company bank account.
- **Team members do the work.** Each has their own login, ID check and
  background check, and a role:
  - owner: everything;
  - dispatcher: jobs and team, no money;
  - worker: only the jobs assigned to them, no money.
- **The customer sees** "ABC Plumbing · Mike is on the way", with business,
  licensed and insured badges.
- **One person, two hats** (decided 2026-10-23). A team member may also work
  on their own as an individual, if the business allows it. See §7.
- **Teams are of individual providers, never businesses** (decided
  2026-10-24), so there are no chains of businesses inside businesses.
  Someone who runs a business can't be invited to another one, or accept an
  invite. A member can't register a business while on a team, and is on one
  team at a time.
- **Leaving is on notice; the owner can't hold anyone** (decided 2026-10-24).
  A member asks to leave and goes `BUSINESS_LEAVE_NOTICE_DAYS` (7) days later,
  or sooner if the owner taps **Release now**. They can change their mind
  until then.
  - **During the notice:** they can only be given jobs that start before
    their last day.
  - **When they ask,** both sides are told how many of their team jobs fall
    after the leave date.
  - **On the leave date,** an hourly task (`finish_due_leaves`) completes the
    leave and turns their own jobs back on:
    - **A team job they're doing** (on their way or working) keeps them
      until it's done.
    - **One they haven't set off for** goes back to the owner, who gives it
      on (`JobService.hand_back_to_owner`). So no owner can hold someone by
      booking them past their notice.
    - **A disputed job** keeps nobody: Dodorez settles it without them.
  - **The owner removing someone** is still immediate.
  - **After leaving:** the 12-month no-poaching rule (§7) still applies.

## 2. US onboarding and verification

Dodorez never holds the tax number or owners' details: Stripe collects and
checks them, as it does for individuals today. Dodorez keeps the legal name,
the state and the insurance certificate.

| Check | How | Who reviews |
|---|---|---|
| Legal name, entity type, EIN, owners with 25% or more, representative | Stripe Connect onboarding with `business_type="company"` (KYB; the EIN is matched with the IRS) | Stripe |
| State registration | The Secretary of State record (manual lookup at first; a vendor such as Middesk later) | Portal |
| Trade licences | The existing per-skill licence review, held by the company | Portal |
| Insurance | Certificate of insurance upload: carrier, policy, limits, expiry. **At least $1M general liability** for work at a home or on site (decided 2026-10-23); not required for online-only work. Workers' comp where the state requires it for their staff | Portal |
| Each team member | ID verification and a background check (Checkr), as for individuals | Existing flows |

- **Insurance lapses** are handled by a daily task. It reminds the business
  30 days before expiry, and on expiry pauses it from new jobs until a new
  certificate is approved.
- **Tax forms:** the 1099-K goes to the company's EIN (Stripe files it).

## 3. Jobs: apply, hire, assign

- **Matching** reaches the business by its members' verified skills and
  locations, as it does for individuals (`DistributionAgent`). One business
  appears once per job, not once per member.
- **The team is told.** The first time a new job reaches anyone on the team,
  the owner and dispatchers get "A new job for your team" (`TeamJobAlerts`).
  It's sent once per job per business, and never for direct requests or the
  team's own jobs.
- **The business applies.** Rates and hosting work per business, as on the
  individual rate card ([JOB_SHAPE_DESIGN.md](JOB_SHAPE_DESIGN.md) phase 4).
- **Assigning.** Once hired, an owner or dispatcher assigns a member who has
  the skill and a clear background check. That member does every step:
  - setting off, arrival, check-in and hand-back codes;
  - photos and updates;
  - completion.

  The step audit (`job_step_events`) records the member as the actor.
- **Reassigning** is allowed before the member sets off, for example if
  someone is sick. The customer is told who's coming instead. After set-off,
  the existing cancel and no-show rules apply to the business.
- **No-shows and lapses** count against the business, and against the member
  inside the business (§6).

## 4. The owner's view, end to end

What the owner (and a dispatcher) can follow:

- **Jobs board:** every job and its stage (applied, hired, assigned, on the
  way, arrived, in progress, completed, disputed), and who's on it.
- **Job timeline:** each step with its time and evidence, from
  `job_step_events`. That covers:
  - set-off;
  - arrival with GPS;
  - codes;
  - inspection report and photos;
  - quote and payment;
  - hand-over and care updates;
  - completion and rating.
- **Live location** of members on a job (§5), on one map of the whole team.
- **Chat and calls:** the owner can read and join a job's chat, and call the
  customer. The customer is told "ABC Plumbing's team can see this chat".
  Recorded calls follow [CALL_RECORDING_DESIGN.md](CALL_RECORDING_DESIGN.md)
  consent.
- **Alerts:**
  - a member late or not set off;
  - a customer not there;
  - a dispute or guarantee claim;
  - an update due on a pet in care;
  - insurance expiring.
- **Money:** earnings per job and per member, fees against the cap or the
  fixed plan, and payouts. Owner only.

**Where it shows in the provider app.** The business sits on the home screen,
not only under Profile; `hooks/use-my-business.ts` feeds all of it.
- **Owner or dispatcher of an approved business:** a **Myself | [business]**
  switch (`workspace-switch.tsx`), remembered between visits. The Team side
  swaps the job feed for `team-panel.tsx`:
  - open jobs, each with **Assign**;
  - jobs under way, with their stage;
  - shortcuts to Team jobs, Map and Business;
  - a warning if the team can't be hired.
- **Everyone else with a business tie** gets a one-line row
  (`business-home-card.tsx`):
  - **Owner, not yet approved:** what's still needed.
  - **Worker:** whose team they're on.
  - **Someone invited:** the invite.

Jobs the team can take carry a "For your team" badge in the feed. On job
details they show **Apply as [business]**, which opens Team jobs with that
job's member picker, beside **Apply for myself**. The personal button is
hidden for someone who takes jobs only through the business. A team-job
alert opens the same picker.

## 5. Live location: what's tracked, and when

The owner sees the same live location the customer already gets: the provider
app sends it (`PATCH /providers/me/location`) and it's relayed to the job's
screens.

| Rule | Why |
|---|---|
| **Only while on a job:** from "Set off" until arrival, and while a job is in progress away from home. Never off duty, never between jobs | Tracking employees off duty is a legal risk in several states, and it's the line workers accept |
| **Written notice and acknowledgement** when a member joins a team, and again if the rules change | New York requires written notice of electronic monitoring, acknowledged by the employee; Connecticut and Delaware require notice. Doing it everywhere is simplest |
| **Always visible to the worker:** "Sharing location with ABC Plumbing" shows while it's on, and they can see what the owner sees | Trust, and California's notice-at-collection rules for employee data |
| **History kept as job evidence only:** the GPS points on steps (set-off, arrival, codes), not a breadcrumb trail of every minute | Collect the minimum. A full trail can be added if the lawyer is comfortable |
| **The individual hat isn't shared** (§7): a member's own jobs are never visible to the business | They aren't the business's work |

## 6. Reputation

- **The business has its own score** from its jobs, and that is what
  customers see first.
- **Each member has a personal score** from every job they did, in either
  hat. It stays theirs and goes with them if they leave (the portable-score
  thesis).
- **On the business profile,** customers see the team size, how long members
  have been there, and the share of members with clear checks. Members aren't
  ranked publicly.

## 7. One person, two hats (decided)

| | Working for the business | Working for themselves |
|---|---|---|
| Who's hired | The business | The person |
| Paid to | The company's Connect account | Their own individual Connect account |
| Reputation | The business's score and theirs | Theirs only |
| Insured badge | Yes, the business's insurance | Only with their own certificate |
| Background check | One check, reused | Same check |
| Visible to the business | Yes (§4) | No |

**Who may (decided 2026-10-24).** The owner allows it per member
(`business_members.own_work_allowed`, migration 081). **"All members"**
(`businesses.allow_independent_work`) sets everyone at once and is what new
members start with. Turning any one member off turns "all members" off. An
allowed member then chooses with their own switch (`providers.works_independently`).
That switch is **opt-in** on a team: joining turns it off, and leaving or
being removed turns it back on. Being allowed never starts someone's own jobs
by itself.
A member works for themselves only when both are on (`works_for_self_sql`).
Both rules are checked when they apply and again at hire, because the owner
may turn someone off in between.

**No poaching, whoever they work for (2026-10-24).** A person can be on one
team at a time, and can't register a business while on one. The 12-month
rule follows the **person doing the work**, not who they work for. Someone
who served a customer for ABC can't serve that customer for 12 months in any
of these ways:
- for themselves;
- through a business they start after leaving;
- through another team.

This is checked when a business applies, when it is hired, and when it hands
a job to another member (`BusinessHiring.ensure_not_poaching`). Serving the
same business's customers again is fine. The customer is always free to book
anyone else.
`scripts/smoke_member_own_work.py` was run against a copy of the database.

Guardrails:
1. **One calendar per person,** so no double-booking across both hats (the
   availability and no-show checks are per person).
2. **No poaching:** for 12 months a member can't take a private booking from a
   customer they served for the business, unless the business allows it. The
   same idea as the off-app rule (MONEY_RULES_DESIGN.md §10).
3. **The business decides** whether its members may work for themselves at
   all. That's between employer and employee.

### How a member works their own jobs

- **A switch at the top of the provider app:** "Working as: ABC Plumbing |
  Myself". Each side has its own job feed, active jobs, earnings and
  notifications, each labelled with the hat.
- **Their own provider profile:** turning on "Myself" the first time
  creates it. It has its own rate card, skills and service area, and its own
  Stripe Connect payout account as an individual. The background check is
  reused; ID verification is the same person, so it isn't redone.
- **Their own jobs** come exactly as an individual's do: matched, applied
  for, hired, run step by step and paid to them. The business never sees them.
- **The one shared thing is time.** While they're booked on their own job,
  the business sees them as "Busy" with no details, and can't assign them over
  it. While they're on a business job, their own feed pauses new direct
  requests that clash.
- **Turning it off:** a member can turn "Myself" off at any time. If the
  business stops allowing it, their own active jobs finish normally and no
  new ones come.

**Data:** one login (`users`) can hold an individual provider profile and
memberships in businesses. Build this from the start; the second hat is
switched on with teams (§9, step 3).

## 8. Money

- **Charges and payouts:** the customer is charged as today, and payouts go to
  the business's Connect account. Stripe Connect is still to be enabled on
  the account.
- **Fees:** Fair Cap, or the fixed monthly plan (MONEY_RULES_DESIGN.md: for
  "businesses that know they'll hit the cap"). Each member has their own
  monthly cap for the work they do for the business, so the business's cap is
  one cap per member who earned (decided 2026-10-24, MONEY_RULES_DESIGN.md §1).
- **Tips** go to the business. Passing them on to the member is the
  business's duty under US tip law, and the terms must say so.
- **Chargebacks, the guarantee and recoveries** are taken from the
  business's payouts.
- **Background checks** (decided 2026-10-23). Dodorez orders every check, so
  there's one consistent process under the Fair Credit Reporting Act
  (disclosure, signed authorisation, adverse-action notices).
  - The first 3 member checks per business are free.
  - After that, the cost (about $30–$50) is deducted from the business's
    payouts, shown as "Background check — <name> — $35".
  - A failed check isn't charged.
  - A check belongs to the person, so it's reused across hats and businesses
    and nobody pays twice.
  - Individuals later: the same, deducted from their first payout.

## 9. Order

1. **Decisions and the lawyer:** §10, plus the business terms of service and
   the insurance minimums.
2. **Teams, as the first release** (decided 2026-10-23: a one-person company
   is just a team of one; a business without staff adds little over an
   individual). Built in this order:
   1. The business account: company onboarding and Stripe company payouts,
      licences, insurance, badges, and the portal review queue.
   2. Invites and roles, with an ID check and background check per member.
   3. Assigning and reassigning jobs.
   4. The owner's view and live location (§4–5).
   5. Two hats (§7).
3. **Polish:** business analytics, multiple locations, and a vendor for
   Secretary of State checks.

## 10. Architecture

Chosen so the existing system doesn't bend: a provider **is** a person
(`providers.id = users.id`), and every job step, location update, code,
payout, fee, lapse and score is keyed on that person. A business is added
*around* the person, never as a fake person.

| Concern | How |
|---|---|
| The company | New tables: `businesses` (the account, review status, Stripe company account, settings) and `business_documents` (insurance and licences, each reviewed, with expiry) |
| The team | `business_members`: a provider (person) in a business, with a role (owner, dispatcher or worker) and status. One active membership per person |
| Who does the work | Unchanged: `jobs.assigned_provider_id` is the person. Step handlers, tracking, codes and the step audit need no change |
| Business work | `jobs.business_id` (and on the application) marks a job done for a business. Two hats are separated by this column alone: the business sees only rows with its id |
| Money | One routing point, `payee_for(payment)`, which returns the Stripe account and the fee subject (business or person). `PaymentService` asks it instead of reading the provider's account in four places. The fee cap counts per fee subject |
| Stripe onboarding | One `ConnectGateway` for Express accounts, individual or company, used by both `PaymentService` (individuals) and `BusinessPayoutService` |
| Readiness | Pure `app/domain/business.py`: `readiness(business, documents, today)` returns ready or what's missing. Approved, Stripe complete, and in-date $1M general liability for on-site work |
| Review | The portal's Businesses page, behind the existing `providers.verify` permission |
| Switched off | `BUSINESS_ACCOUNTS_ENABLED` (off by default) until the lawyer confirms §11 of LEGAL_QUESTIONS.md |

Each step (§9) ships with its own migration, tests, and app and portal
screens.

## 11. Open decisions

1. ~~Teams first, or single-person businesses first?~~ **Decided: teams.**
2. ~~Insurance minimum.~~ **Decided: $1M general liability** for work at a
   home or on site; not required for online work (§2).
3. ~~Who pays for background checks?~~ **Decided: Dodorez orders, the
   business pays from its earnings** (§8).
4. Whether the owner can read the job chat by default, or only when a
   member asks for help. *Suggested: by default, with the customer told.*
