# Questions for the lawyer

One list of what needs legal advice before launch (USA). Each item says what's
built today, so the answer can be turned into a change. Nothing here is legal
advice. Add new questions here rather than leaving them inside feature docs.

Last updated: 2026-10-21.

## 1. Provider classification: contractors and businesses

**Today:** every provider is treated as an independent contractor. They choose
which jobs to take, set their own quote, and use their own tools. Dodorez takes
a platform fee.

**Under discussion (not built):** two provider types. *Individuals* would be
gig workers / independent contractors. *Businesses* (licensed trades with their
own staff and insurance) would be B2B suppliers, not gig workers.

- Under which state tests (ABC test in CA/MA/NJ, common-law elsewhere) are
  individual providers safely contractors? What must the platform avoid doing
  (setting prices, ranking by acceptance rate, required uniforms, …)?
- Do the HCI score and its effect on job distribution count as "control"?
- For businesses: what do we need to collect (licence, insurance, EIN), and do
  we owe anything to their staff?
- 1099-K / 1099-NEC reporting through Stripe Connect: anything we must do
  ourselves?

## 2. Pickup & Drop from a person

**Today:** a customer can have an item collected from any address (a friend's
house) and brought to them, not only from a store. The safeguards:

- The customer must have an ID number and photo on their profile. Nobody
  reviews it yet; a KYC vendor will.
- They say what the item is and what it's worth, up to $500.
- They tick a box saying it isn't alcohol, tobacco or vapes, cannabis,
  medication, weapons, cash, or anything illegal.
- The provider must see the item and tick three checks before taking it, or
  refuse it. A refusal cancels the job, and the provider keeps the trip fee.
  "Nobody there" needs a photo of the door.

**Store pickups too (since 2026-11-15):** the customer ticks a box saying
nothing in the order is alcohol, tobacco or vapes, cannabis, controlled
medication, weapons, or anything illegal. Before that, store orders only
showed a note and nothing was recorded.

**What's kept with every pickup job** (and shown in the admin portal's job
view; kept after the account is deleted, without the person's name):

- The exact sentence the customer ticked, when, and which version of the
  wording it was (`app/domain/pickup.py` `DECLARATION_VERSION`). If the
  wording changes, a customer on the old one is asked to tick it again.
- From a person: each of the provider's three checks, word for word with
  its version, and the photo, GPS position and time at the pickup.
- Every pickup: the provider's photo at the pickup, taken at the pickup's
  location (the receipt too, for buy-for-me).
- Every delivery: a photo of the order at the drop-off, taken there before
  the customer's code is accepted, plus the code's GPS position and time.

**Questions:**

- Who is liable if a banned or illegal item is carried anyway: the customer
  (who declared it), the provider (who checked it), or Dodorez? Is the
  declaration plus the provider check a reasonable-care defence?
- What if the item is lost, damaged or stolen in transit? Do we need goods-in-
  transit cover, a value cap in the terms (we have $500), or both?
- Is an unreviewed ID on file enough, or must it be verified before a pickup
  from a person?
- Is keeping the trip fee after a refusal fair under consumer-protection law,
  including when the refusal was "nobody there"?
- Does a provider opening or inspecting a closed package raise privacy issues?
  Today they may refuse a sealed box they can't check, but aren't told to open
  it.
- Courier or delivery licensing in any state for carrying goods between
  private people?

## 3. Call recording

Full draft and questions: [CALL_RECORDING_CONSENT_DRAFT.md](CALL_RECORDING_CONSENT_DRAFT.md).
Recording stays off until it's answered.

- Is an on-screen notice enough in all-party-consent states, or is a spoken
  notice needed too?
- Consent once at sign-up, on every call, or both?
- Is 90-day retention (longer for open disputes) appropriate?
- CCPA/CPRA access and deletion requests for recordings.
- Calls where one person is outside the US.

## 4. Licensed work in the catalogue

**Today:** the catalogue includes Caretaker (elder and child care,
post-hospital care) and Physiotherapist. See
[JOB_SHAPE_DESIGN.md](JOB_SHAPE_DESIGN.md), open question 1.

- Childcare, elder care and physiotherapy are licensed in many states. Should
  these skills be off at launch, or allowed only for verified licence holders?
- Any other skills in the catalogue that need a licence check before a provider
  can take the work?

## 5. Card fee passed to the customer

**Today:** the customer pays the job amount plus Stripe's card fee (2.9% + 30¢)
on every card payment (`app/services/offer_pricing.py`, `processing_fee`).

- Is this a card *surcharge* under state law and card-network rules? Some
  states restrict or ban surcharging, networks cap it, and debit cards can't
  be surcharged at all.
- Would a fee charged the same way whatever the payment method (card, debit,
  bank) avoid those rules? What must the checkout show for it to be compliant
  with all-in pricing laws such as California SB 478?

## 6. Fees, cancellations and penalties in the terms

**Designed, not built:** [MONEY_RULES_DESIGN.md](MONEY_RULES_DESIGN.md).
Providers pay a banded fee with per-job and monthly caps. A customer who
cancels after the provider set off or arrived pays part or all of the visiting
fee. A provider who backs out or is late is paused after repeats (no fines).

- What must the terms and the checkout say for cancellation fees to be
  enforceable and fair under state consumer law?
- Are reliability pauses for providers consistent with treating individuals
  as independent contractors (§1)?

## 7. Customer strikes

**Designed, not built:** a customer whose dispute is found "contradicted by
evidence" gets a strike that lasts 12 months. Two strikes end their 24-hour
hold on visiting fees; three lead to an account review and possible suspension.
Every strike can be appealed to a different, more senior agent.

- What must the terms say before strikes are enforced?
- Is it fair and lawful to restrict an account on a support agent's finding,
  with this appeal process?
- How long may we keep the strike record and the evidence?
- **Added 2026-10-22, paying outside the app** ([MONEY_RULES_DESIGN.md](MONEY_RULES_DESIGN.md) §10):
  chat messages are scanned, and phone numbers and emails are hidden. Flagged
  messages are kept, original text included, for support to review. Asking
  to be paid outside the app is a strike for either side, and a provider's
  second strike pauses them from new jobs for 30 days. What must the terms
  and the privacy notice say about scanning and keeping chat messages? Is
  pausing an independent contractor for this consistent with their
  classification?

## 8. ID documents and personal data

**Today:** customers and providers upload ID photos, which are stored on our
server. Phone numbers are shared between a customer and their hired provider.

- What may we keep, for how long, and how must it be protected (state ID-data
  and biometric laws, CCPA/CPRA)?
- What do the privacy policy and terms need to say about IDs, location
  tracking during a job, and the chat?


## 9. Taxes, holding customers' money, the guarantee (added 2026-10-22)

See [MONEY_RULES_DESIGN.md](MONEY_RULES_DESIGN.md) §11–13.

- **Sales tax on services.** Some states tax certain services, such as
  cleaning, repairs and installations in some places. Must Dodorez collect
  sales tax as a marketplace facilitator on jobs, on its customer fee, or
  both? On which categories and in which states?
- **Tax forms for providers.** Is Stripe Connect's 1099-K enough, or does
  Dodorez file anything itself?
- **Holding customers' money.** Payments are held in Dodorez's Stripe
  balance until the work is done, and paid out by transfer (separate
  charges and transfers). Does that need a money-transmitter licence, or
  does Stripe's role cover it?
- **The guarantee.** "Covered for 30 days, refunds up to $1,000, paid by
  Dodorez" — what must the terms say? Does it count as insurance or a
  warranty in any state?
- **Charging a saved card for repeat bookings and automatic quote
  acceptance.** What consent and notice do the terms need?
- **Recovering a chargeback from a provider's payouts.** Is it fair and
  lawful under the provider agreement, with the notice and appeal described?

## 10. Pets and homes in a provider's care (added 2026-10-22)

See [MONEY_RULES_DESIGN.md](MONEY_RULES_DESIGN.md) §14 and
[JOB_SHAPE_DESIGN.md](JOB_SHAPE_DESIGN.md) phase 4.

- **Boarding at a provider's home.** Do any states or cities need a kennel
  or boarding licence for someone who looks after other people's pets at
  home? Must Dodorez check for one, or limit how many a provider hosts?
- **A pet injured, lost or dies in care.** Who is liable: the provider,
  Dodorez, or neither? What must the terms say? Does the guarantee (§12)
  cover it, or should it be excluded? Is insurance needed?
- **House-sitting.** The provider has access to the customer's home. Are
  background checks required (we require them in-app, but is that enough)?
  Who is liable for theft or damage?
- **Holding the full price at booking.** Is the full price held until the
  job is done, then kept in part on a late cancel, fair and lawful? What
  must the terms and the booking screen say?
- **Photo updates.** Photos of the customer's pet or home are stored and
  shown to the customer. How long may we keep them, and what consent is
  needed?

## 11. Business providers and their staff (added 2026-10-23)

> **Status: pending — to be taken to the lawyer later.** The owner decided on
> 2026-10-23 to switch business providers on before this review. Anything
> the lawyer changes is applied afterwards: the notice and disclosure
> wording, the insurance minimum, the no-poaching rule, the business terms.
> Questions §1 (classification) are also still open. Switching off again is
> one setting: `BUSINESS_ACCOUNTS_ENABLED=false`, then restart.

See [BUSINESS_PROVIDERS_DESIGN.md](BUSINESS_PROVIDERS_DESIGN.md).

- **Location tracking of a business's employees.** The owner sees a member's
  live location while they're on a job, never off duty. What notice and
  consent must the business give, and Dodorez show? New York, Connecticut
  and Delaware have monitoring-notice laws; California employee data falls
  under CPRA. Is Dodorez the business's service provider for this data?
- **The owner reading job chats and calls.** The customer is told. Is that
  enough, and what about recorded calls?
- **Insurance:** $1M general liability is required for on-site work. Is
  that enough, and should Dodorez require workers' comp proof?
- **Background checks ordered by Dodorez for a business's staff,** paid from
  the business's payouts. The disclosure and authorisation wording shown on
  joining is `BACKGROUND_CHECK_DISCLOSURE` in `app/domain/business.py`. Is it
  enough as a standalone FCRA disclosure? What Fair Credit Reporting Act disclosure,
  authorisation and adverse-action process do we need, and can a check be
  reused for the same person across businesses?
- **The business terms of service:** indemnity, insurance, the business's
  duties to its staff (tips passed on, wage law), and recovering chargebacks
  from its payouts.
- **One person, two hats.** A member also works for themselves, with a
  12-month rule against taking the business's customers. Is that
  enforceable, and fair to both sides? Since 2026-10-24 it follows the
  person whoever they work for, including a business they start after
  leaving. Several states limit non-solicitation of former employers'
  customers (California especially). Is a platform rule enforceable there?
- **Leaving a team on notice (2026-10-24).** A member who asks to leave goes
  after 7 days, sooner if the owner releases them. The owner can't refuse,
  and unfinished team jobs must be finished or handed on first. Is a notice
  period like this fine for at-will employees and for contractors? Does
  holding someone until their last job is done ever cross a line?

## 12. Deleting accounts and what's kept (added 2026-10-24)

See [ACCOUNT_DELETION_DESIGN.md](ACCOUNT_DELETION_DESIGN.md). Built: the account
closes at once and identity is removed after 30 days. Jobs, payments, reviews,
disputes and claims are kept under "Former customer" / "Former provider".

- **Is that de-identified enough** under CCPA/CPRA and other states' laws? It
  keeps the same internal ID, the city, and job locations rounded to about 1 km.
- **Retention periods:** payments and tax (about 7 years?), disputes and claims,
  strikes, background-check order records (FCRA), insurance certificates of
  closed businesses (liability claims can come years later).
- **Chat and support messages** are blanked at 30 days. Should some be kept
  longer for disputes or safety?
- **Banned numbers** are kept as a keyed fingerprint to stop re-registration.
  Is that a lawful fraud-prevention purpose?
- **Backups:** how long, and what the privacy policy must say about them.
- **Legal hold:** who may set it, and is a written reason needed?
- **Removal delayed past the 30 days** (built 2026-10-27): a dispute, guarantee
  claim or bank chargeback from a finished job holds removal back until it's
  settled, however long. Money owed to Dodorez holds it back **90 days at
  most** (`ACCOUNT_DELETION_OWED_WAIT_DAYS`); then the identity goes and the
  debt record stays without a name, so it can no longer be collected. Is 90
  days reasonable, and may the identity be kept longer to collect a debt? The
  person is texted once that removal is delayed and why.
- **Privacy policy wording** for all of the above.
- **The privacy policy draft** at `/v1/legal/privacy`
  (`skofy-backend/app/legal/privacy_policy.html`, 2026-10-25) needs review
  before launch. It was written from what the apps do. Points to check:
  - **Storage in India:** our servers run on AWS in India. Is that fine for
    US users, and is the wording right?
  - **Age 18+.**
  - **AI processing by Groq.**
  - **The list of state rights.**
  - **"Backups are replaced":** a backup retention period is still to set.
  - **Contact email:** a placeholder.

## 13. Employers and work hours (added 2026-11-05)

> **Status: built, behind `BUSINESS_ACCOUNTS_ENABLED` (off).** See
> [EMPLOYER_PARTNERS_DESIGN.md](EMPLOYER_PARTNERS_DESIGN.md). A team member
> agrees days and hours with a business. In those hours, jobs reach them
> through the business, which applies, sets the price, assigns them and is
> paid. Outside them, they work for themselves. Pay, tools and training are
> agreed between business and member, outside Dodorez.

- **Classification.** The business assigns its people in agreed hours and
  is paid for the work. Is the member then its employee rather than a
  contractor? Is that the business's responsibility alone, or Dodorez's
  too? Does it change the answer to §1?
- **Wage and hour rules.** Do agreed hours bring in minimum wage, overtime
  or predictive-scheduling laws (for example in Oregon, Seattle, New York
  City or San Francisco)? Must the terms make the business responsible for
  them?
- **The terms.** What must say that Dodorez isn't a party to the business's
  arrangement with its people (pay, tools, training), and that the business
  answers for its people in its hours: pay, classification, insurance?
- **A maximum.** Should Dodorez cap employer hours (for example 8 a day,
  5 days a week) to keep members independent outside them, or leave it to
  what both agree?
- **Leaving.** Is 7 days' notice, ending automatically, fair and enforceable
  in every state?
- **Provider-initiated requests** (built —
  [BUSINESS_DISCOVERY_DESIGN.md](BUSINESS_DISCOVERY_DESIGN.md)): a provider
  can now ask a business to join, not only the other way round. Doesn't
  change the classification question above — who starts the contact isn't
  the issue — but it does mean more people reach that question sooner,
  since asking is easy. Worth a line to the lawyer alongside the rest of
  this section, not a new one.

## 14. Providers paying for customers' shopping (added 2026-11-15)

- Today, on "Buy it for me", the provider pays the store with their own money
  and is paid back from the customer's held card when the job is done. It's
  opt-in, up to an amount the provider chooses (max $200). Does a provider
  fronting a customer's money, and being reimbursed through us, raise any
  money-transmission, lending or consumer-credit question?
- Planned instead: a Dodorez virtual card per order (Stripe Issuing), loaded
  from the customer's hold, that the provider pays with. Are we then the
  card programme's business user, and does that change how providers are
  classified (we'd be equipping them)? Any state rules on issuing cards to
  independent contractors?
- Tax forms: a shopping payout is the receipt paid back (the provider's own
  money returned) plus the shopping fee (income). We record the two apart
  (`payments.reimbursed_amount`) and count only the fee as earnings. When
  Stripe Connect's 1099 reporting is turned on it totals transfers by
  default — confirm we should report only the fee (adjusting Stripe's figures
  before filing, or filing ourselves), and whether a fee paid to a team
  member's own account on a business's job is reported to them, not the
  business.

## 15. Providers' vehicles and insurance (added 2026-11-16)

- Providers carry customers' items in their own vehicles. Car-trunk and
  van-size trips now need a vehicle Dodorez has checked: photos, plate and
  registration (docs/VEHICLES_DESIGN.md). We also collect proof of insurance,
  but don't require it yet (`VEHICLE_INSURANCE_REQUIRED=false`).
- Most personal auto policies exclude paid delivery ("business use"). Should
  we require delivery or commercial cover, or a rideshare/delivery
  endorsement, before these trips reach a provider? Does it differ by state?
- If an item is damaged or someone is hurt during a delivery and the
  provider's policy refuses, what's our exposure? Should Dodorez carry a
  contingent or hired-and-non-owned auto policy, as delivery platforms do?
- Does checking vehicles and their papers make providers look more like
  employees (we "equip" or control them)? Is it any different from checking
  ID or licences?
- We keep plate numbers, vehicle photos and registration cards. We show the
  plate only to the customer of a booked job, and remove it when an account
  is deleted. Does any state's privacy law (DPPA, CCPA) need more, e.g.
  a notice when the photo is taken?

## 16. Watching providers take the skill test (added 2026-11-21)

- Skill tests are now timed by our server. Leaving the app or taking a
  screenshot is recorded, and a pass that looks wrong is held for a person to
  review (docs/SKILL_VERIFICATION_DESIGN.md). Nothing is filmed yet. Is
  keeping those signs and each answer's timing fine under CCPA and similar
  laws, with the notice on the test's rules screen?
- Built 2026-11-21: for skills an admin picks (electrical, gas, HVAC), the
  front camera is on during the test: a face photo at the start, photos
  every 15 seconds, and two spoken answers that are recorded, transcribed and
  scored by AI. The provider ticks a consent screen first. A **person**
  compares the photos with the ID by eye. No software measures the face, so
  we believe no "biometric identifier" (face geometry) is created under
  Illinois BIPA, Texas CUBI or Washington's law. Is that right? Is our consent
  wording enough, and is the retention (30 days after the decision, 180 if
  turned down, at once on account deletion) fine?
- If we later add automatic face matching (e.g. AWS Rekognition), what
  written consent, retention schedule and published policy would we need?
- Is a voice recording of a spoken answer biometric (a voiceprint) if we
  only transcribe and score it, never identify the speaker?
- Turning down a test for suspected cheating takes a provider's badge away
  and so costs them work. Do we owe them the evidence, an appeal, or a
  person's decision (never the AI's alone)? We already require a person.
