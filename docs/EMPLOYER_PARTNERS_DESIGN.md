# Employers and work hours: one person, two hats, by the clock

**Status:**
- **Built 2026-11-05** (migration 097) on top of business providers
  ([BUSINESS_PROVIDERS_DESIGN.md](BUSINESS_PROVIDERS_DESIGN.md)).
- Behind the same switch, `BUSINESS_ACCOUNTS_ENABLED` (off by default).
- Legal questions are in [LEGAL_QUESTIONS.md](LEGAL_QUESTIONS.md) §11 and §13.
- **Not run on a server yet:** the AWS account was closed. Before switching
  business accounts on, run the migration and
  `scripts/smoke_work_hours.py` on the new server.

## Why

Dodorez is person to person: customers hire people. Employers (companies,
contractors) can reach and prepare people Dodorez can't:
- tools and equipment;
- steady work;
- training;
- insurance.

The design keeps both:
- **Inside the hours a person agrees with an employer,** they work under the
  employer's name. The employer applies for jobs, assigns them, sets the
  price and is paid.
- **Outside those hours,** they work for themselves. They have their own
  feed, their own prices and their own customers, and the employer doesn't
  see any of it.

There's one person, with two hats, and **the clock picks the hat.** No
switch does.

What the employer and the person agree between themselves stays between
them, outside Dodorez: tools, training, and pay for the employer's jobs.

## 1. How a job flows

There's one algorithm, and customers are always Dodorez's. Employers have no
customers of their own.

1. **A customer posts a job.** Matching reaches people on their own skills,
   rating and distance, as it always has.
2. **The job's work time decides the hat** for each person it reaches. The
   work time is the scheduled start, or for "as soon as possible", when the
   job was posted. Both are fixed, so the label never changes afterwards.
   - **In their agreed hours:**
     - the job goes to their employer's board;
     - the owner and dispatchers are told;
     - the person gets no push and doesn't see it in their own feed.
   - **Outside their hours:** it's in their own feed with a push, as for anyone.
3. **The employer applies once,** naming who'd do it, at its own prices. The
   customer sees one applicant for the employer, beside individuals.
4. **The employer can only assign someone in their hours for that job.**
   That's checked when it applies, at hire, and when it hands the job on.
5. **Payment** goes to the employer's account (business providers §8). How
   it pays its people is between them.

**Direct bookings:**
- Everyone on the customer's shortlist must be outside their work hours at
  the job's time.
- That's checked when booking, with the message worded for the customer.
- Moving down the shortlist skips anyone who can't take it any more (who
  joined a team, or agreed hours that cover it). If nobody's left, the job
  goes out to everyone.

**Repeat bookings:**
- A customer can't set one up with someone whose employer hours cover the
  first visit. Each visit keeps the first one's day and time.
- A provider can't accept one their hours now cover, judged by the first
  visit still ahead.
- If a visit can't be booked anyway, both the customer and the provider are
  told.

**Owner's member picker:** it greys out anyone not in their hours for the
job, with their hours shown.

**New hours never move work already booked.** A job's hat is fixed by its
work time, so hours can't be agreed (on joining, or as a change) while they
would:
- cover the member's own booked work:
  - hired jobs not set off yet, scheduled ahead or as soon as possible (by
    when they were posted, their fixed work time);
  - customers' direct requests still waiting for their answer (they accept
    or decline first, so no customer is left with a request they can't take);
  - jobs they applied for themselves and are waiting to hear about (they
    withdraw first, from the job — hired, it would be refused anyway). A
    business's application naming them isn't theirs to withdraw;
  - the next 6 visits of their regular bookings (on or paused);
- leave out the business's jobs given to them, scheduled or as soon as
  possible.

What each side is told:
- **The member** is told exactly what clashes and what to do: finish it,
  choose hours that leave it out, or ask the business to give its job to
  someone else.
- **The owner** is told about the business's jobs by name, but only that
  the member has work of their own, never what it is.
- **When the owner offers hours,** only the business's jobs are checked.
  The member's own work is checked when they answer.

The code is `clashes` and `clash_message` in `app/domain/work_hours.py`, and
`BusinessTeamService._ensure_no_clash`.

**Applications already sent** are checked again at hire, and the customer is
told to choose another applicant.

The owner always has both hats. A person on no team is always themselves.

## 2. Agreeing the hours

**What hours are:**
- the same hours on each chosen day (Mon–Fri, 09:00–17:00);
- in the person's own time zone, taken from their phone;
- ending after they start on the same day, with no overnight windows.

**The invite:**
1. **The owner invites** someone by phone and **offers hours.**
2. **The person:**
   - **accepts** them and joins;
   - **declines**;
   - or **edits them and sends them back.** They read the location notice
     and authorise the background check at the same time, so the owner's
     acceptance alone lets them in.
3. **The owner then** accepts them (the person joins), offers other hours,
   or withdraws the invite.
4. **Invites don't lapse.**

**Changes later:**
- Either side can offer new hours at any time, and the other accepts or
  declines.
- **The agreed hours stay until both agree.**
- A change left unanswered for `BUSINESS_HOURS_OFFER_DAYS` (7) lapses.
- The side that sent it can withdraw it.

**Leaving:**
- It's on 7 days' notice and happens automatically on the day
  (`finish_due_leaves`). The employer doesn't have to answer.
- The owner can release them sooner, or remove them.

**Members from before work hours:**
- Migration 097 gave them "all the time" hours. Every job comes through the
  business, as it did by default before.
- One case changes: a member the owner had also allowed to work for
  themselves now has one hat at a time. Their own jobs stop until new hours
  are agreed.
- An invite from an app older than work hours offers "all the time".

## 3. What each side sees (provider app)

- **Someone working alone:**
  - the Business screen explains what an employer offers (tools, steady jobs
    in your hours, training and insurance) and what stays theirs (their own
    time, their rating, leaving on 7 days' notice);
  - it says pay and tools are agreed with the employer, not Dodorez;
  - a dismissible line on Home links there (`employer-benefits.tsx`).
- **An invite** shows the hours offered and what they mean. The person can
  accept, suggest other hours, or decline (`business-invites.tsx`).
- **A member:**
  - "Your hours" shows whether they're working for the business or for
    themselves right now, and the agreed hours (`my-hours.tsx`);
  - they can answer the owner's offer, ask for different hours, or withdraw
    a request;
  - the home card and the empty feed say which hat is on.
- **The owner:**
  - the team list shows each member's hours, with badges for hours sent
    back, asked for or offered;
  - from the list they accept, offer other hours, or decline;
  - the invite form has the hours editor (`team-section.tsx`).

## 4. Code

| Concern | Where |
|---|---|
| The rule | `app/domain/work_hours.py`: `in_hours`, `works_for_self(role, hours, at)`, `job_work_time`, `describe`, `hours_view`, `offer_open`. Pure, and tested in `tests/domain/test_work_hours.py` |
| The rule in SQL | `app/models/user.py`: `in_work_hours_sql`, `works_for_self_sql(at=…)`, `JOB_WORK_TIME_SQL`. Kept in step with the Python, and compared on Postgres by `scripts/smoke_work_hours.py` |
| Data | `business_members.work_hours`, plus `proposed_hours`, `proposed_by` (`BUSINESS` or `MEMBER`) and `proposed_at` (migration 097) |
| Reach | Own feed: `get_inbound_for_provider`. Pushes: `working_for_self(ids, at)`. Team board: `open_team_jobs`. Team alert: `businesses_newly_reached(…, at)`. Who sees a team job: `is_distributed_to_business`. Customer map: `get_nearby` (now) |
| Checks | `BusinessHiring.ensure_may_take_for_self(…, at)` when applying, hiring and booking directly; `ensure_worker` for the business's member, by the job's work time |
| Offers | `BusinessTeamService`: `invite(…, hours)`, `respond(…, hours)`, `offer_hours`, `answer_hours`, `offer_my_hours`, `answer_my_hours`, `withdraw_my_hours` |
| API | `POST /businesses/me/team/invites` and `POST /businesses/invites/{id}` take `hours`. `PUT /businesses/me/team/{id}/hours` and `POST …/hours/answer` (owner). `PUT`, `DELETE /businesses/me/hours` and `POST /businesses/me/hours/answer` (member) |
| Removed | The three switches: `businesses.allow_independent_work`, `business_members.own_work_allowed`, `providers.works_independently`, and their endpoints |
| Time zones | `tzdata` is pinned in requirements, since slim images may have no zone database |

## 5. Open

- **The lawyer** (LEGAL_QUESTIONS §13):
  - do agreed working hours make the person the employer's employee?
  - do minimum-wage or overtime rules follow?
- **A maximum** for employer hours (for example 8 a day, 5 days a week). It
  isn't enforced: the hours are whatever both agree.
- **Days off:** a member turning availability off pauses all new jobs. A
  "not today" just for employer hours isn't built.
- **Overnight hours** (22:00–06:00) aren't supported. They would be two
  windows.
- **Customers seeing an employer's name** on an individual's own work:
  decided no. The employer is only shown on its own jobs.
