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
- **Privacy policy wording** for all of the above.

