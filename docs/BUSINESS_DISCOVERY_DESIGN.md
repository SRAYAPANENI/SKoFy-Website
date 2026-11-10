# Finding and requesting an employer: provider-initiated discovery

**Status:** Built 2026-11-10 (migration 105). Layered on
[EMPLOYER_PARTNERS_DESIGN.md](EMPLOYER_PARTNERS_DESIGN.md) and
[BUSINESS_PROVIDERS_DESIGN.md](BUSINESS_PROVIDERS_DESIGN.md), behind the same
switch, `BUSINESS_ACCOUNTS_ENABLED`. Not run on a server yet — same
not-deployed note as EMPLOYER_PARTNERS_DESIGN.md; run the migration before
switching business accounts on.

## Why

Today a provider can only be *found* by a business: an owner invites someone
by phone number (EMPLOYER_PARTNERS_DESIGN.md §2). Someone who wants to work
with an employer has no way to say so — they wait to be noticed. This adds
the other direction: a provider browses businesses near them and asks to
join. The owner still decides, exactly as with any invite today: accept,
decline, or come back with different hours. Nothing about who's in charge
changes.

## 1. Discovery: "Businesses near you"

- Reached from the Home banner (`employer-benefits.tsx`'s
  `EmployerBenefitsLine`, the "Join" pill) and from the Business screen's
  explainer (`EmployerBenefits()`) — both already shown only to someone with
  no business tie, which is exactly the population allowed to ask.
- Lists **every approved business** (`businesses.status = 'APPROVED'`, not
  closed) within the provider's radius — not only ones hiring (decided).
  No new location field: distance is measured from the provider to the
  business owner's own `providers.location`, the point every provider
  already has. `provider_repo.get_nearby`'s `ST_DWithin` query is the
  template.
- Each row carries name, logo, distance and team size — enough to judge at a
  glance, in one query with no per-business lookups. Licensed/insured and
  clear-check-share (BUSINESS_PROVIDERS_DESIGN.md §6) were considered for
  this list too but dropped for v1: each needs its own per-business fetch
  (documents, members), and a list is the wrong place to pay that cost for
  every row. There's no deeper business-profile view to fall back on yet
  either — a provider judges from the list, then asks. Add one if that
  turns out not to be enough.
- **"Open to join requests"** (new, owner-set, off by default): a badge,
  sorts to the top. It never filters the list — every approved business
  stays visible and askable; the flag only signals the owner's actively
  looking right now.
- Already requested, or already invited by them: the row shows that instead
  of a "Request to join" button — no duplicate request (§3 explains why one
  can't happen anyway).

## 2. The ask, answered by what already exists

The turn-up: a provider taps **Request to join**, optionally picks the hours
they'd want (or leaves it open — "whatever you need"), and sends it
(`BusinessTeamService.request`). This writes the same shape of
`business_members` row an owner's invite writes (`status = INVITED`,
`proposed_by = 'MEMBER'`, `proposed_hours` set) — just started from the
other side, tagged `requested_by = 'MEMBER'`. Always as **worker**; a
provider can't request a trusted role, only earn dispatcher afterwards the
way anyone does today (`change_role`).

The payoff of writing it in that exact shape: **no new "decide" or
"withdraw" verb was needed.** EMPLOYER_PARTNERS_DESIGN.md §2 already has the
owner answer hours a member sent while INVITED — `answer_hours(accept=True)`
already joins them (`_join`), `answer_hours(accept=False)` already removes
the row, and `offer_hours` already lets the owner counter with different
hours. Those three, built for "a member sent hours back on an invite",
answer a request unchanged:

- **accept** → `answer_hours(..., True)` joins them, with the hours asked for.
- **decline** → `answer_hours(..., False)` removes the row.
- **counter with different hours** → `offer_hours(...)`.
- **Ignoring it costs nothing** — same as today's owner-sent invites
  ("Invites don't lapse"): nothing times out, there's no badge for leaving
  it unanswered. That's the "ignore" the idea asked for.

The owner sees these in the same **Team** list as everyone else
(`team-section.tsx`) — not a separate "Requests to join" list as first
sketched here. The badge reads "Wants to join" instead of "Invited", and the
decline/withdraw button reads "Decline" instead of "Withdraw invite", by
checking `requested_by`; everything else (the hours card, Accept/Offer
other hours/Decline buttons) is the exact same UI an owner-sent invite with
a countered offer already uses. One component, both directions — simpler
and guaranteed visually consistent, rather than two.

On the provider's own side: `my_invites()` (what feeds the Business screen's
invite card) already includes this row — it was only ever filtered by
`provider_id`, not by direction — and already computes `waiting_on =
'BUSINESS'` correctly from `proposed_by = 'MEMBER'`. That state already
renders as "Sent to {business} — you'll join once they accept your hours"
with a single **Withdraw** button (`respond(accept=False)` removes the
row) — exactly the UX a pending request needs, with zero new code. The only
change made there was a cosmetic one: the card's title reads "Your request
to join" instead of "Team invite" when `requested_by === 'MEMBER'`.

Someone already on a team, or who owns their own business, never gets this
far — `_ensure_may_join` already blocks it and needed no change.

**Asking again.** A business the provider already has a row with (an old
invite, a past request, or past membership) can't take a second row — the
same `(business_id, provider_id)` uniqueness `invite()`/`reinvite()` already
work around. `request()` mirrors that: a REMOVED row is reopened in place
(`re_request`, the mirror of `reinvite`) rather than inserted fresh; any
other status is rejected with "You already have an invite or a pending
request with them."

## 3. Data

No new table.

`business_members` gets one column:

| Column | What |
|---|---|
| `requested_by` | `BUSINESS` \| `MEMBER` — who started this row. Existing rows backfill to `BUSINESS` (today, only an owner ever creates one). Mirrors `proposed_by`'s existing BUSINESS/MEMBER vocabulary for hours offers. |

`businesses` gets one column:

| Column | What |
|---|---|
| `open_to_join_requests` | boolean, owner-set in Business settings, default **false**. Cosmetic only (§1) — never gates who can be asked. |

The existing unique constraint (`business_id`, `provider_id`) already stops
a second request to a business they're already pending with or active on —
the same `IntegrityError` → `ConflictError` path `invite()` relies on today.
A new cap, `MAX_OPEN_REQUESTS_PER_PROVIDER` (suggest 10), stops one provider
papering every nearby business with requests at once — the mirror of
`MAX_OPEN_INVITES` on the business's side.

## 4. Code (where this plugs in)

| Concern | Where |
|---|---|
| Data | `business_members.requested_by` (`BUSINESS` \| `MEMBER`, mirrors `proposed_by`); `businesses.open_to_join_requests` (migration 105) |
| Discovery query | `BusinessRepository.nearby(provider_id, radius_km)` — the same `ST_DWithin` shape as `provider_repo.get_nearby`, joined to the owner's `providers.location`, filtered to `status = 'APPROVED'` |
| Discovery service | `BusinessTeamService.nearby(provider_id, radius_km=None)` — defaults to the provider's own `preferred_radius_km` |
| Sending a request | `BusinessTeamService.request(provider_id, business_id, hours=None)` — `_ensure_may_join`, then `BusinessRepository.request` (fresh) or `.re_request` (a REMOVED row, reopened) |
| Owner answering | Nothing new — the existing `offer_hours` / `answer_hours` (EMPLOYER_PARTNERS_DESIGN.md §2) already handle a MEMBER-sourced INVITED row; see §2 above |
| The "open to join requests" toggle | `BusinessService.set_discoverable(provider_id, open)`; `PUT /businesses/me/discoverable` |
| API | `GET /businesses/nearby`, `POST /businesses/{business_id}/requests` |
| App | New screen `business-nearby.tsx` (list, off the Home banner and `EmployerBenefits()`); `team-section.tsx` and `business-invites.tsx` get `requested_by`-aware copy, not a new list; a toggle row on `business.tsx` for the owner |
| Switch | `BUSINESS_ACCOUNTS_ENABLED`, same as everything else here |

## Open

- **"Open to join requests" default.** Built **off** — an owner opts in
  deliberately rather than every new business starting out lit up as
  "Hiring" without having said so. Flip it if every approved business
  should default to showing it instead.
- **One anchor point for a multi-person business.** The owner's own
  location stands in for the whole business. A business whose owner sits
  in one spot while the team covers a wide area could read as farther (or
  closer) than it really is to a provider nearby. Fine for a first
  version; an explicit service-area field is the fallback if that's
  misleading in practice.
- **The cap on open requests per provider** — built as 10
  (`MAX_OPEN_REQUESTS_PER_PROVIDER`). Tune or drop it.
- **Legal.** This doesn't change the classification question already open
  in LEGAL_QUESTIONS.md §13 — who starts the contact doesn't change what
  agreed hours mean — but it does mean more people reach that question
  sooner, since asking is now easy. Worth a line to the lawyer alongside
  §13, not a new one.
