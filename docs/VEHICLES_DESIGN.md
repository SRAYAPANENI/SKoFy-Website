# Providers' vehicles — design

> Built 2026-11-16 (migration 113). On for everyone; insurance is collected but
> not required until the lawyer answers (`VEHICLE_INSURANCE_REQUIRED`,
> LEGAL_QUESTIONS.md §15).

## Why

A Pickup & Delivery trip can need room: an item that fills a car trunk
(`item_size = LARGE`) or needs a van or pickup truck (`BULKY`). Before this, a
provider said they could carry it just by setting a size price (even $0) on
their rate card. Nothing checked it. Someone in a small car could take a sofa
job, turn up, fail the job and cost the customer their slot.

Now those trips reach only a provider with a vehicle Dodorez has checked.
Bags and boxes (`SMALL`, `MEDIUM`) need no vehicle, because they're done on
foot, by bike or by bus.

## The rules (`app/domain/vehicles.py`, pure)

| Vehicle type | Carries a car-trunk load | Carries a van load |
|---|---|---|
| Car, SUV, minivan | yes | no |
| Pickup truck, cargo van, box truck | yes | yes |

- **A vehicle counts** when it's `APPROVED` and has a `REGISTRATION` document
  that's approved and runs past today. Once `VEHICLE_INSURANCE_REQUIRED` is on,
  it also needs an approved, in-date `INSURANCE` document. This is worked out
  every time (`readiness`) and never stored, so a registration that runs out
  stops new jobs of that size on its own.
- **A trip needing room reaches a provider** only if they have:
  1. the size price on the job's skill (unchanged — it's their price); and
  2. a vehicle that counts and carries that size.
- **The smallest fitting vehicle is used** (`smallest_fitting`). A provider
  with a car and a van comes in the car for a car-trunk load.
- **The vehicle belongs to the person who does the job**, not the rate card.
  A business member applying under the owner's prices needs their own
  vehicle. Vehicles owned by a business are a later step.

## Where it's enforced

The SQL and the Python both come from the same tables (`CARRIERS`,
`TYPE_ORDER`), so they can't drift apart.

| Where | How |
|---|---|
| Job feed and distribution | `can_carry_sql` (`app/models/user.py`) includes `carries_sql` |
| Customer's trip estimate ("most nearby charge…", "no one nearby carries…") | `extra_counts_sql`: a size price counts only with a vehicle behind it |
| Applying (yourself, or a business naming its member) | `VehicleService.ensure_carries`, called first in `JobService._apply` |
| Booking a provider directly | `ensure_carries_size(..., asked_by="customer")` in `create_job` |
| Hire | checked again in `_finalize_hire`, then fixed on the job as `jobs.vehicle_id` |
| A business giving the job to another member | `reassign_for_business` checks the new member and moves `vehicle_id`; hand-back to the owner clears it |
| The hired provider released | `vehicle_id` is in `_VISIT_PROGRESS_CLEARED` |

Each error is worded for whoever is asking (`asked_by`): the provider (what
to add, or "still being checked"), their team, or the customer ("choose
someone else").

## What people see

- **Provider app, Profile → My vehicles** (`app/vehicles.tsx`):
  - per size: "Reaches you", "Being checked" or "Needs a vehicle";
  - each vehicle with what's missing, its papers and their end dates;
  - adding one: type, make, model, year, colour, plate and state, a photo of
    the front with the plate readable, a photo of the cargo space, and the
    registration with its end date;
  - renewing a registration or adding insurance; removing a vehicle.
- **Rates & hosting**: under the car-trunk and van prices, whether a checked
  vehicle backs them, with a link to My vehicles. The price can be set either
  way.
- **Provider's job sheet, before pickup**: "Come in your White Ford Transit
  (7ABC123) — the customer was told it's the one coming."
- **Customer, applicant card**: "White Ford Transit · cargo van · checked".
  No plate before hire. A waiting applicant with no checked vehicle for the
  load (applied before checks began, or their registration ran out since) is
  marked "can't be hired until there is" (`vehicle_missing`), rather than the
  customer finding out from a failed hire.
- **Customer, tracking screen after hire**: the vehicle, its plate and a
  photo of its front, so they can recognise it. A different vehicle turning
  up is something to report.
- **Portal → Vehicles** (`providers.verify`, with a sidebar count):
  - the queue: new vehicles, and renewals or insurance on approved ones;
  - the detail: both photos, the plate, a warning when another account has
    the same plate, and each document.
  - Approving a vehicle also approves the registration it came with, because
    that's what shows the plate is theirs. Turning a vehicle down also turns
    down anything still waiting on it (insurance added meanwhile); otherwise it
    would leave the queue undecided. Turning anything down needs a note.
- **Photos are images only** (`upload_photo`, `IMAGE_MIME`): JPEG, PNG or WEBP.
- **Support** sees the vehicle fixed on a job, with plate and photos, on the
  portal's job page and dispute page (`AdminJobService.get_full` → `vehicle`).
  That's what a "a different vehicle turned up" report is checked against. A
  provider's vehicles, removed ones too, show on their provider panel.

## Lifecycle

- **Vehicle:** `PENDING_REVIEW` → `APPROVED` or `REJECTED`. A rejected one is
  removed and added again, fixed.
- **Removing a vehicle:**
  - never approved → deleted, with its files;
  - approved → `RETIRED` and kept, because jobs point at it;
  - jobs already booked with it keep it.
- **Documents:** `PENDING_REVIEW` → `APPROVED` or `REJECTED`, then `EXPIRED`.
  - A renewal is a new row. The approved one keeps counting until it ends.
  - A newer upload of the same kind that's waiting or turned down replaces
    the old one.
- **Daily** (`check_vehicle_document_expiries`, 04:25 UTC):
  - a reminder `VEHICLE_DOC_REMIND_DAYS` (14) before a document ends;
  - marked expired after, and the provider is told;
  - nothing is sent if a renewal is already in.
- **Limits:** 3 listed vehicles per provider; the same plate can't be added
  twice by one provider.

## Privacy and deletion

- **History points at vehicles:** `jobs.vehicle_id` is `RESTRICT`, and so is
  `provider_vehicles.provider_id`.
- **When an account is anonymised** (`AccountRepository.anonymise`):
  - its vehicle documents are deleted;
  - vehicles no job used are deleted;
  - the rest keep only what they were ("a white Ford Transit"), with no
    plate, photos or note;
  - all files are removed from storage (`files_of`).
- **The plate is shown only** to the provider, the portal, and the customer
  of a job it's booked on, after hire.

## Rolling out

1. Deploy migration 113. From then on, car-trunk and van trips need a checked
   vehicle. Providers who'd only set a price stop getting those trips.
2. Run the one-off notice telling those providers why:
   `app.tasks.vehicle_tasks.tell_priced_without_vehicle`.
3. Check new vehicles in Portal → Vehicles. Approval is aimed at within a day.

## Later

- Vehicles owned by a business and assigned to its members.
- Insurance required, once the lawyer answers (`VEHICLE_INSURANCE_REQUIRED=true`).
- Reading the plate and registration automatically (OCR) to speed up review.
